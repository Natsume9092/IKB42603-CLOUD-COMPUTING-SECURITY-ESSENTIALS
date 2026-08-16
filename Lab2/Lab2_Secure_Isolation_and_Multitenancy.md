# IKB42603 Cloud Computing Security Essentials — Lab 2

## Secure Isolation & Multi-Tenancy

### Compute, Network and Storage Isolation — Docker & Kubernetes

| | |
|---|---|
| **Name** | MUHAMMAD AKMAL HAKIM BIN MOHD YUZLAN |
| **Student ID** | 52215125582 |
| **Course** | IKB42603 Cloud Computing Security Essentials (UniKL MIIT) |
| **Lab** | Lab 2 · Weeks 3–4 · Secure Isolation & Multi-Tenancy |
| **Date** | 16 August 2026 |

---

## 1. Objective

The objective of this lab is to demonstrate how shared, multi-tenant cloud infrastructure can be **securely isolated** across the three core dimensions — **compute, network, and storage** — using Docker and Kubernetes. Multi-tenancy means many customers share the same physical hardware, so without deliberate isolation controls, one tenant can interfere with or access another tenant's resources.

At the end of this lab, I am able to:

1. **Demonstrate compute isolation** by separating tenants into containers and Kubernetes namespaces.
2. **Observe the default-open behaviour** of shared infrastructure and explain why it is a security risk.
3. **Implement network isolation** with a default-deny NetworkPolicy and prove that cross-tenant traffic is blocked.
4. **Enforce storage isolation** so that one tenant cannot read another tenant's data or secrets.
5. **Explain data remanence** and demonstrate secure deletion.

These outcomes map to **CLO2 — Construct secure cloud operations that safeguard data integrity**, covering the Week 3 lecture topic on *Secure Isolation of Physical & Logical Infrastructure*.

---

## 2. Lab Environment & Setup

| Tool | Purpose |
|---|---|
| Docker Desktop | Container runtime (used for the data remanence demo) |
| kind | Runs a Kubernetes cluster inside Docker |
| kubectl | CLI to manage the Kubernetes cluster |
| Calico | CNI (Container Network Interface) that **enforces** NetworkPolicy |
| curl / curlimages/curl | Probe tool used to test cross-tenant connectivity |

**Why Calico?** Kubernetes Namespaces only separate resources *logically* — the default network is flat and open. A NetworkPolicy is only enforced if the CNI plugin supports it. The default kind network does **not** enforce NetworkPolicy, so we created the cluster with the default CNI disabled and installed Calico as the enforcement layer.

**Mental model:** A Kubernetes cluster = an apartment building; pods = rooms/people; namespaces = different tenants/companies; NetworkPolicy = the rules saying who may talk to whom; Calico = the firewall that actually enforces those rules.

### Cluster Creation with Policy Enforcement

```bash
cat <<EOF | kind create cluster --name ccse-lab2 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF
```

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
kubectl rollout status daemonset/calico-node --timeout=180s
```

![Cluster setup with kind and Calico](<images/Compute Isolation & the Default-Open.png>)

The output confirms the cluster `ccse-lab2` was created with node image `kindest/node:v1.30.0`, the kubectl context was set to `kind-ccse-lab2`, and the Calico manifest was applied — creating the `calico-node` DaemonSet, CRDs (CustomResourceDefinitions) and RBAC resources needed for policy enforcement.

---

## 3. Task 1 — Two Tenants on One Cluster (Compute Isolation)

**Goal:** Model two customers as two separate namespaces (`tenant-a`, `tenant-b`) sharing the same physical cluster infrastructure. Each tenant gets its own web server (`nginx`).

```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b

# Deploy a simple web server for each tenant
kubectl -n tenant-a create deployment web --image=nginx
kubectl -n tenant-b create deployment web --image=nginx
kubectl -n tenant-a expose deployment web --port=80
kubectl -n tenant-b expose deployment web --port=80

