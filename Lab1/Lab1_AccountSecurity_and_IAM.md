# LAB 1 — Cloud Account Security, Identity & Access Management (IAM)

| | |
|---|---|
| **Course** | IKB42603 Cloud Computing Security Essentials |
| **Institution** | Universiti Kuala Lumpur Malaysian Institute of Information Technology (UniKL MIIT) |
| **Lab** | Session A (Week 1) — LocalStack IAM · Session B (Week 2) — Kubernetes RBAC |
| **Name** | MUHAMMAD AKMAL HAKIM BIN MOHD YUZLAN (52215125582) |

---

## 1. Objective

The objective of this lab is to build a hands-on understanding of cloud identity governance and the principle of **least privilege**. Using LocalStack (an AWS-compatible simulator) running locally in Docker, the lab demonstrates how to:

1. Stand up a local cloud lab environment using Docker and LocalStack without touching a real cloud provider.
2. Replace all-powerful **root** usage with scoped IAM users, groups and policies.
3. Create and test fine-grained permissions and understand the difference between what an identity is *allowed* to do versus *denied*.
4. Implement and verify **Role-Based Access Control (RBAC)** in a local Kubernetes cluster created with `kind`.
5. Reason about credential hygiene, MFA, access keys and blast-radius reduction.

> **Security note** : Nothing in this lab connects to a real cloud provider. LocalStack emulates AWS APIs locally and `kind` runs Kubernetes inside Docker on the laptop itself.

---

## 2. Session A (Week 1) — Cloud Identity with LocalStack

### 2.1 One-Time Environment Setup

The environment was prepared by verifying Docker and starting the LocalStack container, then pointing the AWS CLI at it using dummy credentials.

```bash
docker --version
docker run -d --name localstack -p 4566:4566 localstack/localstack
curl http://localhost:4566/_localstack/health

# Configure dummy credentials (LocalStack accepts any value)
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
```

The variable `EP` was defined once to keep the commands short:

```bash
EP='--endpoint-url=http://localhost:4566'
```

To prove that the CLI was talking to LocalStack and **not** real AWS, `sts get-caller-identity` was run. The all-zero account number confirms this is the local simulator.

**Key output — `sts get-caller-identity` (proves we are operating as root in LocalStack):**

![One-Time Environment Setup](images/One-Time%20Environment%20Setup.png)

**Observation** : At this point the session is operating as the **root** user (`...:root`) — the exact all-powerful identity the lab wants us to migrate away from.

---

### 2.2 Task 1 — Map the Cloud Identity Landscape

Before creating anything, the building blocks of cloud identity were mapped:

| Concept | AWS term | Purpose (in my own words) |
|---|---|---|
| All-powerful owner | Root user | The account owner identity created with the AWS account. It has unrestricted access to everything and should only be used for account-level tasks, never daily operations. |
| Human/app identity | IAM User | A long-lived identity that represents a person or an application, with its own credentials to make API calls. |
| Permission bundle | IAM Policy | A JSON document that defines a set of allowed or denied actions and resources; the actual "rules" of permission. |
| Collection of users | IAM Group | A container that bundles several IAM users together so one policy can be attached once and inherited by every member. |
| Temporary identity | IAM Role | An identity without long-term credentials; it is assumed to obtain short-lived, automatically expiring credentials. |

---

### 2.3 Task 2 — Create a Least-Privilege Admin (Stop Using Root)

The root user is a liability — if its credentials leak, the entire account is compromised. A dedicated admin identity was created, and the admin permission was attached to a **group**, never directly to a user.

```bash
# 2.1 Create a group and attach an admin policy to the GROUP
aws $EP iam create-group --group-name Admins
aws $EP iam attach-group-policy --group-name Admins \
    --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# 2.2 Create the personal admin user
aws $EP iam create-user --user-name CloudAdmin_Kazuki

# 2.3 Put the user in the group (permissions flow from the group)
aws $EP iam add-user-to-group --group-name Admins \
    --user-name CloudAdmin_Kazuki

# 2.4 Verify the membership
aws $EP iam get-group --group-name Admins
```

