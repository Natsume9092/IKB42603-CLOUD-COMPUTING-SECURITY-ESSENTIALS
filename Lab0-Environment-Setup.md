# Lab 0 - Environment Setup

| | |
|---|---|
| **Course** | IKB42603 Cloud Computing Security Essentials |
| **Lab** | Lab 0 - Environment Setup |
| **Name** | MUHAMMAD AKMAL HAKIM BIN MOHD YUZLAN (52215125582) |
| **Date** | 27 July 2026 |

---

## Objective

The objective of Lab 0 is to set up and verify the local development environment required for all subsequent labs in the IKB42603 Cloud Computing Security Essentials course. This includes installing and configuring Docker, AWS CLI v2, Kubernetes tooling (kind & kubectl), helper tools (OpenSSL, oathtool), and verifying that LocalStack (a local AWS simulator) and a local Kubernetes cluster can run correctly. Completing this setup ensures a fully functional local environment that does not require a real AWS account or internet connectivity during lab sessions.

---

## Step-by-Step Guide

### Step 1: Install Docker

Docker is the foundation of the lab environment. It runs containers and the LocalStack cloud simulator used throughout all labs.

**Installation:**
- Downloaded and installed Docker Desktop from [docker.com](https://www.docker.com) with the WSL 2 backend.
- Rebooted the system after installation.

**Verification:**

```
$ docker --version
Docker version 27.1.1, build 6312585

$ docker run --rm hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

![Docker Installation](install%20docker.png)

---

### Step 2: Install AWS CLI v2

The AWS CLI v2 is used to send AWS commands to LocalStack in Labs 1, 3, and 5. No real AWS account is needed — the CLI is pointed at the local endpoint.

**Installation:**
- Downloaded and ran the AWS CLI MSI installer for Windows from the AWS website.

**Verification:**

```
$ aws --version
aws-cli/2.17.18 Python/3.12.6 Windows/10 exe/AMD64
```

![AWS CLI v2 Installation](install%20aws%20cli%20v2.png)

---

### Step 3: Install kind & kubectl

kind (Kubernetes-in-Docker) runs a local Kubernetes cluster inside Docker containers. kubectl controls the cluster.

**Installation (using Chocolatey):**

```
$ choco install kind
$ choco install kubernetes-cli
```

**Verification:**

```
$ kind --version
kind v0.23.0 go1.22.5 windows/amd64

$ kubectl version --client
Client Version: v1.30.3
Kustomize Version: v5.0.4-0.20230601165947-4ce0d4c5b0cf
```

![kind & kubectl Installation](install%20kind%20&%20kubectl.png)

---

### Step 4: Install Helper Tools (OpenSSL, oathtool, Trivy)

Helper tools are used in later labs for encryption, MFA/TOTP codes, and container vulnerability scanning.

| Tool | Status |
|------|--------|
| **OpenSSL** | Comes pre-installed with Git Bash on Windows. No separate install needed. |
| **oathtool** | Will be used via WSL or a phone authenticator app in Lab 4. |
| **Trivy** | No install needed — runs via Docker in Lab 4. |

**Verification:**

```
$ openssl version
OpenSSL 3.2.1 30 Jan 2024 (Library: OpenSSL 3.2.1 30 Jan 2024)
```

![Helper Tools](helper%20tools.png)

---

### Step 5: Start & Verify LocalStack

LocalStack emulates AWS services locally. It runs as a Docker container.

**Start LocalStack:**

```
$ docker run -d --name localstack -p 4566:4566 localstack/localstack
```

This command downloads the LocalStack image (first run only) and starts the container in detached mode on port 4566.

![Starting LocalStack](start%20localstack.png)

**Verify LocalStack health:**

```
$ curl http://localhost:4566/_localstack/health
{"services": {...}, "version": "3.5.0"}
```

The health endpoint confirms LocalStack is running and its services are available.

![LocalStack Health Check](localstack%20health.png)

---

### Step 6: One-Time AWS CLI Configuration

LocalStack accepts any credentials. Dummy values are configured once so the CLI stops prompting for them.

```
$ aws configure set aws_access_key_id test
$ aws configure set aws_secret_access_key test
$ aws configure set region us-east-1

$ EP='--endpoint-url=http://localhost:4566'

$ aws $EP sts get-caller-identity
{
    "UserId": "test",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:user/test"
}
```

The `sts get-caller-identity` command proves the CLI is successfully communicating with LocalStack.

![AWS CLI Configuration](one-time%20AWS%20CLI%20configuration.png)

---

### Step 7: Create & Verify Kubernetes Cluster (kind)

A local Kubernetes cluster named `ccse` is created using kind.

**Create the cluster:**

```
$ kind create cluster --name ccse
Creating cluster "ccse" ...
 ✓ Ensuring node image (kindest/node:v1.30.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-ccse"
You can now use your cluster with:

kubectl cluster-info --context kind-ccse
```

![Creating Cluster](create%20cluster.png)

**Verify the cluster:**

```
$ kubectl cluster-info --context kind-ccse
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

$ kubectl get nodes
NAME                   STATUS   ROLES           AGE   VERSION
ccse-control-plane    Ready    control-plane   75s   v1.30.0
```

![Checking Cluster](check%20cluster.png)

---

## Pre-Lab Verification Checklist

All items verified and passing:

- [x] `docker --version` prints a version, and `docker run hello-world` works.
- [x] `aws --version` prints `aws-cli/2.x`.
- [x] `kind --version` and `kubectl version --client` both work.
- [x] LocalStack starts and `curl .../health` responds.
- [x] `aws $EP sts get-caller-identity` returns an identity.
- [x] `kind create cluster` works and `kubectl get nodes` shows a node.
- [x] Working inside Git Bash (Windows) for all lab commands.

---

## Notes

- All lab commands were executed inside **Git Bash** to ensure compatibility with bash features (heredocs, sha256sum, single-quoting).
- Hardware virtualization (VT-x) was verified as enabled in BIOS.
- Docker Desktop is configured with the **WSL 2 backend**.
- No real AWS account was used — all commands target the local LocalStack instance via the `$EP` endpoint variable.

---

## Troubleshooting Encountered

| Issue | Resolution |
|-------|-----------|
| *None encountered* | All tools installed and verified successfully on the first attempt. |

---

## Summary

All components of the Lab 0 environment have been successfully installed, configured, and verified:

| Component | Status |
|-----------|--------|
| Docker | ✅ Installed & verified |
| AWS CLI v2 | ✅ Installed & verified |
| kind | ✅ Installed & verified |
| kubectl | ✅ Installed & verified |
| OpenSSL | ✅ Available (via Git Bash) |
| LocalStack | ✅ Running & healthy |
| AWS CLI → LocalStack | ✅ Connected & responding |
| kind cluster (ccse) | ✅ Created & nodes ready |

The environment is fully prepared for Labs 1 through 5.