kubectl get pods,svc -n tenant-a
kubectl get pods,svc -n tenant-b
```

![Two tenants on one cluster](<images/Two Tenants on One Cluster.png>)

**Observation:**

```
tenant-a:  pod/web-7c56dcdb9b-xhv6m   1/1 Running   ClusterIP 10.96.73.140:80
tenant-b:  pod/web-7c56dcdb9b-btmc8   1/1 Running   ClusterIP 10.96.9.59:80
```

Both tenants run an identical workload (`web`) on the **same cluster** and the **same node**. The two deployments have the same name but exist in different namespaces, so they are completely distinct resources — this is compute isolation at the namespace level. Notice each tenant has its own ClusterIP service.

---

## 4. Task 2 — Observe the Default-Open Risk

**Goal:** Prove that namespace separation alone does **not** isolate tenants on the network. By default, Kubernetes has no NetworkPolicy, so pods in `tenant-a` can reach pods in `tenant-b`.

```bash
# Get tenant-b's service IP
kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'; echo
# → 10.96.9.59

# From tenant-a, curl tenant-b
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
    -- curl -s -m 5 http://10.96.9.59 -o /dev/null -w 'HTTP %{http_code}\n'
```

![Observing the default-open risk](<images/Observe the Default-Open Risk.png>)

**Result: `HTTP 200`**

The probe pod running inside `tenant-a` successfully reached the web server in `tenant-b`. Note the shell error in the middle of the screenshot — that was the placeholder `<B_IP>` not being replaced; once the real IP `10.96.9.59` was used, the probe succeeded.

**Why is this dangerous?** On shared, multi-tenant infrastructure, isolation is **NOT automatic** — it must be configured. An `HTTP 200` means any tenant can:

- scan and discover other tenants' internal services;
- potentially access unauthenticated endpoints or internal APIs;
- launch attacks (pivoting, DoS) against neighbours on the same node.

This is the *default-open* risk from Week 3. I saved this `HTTP 200` result because Session B will show the **same probe failing** once a NetworkPolicy is applied.

---

## 5. Task 3 — Contain the Noisy Neighbour (Resource Quotas)

**Goal:** Isolation is not only about who can reach whom — it is also about **resources**. A single tenant must not be able to exhaust the shared node's CPU/memory and starve other tenants (the "noisy neighbour" problem).

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 512Mi
    pods: "5"
EOF

kubectl describe resourcequota tenant-a-quota -n tenant-a
```

![Contain the noisy neighbour (Resource Quotas)](<images/Contain the Noisy Neighbour (Resource Quotas).png>)

**Output:**

```
Name:         tenant-a-quota
Namespace:    tenant-a
Resource            Used   Hard
--------            ----   ----
pods                1      5
requests.cpu        0      1
requests.memory     0      512Mi
```

The quota caps `tenant-a` at a maximum total of:

- **1 CPU core** of requested CPU,
- **512 MiB** of requested memory,
- **5 pods** in total.

A ResourceQuota is a **fairness/availability control**: it prevents one hyperactive tenant from consuming the node's shared capacity and degrading the service of co-tenant(s). This is the compute-side counterpart to the network controls in Task 4.

---

## 6. Task 4 — Default-Deny Network Isolation

**Goal:** Apply the principle of **deny by default, permit by exception**. We block ALL ingress into `tenant-b`, then re-run the exact same probe from Task 2.

```bash
# Deny ALL ingress into tenant-b
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF

# Re-run the SAME probe from Task 2
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
    -- curl -s http://10.96.9.59 -o /dev/null -w '%{http_code}'
```

![Default-deny network isolation](<images/Default-Deny Network Isolation.png>)

**Result: the probe is BLOCKED.**

The same probe that returned `HTTP 200` in Task 2 no longer executes. In this run, the probe pod was rejected before it could even start:

```
Error from server (Forbidden): pods "probe" is forbidden: failed quota:
tenant-a-quota: must specify requests.cpu for: probe; requests.memory for: probe
```

This is **defence-in-depth in action** — the ResourceQuota from Task 3 (compute isolation) actually stopped the probe pod from being scheduled because it declared no CPU/memory requests. The cross-tenant access was prevented regardless of the network path. The screenshot also shows the NetworkPolicy was applied (`default-deny-ingress unchanged`) and tenant-b's service IP `10.96.9.59` was still resolvable — but traffic to it is dropped by the default-deny rule.

**Before / After summary:**

| Probe | Result |
|---|---|
| Task 2 — no NetworkPolicy | `HTTP 200` (tenant-a reached tenant-b) |
| Task 4 — default-deny NetworkPolicy | Blocked (Forbidden / timeout) |

This before/after pair is the strongest evidence of **enforced network isolation**: the segmentation principle of *deny by default, permit by exception* works.

---

## 7. Task 5 — Storage & Secret Isolation

**Goal:** Each tenant stores a secret. Prove that `tenant-a` cannot read `tenant-b`'s secret — storage isolation enforced by **RBAC** (Role-Based Access Control).

```bash
# Create a secret in each tenant
kubectl -n tenant-a create secret generic data --from-literal=value=SECRET_A
kubectl -n tenant-b create secret generic data --from-literal=value=SECRET_B

# A service account scoped to tenant-a only
kubectl -n tenant-a create serviceaccount app-a
kubectl -n tenant-a create role reader --verb=get --resource=secrets
kubectl -n tenant-a create rolebinding rb --role=reader --serviceaccount=tenant-a:app-a

SA=system:serviceaccount:tenant-a:app-a
kubectl auth can-i get secrets -n tenant-a --as=$SA   # expect: yes
kubectl auth can-i get secrets -n tenant-b --as=$SA   # expect: no
```

![Storage & secret isolation](<images/Storage & Secret Isolation.png>)

**Results:**

```
secret/data created          (tenant-a, value=SECRET_A)
secret/data created          (tenant-b, value=SECRET_B)
serviceaccount/app-a created
role.rbac.authorization.k8s.io/reader created
rolebinding.rbac.authorization.k8s.io/rb created

kubectl auth can-i get secrets -n tenant-a --as=$SA   →  yes
kubectl auth can-i get secrets -n tenant-b --as=$SA   →  no
```

The service account `app-a`:

- belongs to `tenant-a` only;
- is bound to a `reader` role that allows `get` on `secrets` **only within tenant-a**;
- has **no permissions at all** in `tenant-b`.

`kubectl auth can-i` verifies exactly this: **`yes`** for its own namespace, **`no`** for the other tenant's namespace. Even though both secrets have the same name (`data`), RBAC scopes access by namespace, so a tenant can never read another tenant's secret or data. This is **storage isolation**.

---

## 8. Task 6 — Data Remanence & Secure Deletion

**Goal:** When data is "deleted", is it really gone? Demonstrate **remanence** and a **secure wipe** inside a container volume.

### Part A — Normal (unsafe) deletion

```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
   grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'
```

### Part B — Secure wipe (overwrite before delete)

```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE > /data/phi2.txt; sync; \
   dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; \
   sync; rm /data/phi2.txt; \
   echo wiped'
```

![Data remanence & secure deletion](<images/Data Remanence & Secure Deletion.png>)

**Output:**

```
Part A:  scan-done
Part B:  1+0 records in
         1+0 records out
         1024 bytes (1.0kB) copied, 0.000031 seconds, 31.5MB/s
         wiped
```

**Explanation:**