**Key output — verification (`get-group Admins`):**

![Create a Least-Privilege Admin (Stop Using Root)](images/Create%20a%20Least-Privilege%20Admin%20(Stop%20Using%20Root).png)

**Security tip applied** : attaching the policy to the group means permissions can be managed and audited at scale — change the group once, and every member updates automatically.

---

### 2.4 Task 3 — Enforce Least Privilege with a Scoped Policy

A read-only user was created to represent a teammate (an analyst) who should look at data but **never modify it**. Instead of any admin power, this user received only one scoped policy: `AmazonS3ReadOnlyAccess`.

```bash
# 3.1 Create a read-only user
aws $EP iam create-user --user-name Analyst_AkmalHakim

# 3.2 Attach a scoped, read-only policy (S3 read only)
aws $EP iam attach-user-policy --user-name Analyst_AkmalHakim \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 3.3 List what the user can do
aws $EP iam list-attached-user-policies --user-name Analyst_AkmalHakim
```

**Key output — verification (`list-attached-user-policies`):**

![Enforce Least Privilege with a Scoped Policy](images/Enforce%20Least%20Privilege%20with%20a%20Scoped%20Policy.png)

**Explanation — blast radius** : If the `Analyst_AkmalHakim` account were stolen, the damage is limited because the account only holds **read** permission on **S3 only**. The attacker could read (spy on) data but cannot delete, overwrite, change permissions, create users, or touch any other service such as EC2 or IAM. Compare this to a stolen admin account, which could do *everything* — wipe all storage, create backdoor users, take over the whole account. This is **blast-radius reduction** : a compromise can only spread as far as the identity's narrow permissions allow, so the potential damage of any single compromised credential is kept small.

---

### 2.5 Task 4 — Credential Hygiene & Access Keys

Programmatic access needs access keys. One key was created for the Analyst, listed, and then **rotated by deactivating it** — demonstrating the lifecycle of a long-lived key.

```bash
# 4.1 Create an access key for the Analyst
aws $EP iam create-access-key --user-name Analyst_AkmalHakim

# 4.2 List access keys (note the AccessKeyId and status)
aws $EP iam list-access-keys --user-name Analyst_AkmalHakim

# 4.3 Rotate: deactivate the old key
aws $EP iam update-access-key --user-name Analyst_AkmalHakim \
    --access-key-id AKIAIOSFODNN7EXAMPLE --status Inactive

# 4.4 Confirm the key is now inactive
aws $EP iam list-access-keys --user-name Analyst_AkmalHakim
```

**Key output — verification (key rotated from `Active` to `Inactive`):**

![Credential Hygiene & Access Keys](images/Credential%20Hygiene%20%26%20Access%20Keys.png)

**Caution applied** : in real AWS, access keys are never created on the root user and never committed to code repositories. Long-lived keys are risky because they do not expire by themselves — **prefer short-lived roles** for temporary access, and rotate keys regularly.

---

## 3. Session B (Week 2) — Enforced Access Control with Kubernetes RBAC

LocalStack teaches the *mechanics* of IAM, but it does not fully enforce policies. Kubernetes RBAC actually **blocks** unauthorised actions, so this session proves access control in a real enforcement engine.

### 3.1 Setup — Create a Local Kubernetes Cluster

```bash
kind create cluster --name ccse-lab1
```

**Key output — cluster creation and verification (`cluster-info` / `get nodes`):**

![Create a Local Kubernetes Cluster](images/Create%20a%20Local%20Kubernetes%20Cluster.png)

The cluster came up successfully on Kubernetes v1.30.0 — the control-plane node is `Ready` (`ccse-lab1-control-plane`).

---

### 3.2 Task 5 — Separate Environments with Namespaces