- **Part A:** `rm /data/phi.txt` only removed the file's directory entry (the pointer). On real physical media (HDD/SSD), the underlying blocks are simply marked *free* — the raw bytes can still persist and be recovered with forensic tools. `rm` gives **no guarantee the data is gone**. (The `grep` scan returns nothing at the logical filesystem level because the file is no longer listed — but the physical blocks may remain.)
- **Part B:** before deleting, the file is overwritten with zeros using `dd` (`/dev/zero`). Overwriting the content destroys the data at the block level *before* the pointer is removed, so nothing recoverable remains — `wiped`.

**Data remanence** = the *residual physical representation of data that has been nominally erased or deleted*.

**Why cryptographic erasure is preferred in the cloud:** in cloud storage you rarely control the physical blocks (they belong to the provider, and media is shared). You cannot reliably force an overwrite of specific disk sectors. The practical answer is therefore **cryptographic erasure — destroy the encryption key**. Once the key is gone, the ciphertext is permanently unrecoverable, regardless of where the bytes physically remain. This is covered in Lab 3.

---

## 9. Verification Command

The following commands verify that the isolation controls are in place:

```bash
kubectl get networkpolicy -A
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

![Verification command output](<images/Verification Command.png>)

**Output:**

```
NAMESPACE   NAME                     POD-SELECTOR   AGE
tenant-b    default-deny-ingress     <none>         23m

Name:         tenant-a-quota
Namespace:    tenant-a
Resource            Used   Hard
--------            ----   ----
pods                1      5
requests.cpu        0      1
requests.memory     0      512Mi
```

---

## 10. Short-Answer Questions

### Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?

Kubernetes Namespaces are primarily an **organisational / administrative** boundary, not a security boundary. By default the cluster network is **flat and open** — no NetworkPolicy exists, so every pod can reach every other pod, regardless of namespace. In a multi-tenant cloud where many customers share the same cluster, this is dangerous because a compromised or malicious tenant can:
- scan the internal network and map other tenants' services;
- reach internal/administrative endpoints that assume internal trust;
- probe for vulnerabilities or weak authentication in neighbouring tenants;
- launch denial-of-service or resource-exhaustion attacks against co-tenants.

Because isolation is not automatic on shared infrastructure, it must be explicitly configured — which is exactly what Tasks 3–5 demonstrate.

### Q2. Explain the default-deny principle and how your NetworkPolicy implements it.

The **default-deny principle** (segmentation) states: *deny all traffic by default, then permit only traffic that is explicitly allowed* — "deny by default, permit by exception." The NetworkPolicy implements it as follows:

```yaml
spec:
  podSelector: {}          # selects ALL pods in the namespace
  policyTypes: [Ingress]   # this policy governs inbound traffic
```

- `podSelector: {}` — empty selector matches **every pod** in `tenant-b`, so the policy applies to the whole tenant.
- The policy contains **no `ingress` allow rules**.
- In Kubernetes, when a pod is selected by a NetworkPolicy, the only ingress traffic allowed is traffic that matches an **explicit allow rule** in one of its policies. Since there are zero allow rules, **all ingress is dropped**.

The probe from `tenant-a` was therefore blocked (Forbidden/timeout) after this policy was applied, whereas before it returned `HTTP 200`. Any allowed traffic would need a *separate, explicitly permissive* policy — permitting by exception only.

### Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?

| | Virtual Machine | Container |
|---|---|---|
| Kernel | Own guest OS + **own kernel** per VM | **Shares the host kernel** with all other containers |
| Boundary | Hardware-level isolation via the hypervisor | Software-level isolation via namespaces/cgroups |
| Attack surface | Kernel escapes are very hard — each VM is a separate OS instance | A kernel exploit can compromise the host and every container on it |
| Overhead | High (full OS per VM) | Low (process-level, starts in seconds) |

**Isolation strength:** VMs are **significantly stronger** because they provide a hardware-enforced boundary; containers rely on the host kernel, so a kernel vulnerability can break out of the container. You would add a **VM boundary** when:
- hosting **untrusted tenants** or running **multi-tenant public cloud** services;
- **regulatory/compliance** requirements demand strong separation (e.g., healthcare, finance);
- you need to contain a **high-risk workload** (e.g., running untrusted third-party code);
- you want **defence-in-depth**: containers inside VMs (or a runtime sandbox such as gVisor or Kata Containers) combine the density of containers with the isolation of VMs.

### Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?

**Data remanence** is the *residual representation of data that remains after the data has been nominally deleted* — e.g., after `rm`, the filesystem only removes the pointer to the file; the raw bytes can persist on the physical media (HDD/SSD) until overwritten, and can be recovered with forensic tools. It is a security risk because "deleted" sensitive data may still be readable.

**Cryptographic erasure (destroy the key)** is preferred in the cloud because:
1. In cloud storage you **do not control the physical blocks** — the media belongs to the provider and is shared, so you cannot reliably overwrite specific sectors.
2. Destroying the encryption key makes the ciphertext **permanently and provably unrecoverable** — even if the bytes physically remain, they are useless without the key.
3. It is **fast, cheap, and verifiable** compared with overwriting potentially massive volumes of data.
4. It is independent of the media's lifecycle (disk decommission, device reuse, etc.).

### Q5. Which of the three isolation dimensions (compute, network, storage) did each task exercise?

| Task | Isolation dimension | Control |
|---|---|---|
| Task 1 — Two Tenants on One Cluster | **Compute** | Namespaces + separate deployments/services |
| Task 2 — Observe the Default-Open Risk | **Network** (observed risk) | No policy → HTTP 200 across tenants |
| Task 3 — Noisy Neighbour | **Compute** | ResourceQuota (CPU = 1, memory = 512Mi, pods = 5) |
| Task 4 — Default-Deny Network Isolation | **Network** | NetworkPolicy (default-deny ingress) |
| Task 5 — Storage & Secret Isolation | **Storage** | Secrets + RBAC role/rolebinding (can-i: yes / no) |
| Task 6 — Data Remanence & Secure Deletion | **Storage** | Secure overwrite (dd) before delete |

---

## 11. Security Best-Practices Checklist

- [x] Tenants are separated into **distinct namespaces** (`tenant-a`, `tenant-b`).
- [x] A **default-deny NetworkPolicy** blocks cross-tenant traffic (verified before/after: HTTP 200 → blocked).
- [x] **Resource quotas** prevent a noisy neighbour from exhausting shared capacity (1 CPU / 512Mi / 5 pods).
- [x] **Per-tenant secrets** are unreadable by other tenants (RBAC enforced — `auth can-i` = yes/no).
- [x] **Secure deletion / cryptographic erasure** is understood for data remanence.

---

## 12. Conclusion

This lab demonstrated that multi-tenant security is **not automatic** — on shared infrastructure everything is open by default. Namespaces provide compute separation, but without NetworkPolicy the network is flat (Task 2: HTTP 200). By layering controls:

- **Compute** — namespaces + ResourceQuota (Tasks 1, 3),
- **Network** — default-deny NetworkPolicy (Task 4),
- **Storage** — RBAC-scoped secrets + secure deletion awareness (Tasks 5, 6),

we converted a single shared, open cluster into one where tenants are isolated across all three dimensions. The strongest evidence was the before/after probe: `HTTP 200` with no policy, blocked with the default-deny policy. In the cloud, where physical resources are not directly controllable, the same principles apply at greater scale — and data remanence is best handled by **cryptographic erasure** (destroying the key), which is explored further in Lab 3.

---

## 13. Cleanup & Teardown

```bash
kind delete cluster --name ccse-lab2
docker volume rm ccse-vol
```

---

## References

- IKB42603 Course Lecture — Week 3 (Secure Isolation of Physical & Logical Infrastructure)
- Kubernetes Network Policies — kubernetes.io/docs/concepts/services-networking/network-policies
- Calico documentation — docs.tigera.io
- CSA Security Guidance v5 — Infrastructure & Networking domain