Namespaces logically isolate environments inside the same cluster. A `dev` and a `prod` namespace were created.

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```

**Key output — namespaces created (`dev` and `prod`):**

![Separate Environments with Namespaces](images/Separate%20Environments%20with%20Namespaces.png)

---

### 3.3 Task 6 — Define a Role and Bind It (Least Privilege)

RBAC has two parts: a **Role** (what permissions exist) and a **RoleBinding** (who gets them). A service account representing a developer was created, a role that can only *read* pods in `dev` was created, and the two were linked with a binding.

```bash
# 6.1 Create a service account to represent a developer
kubectl create serviceaccount dev-user -n dev

# 6.2 Create a Role that allows only get/list/watch on pods
kubectl create role pod-reader -n dev \
    --verb=get,list,watch --resource=pods

# 6.3 Bind the Role to the service account
kubectl create rolebinding dev-user-binding -n dev \
    --role=pod-reader --serviceaccount=dev:dev-user
```

**Key output — all three resources created successfully:**

![Define a Role and Bind It (Least Privilege)](images/Define%20a%20Role%20and%20Bind%20It%20(Least%20Privilege).png)

---

### 3.4 Task 7 — Test That Access Control Works

`kubectl auth can-i` was used to impersonate the developer service account and prove the boundaries. Every result was recorded.

```bash
SA=system:serviceaccount:dev:dev-user

kubectl auth can-i list pods -n dev --as=$SA    # expect yes
kubectl auth can-i delete pods -n dev --as=$SA  # expect no
kubectl auth can-i list pods -n prod --as=$SA   # expect no
```

**Key output — the three `auth can-i` results (yes / no / no):**

![Test That Access Control Works](images/Test%20That%20Access%20Control%20Works.png)

| Check | Command | Result | Expected |
|---|---|---|---|
| Read pods in `dev` | `can-i list pods -n dev` | **yes** | YES (granted by `pod-reader` role) |
| Delete pods in `dev` | `can-i delete pods -n dev` | **no** | NO (not granted by role) |
| Read pods in `prod` | `can-i list pods -n prod` | **no** | NO (role is namespaced to `dev`) |

**Authentication vs. Authorization** : In all three checks the service account successfully **authenticates** — Kubernetes verifies *who* it is (`system:serviceaccount:dev:dev-user`) and recognises it as a valid identity. The `yes` result passes both authentication and authorization. The two `no` results are **authorization** failures: the delete action and the `prod` namespace are outside what the `pod-reader` role permits. So authentication answers *"who are you?"* (passed in all cases) while authorization answers *"are you allowed to do this specific action here?"* (blocked for delete and for prod).

**Security tip applied** : this is least privilege *enforced by the platform* — the developer can do exactly what the role permits and nothing more, even in the same cluster, `prod` is off-limits.

---

## 4. Verification Command (Deliverable)

The RBAC configuration was verified by exporting the binding as YAML.

```bash
kubectl get rolebinding dev-user-binding -n dev -o yaml
```

**Key output — RoleBinding YAML (deliverable):**

![Verification Command](images/Verification%20Command.png)

This confirms the binding links `pod-reader` (roleRef) to `dev-user` (subject) inside the `dev` namespace only.

---

## 5. Short-Answer Questions

### Q1. Why is attaching policies to groups better than attaching them directly to users?

Attaching policies to **groups** instead of individual users keeps permission management centralised, consistent and auditable. When a new person joins a team, the admin simply adds the user to the appropriate group and they automatically inherit the correct permissions — no need to figure out each policy from scratch. When a role changes, the policy is updated **once** on the group and every member is updated at the same time, which prevents drift where different users have slightly different, forgotten permissions. It also makes audits easier: you can see at a glance which groups have which policies and who belongs to them, and it removes the risk of someone attaching overly broad policies "just for this user" and forgetting about them.

### Q2. What is the difference between an IAM User and an IAM Role?

An **IAM User** is a long-lived identity with **permanent credentials** (a password and/or access keys). It represents a specific person or application and exists persistently until deleted. An **IAM Role** is an identity with **no permanent credentials** — it is *assumed* by a user, application or service, and every assumption issues **short-lived, automatically expiring** credentials (typically up to a few hours). A user is directly created and managed, whereas a role is used dynamically: an identity switches into the role to obtain temporary permissions, which is safer for workloads, cross-account access and federated logins.

### Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.

Least privilege means giving an identity the **minimum permissions required** to do its job and nothing more. In this lab, `Analyst_AkmalHakim` was given only the `AmazonS3ReadOnlyAccess` policy — it can read S3 objects but cannot write, delete, or change permissions, and cannot touch any other AWS service. If this account were stolen, the attacker is confined to *reading S3 data*: they cannot destroy data, tamper with it, create new users, or escalate privileges. The **blast radius** — the amount of damage that follows from a single compromised credential — is therefore small and contained. Had the same credential been an admin, one leak could have wiped all resources and taken over the entire account. Least privilege limits the spread of a breach and turns a potential catastrophe into a manageable incident.

### Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?

A **Role** defines *what* is allowed — a set of permissions (verbs such as `get`, `list`, `watch`, `delete` on resources such as `pods`) — but it says nothing about *who* can use them. A **RoleBinding** connects the two: it grants the permissions in a Role to specific subjects (a user, group, or service account). In this lab, the `pod-reader` Role defined "can get/list/watch pods" and the `dev-user-binding` RoleBinding granted that Role to the `dev-user` service account. Both are **namespaced** — the Role and its Binding only apply within the namespace they are created in (here, `dev`). In short: Role = permission rules; RoleBinding = the bridge that attaches those rules to an identity.

### Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?

The developer service account (`system:serviceaccount:dev:dev-user`) failed to access `prod` because the `pod-reader` Role and the `dev-user-binding` RoleBinding were created **only in the `dev` namespace**. RBAC permissions in Kubernetes are scoped to the namespace of the Role/Binding, so a namespaced Role created in `dev` grants nothing in any other namespace — including `prod`, even though it is on the same physical cluster. This demonstrates the principle of **least privilege** (along with **namespace isolation / separation of environments**): the developer can perform exactly the permitted actions in `dev` and nothing anywhere else, so a compromised developer identity cannot touch production data.

---

## 6. Security Best-Practices Checklist

- [x] Root user is not used for daily tasks — a dedicated admin identity (`CloudAdmin_Kazuki`) exists.
- [x] Permissions are granted via groups/roles, not directly to individual users (`Admins` group, `pod-reader` Role).
- [x] At least one least-privilege (read-only) identity was created and tested (`Analyst_AkmalHakim` + `AmazonS3ReadOnlyAccess`).
- [x] Access keys were listed and a rotation (deactivate) was demonstrated (Active → Inactive).
- [x] Kubernetes RBAC blocks an unauthorised action (delete pods in `dev` = no, list pods in `prod` = no).

---

## 7. Cleanup & Teardown

After completing the lab, the temporary resources were removed so nothing is left running:

```bash
# Remove the Kubernetes cluster
kind delete cluster --name ccse-lab1

# Stop and remove LocalStack
docker stop localstack && docker rm localstack
```

---

## 8. Conclusion & Reflection

This lab demonstrated the full journey from "operating as root" to a properly governed cloud identity setup:

- **LocalStack** proved that a realistic cloud IAM environment can be practised offline without any risk of affecting a real account.
- Attaching **AdministratorAccess to a group** (rather than a user) showed how permissions stay manageable and auditable at scale.
- The **Analyst read-only user** concretely demonstrated `least privilege` and how a scoped policy shrinks the blast radius of a stolen credential.
- **Access-key rotation** (Active → Inactive) showed the lifecycle and hygiene of long-lived credentials.
- Finally, **Kubernetes RBAC** demonstrated *enforced* access control: the platform itself returned `no` for the delete action and for the `prod` namespace, proving that policies become reality only when the enforcement engine blocks the unauthorised call.

The key takeaway is that identity is the new security perimeter — granting the minimum permissions, never operating as root for daily work, and verifying access boundaries are habits that protect an entire cloud environment.

---

*End of Lab 1 Report — IKB42603 Cloud Computing Security Essentials*
