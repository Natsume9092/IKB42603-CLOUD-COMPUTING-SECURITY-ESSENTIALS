# Lab 0 - Environment Setup

| | |
|---|---|
| **Course** | IKB42603 Cloud Computing Security Essentials |
| **Lab** | Lab 0 - Environment Setup |
| **Name** | MUHAMMAD AKMAL HAKIM BIN MOHD YUZLAN (52215125582) |
| **Date** | 27 July 2026 |

---

## Objective

The objective of Lab 0 is to set up and verify the complete local development environment required for all subsequent labs in the IKB42603 Cloud Computing Security Essentials course. This is a critical prerequisite — every tool installed here will be used in the upcoming labs, and completing this setup ensures a fully functional local environment that operates **without** a real AWS account, credit card, or internet connectivity during lab sessions.

The following components are installed and verified:

| Tool | Purpose | Used In |
|------|---------|---------|
| **Docker** | Runs containers and the LocalStack cloud simulator | All labs |
| **AWS CLI v2** | Sends AWS commands to LocalStack (local AWS emulator) | Labs 1, 3, 5 |
| **kind** | Runs a local Kubernetes cluster inside Docker containers | Labs 1, 2, 4 |
| **kubectl** | Controls and manages the Kubernetes cluster | Labs 1, 2, 4 |
| **OpenSSL** | Encryption, key generation, and certificate management | Lab 3 |
| **oathtool** | Generates MFA / TOTP codes for multi-factor authentication | Lab 4 |
| **Trivy** | Scans container images for security vulnerabilities (run via Docker) | Lab 4 |

---

## Prerequisites

Before beginning the installation, the following system requirements were confirmed:

- **Operating System**: Windows 11
- **Hardware Virtualization**: VT-x enabled in BIOS/UEFI (required for Docker Desktop)
- **WSL 2**: Windows Subsystem for Linux v2 enabled
- **Virtual Machine Platform**: Windows feature enabled
- **Git Bash**: Installed (all lab commands are executed inside Git Bash, not Command Prompt or PowerShell, to ensure compatibility with bash features such as heredocs, `sha256sum`, and single-quoting)
- **Disk Space**: Sufficient free space for Docker images, kind node images, and tool binaries

---

## Step-by-Step Guide

### Step 1: Install Docker

Docker is the foundation of the entire lab environment. It is a containerization platform that allows applications to run in isolated environments called containers. In this course, Docker is used to:

- Run **LocalStack**, a local AWS service emulator (all labs)
- Run **kind** (Kubernetes-in-Docker) clusters (Labs 1, 2, 4)
- Run **Trivy** for vulnerability scanning (Lab 4)
- Run utility containers for testing and development

**Installation Process:**

1. Navigated to the official Docker website at [docker.com](https://www.docker.com).
2. Downloaded Docker Desktop for Windows, selecting the **WSL 2 backend** option when prompted during installation.
3. The WSL 2 backend provides better performance and full Linux kernel compatibility compared to the older Hyper-V backend.
4. After installation completed, the system was rebooted to ensure all Docker services and WSL 2 components are properly initialized.
5. Enabled hardware virtualization (VT-x) in the BIOS/UEFI settings prior to installation — this is a mandatory requirement for running any hypervisor-based virtualization software.

**Verification — Checking the Docker Version:**

```
$ docker --version
Docker version 27.1.1, build 6312585
```

This confirms that Docker Engine version 27.1.1 (build 6312585) is installed and the Docker CLI is accessible from the command line.

**Verification — Running the Hello World Container:**

```
$ docker run --rm hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash
```

The `hello-world` container is a minimal test image that verifies the entire Docker pipeline is functional:

- The Docker CLI (`docker`) can communicate with the Docker daemon (`dockerd`).
- The daemon can pull images from Docker Hub (or a configured registry).
- The daemon can create and run containers from those images.
- The container output is correctly streamed back to the terminal.

The `--rm` flag tells Docker to automatically remove the container after it exits, keeping the system clean.

![Docker Installation](images/install-docker.png)

_Screenshot shows the output of `docker --version` and `docker run --rm hello-world` in a Git Bash terminal._

---

### Step 2: Install AWS CLI v2

The AWS Command Line Interface (CLI) v2 is a unified tool for managing AWS services from the command line. In this course, the AWS CLI is configured to communicate with **LocalStack** (a local AWS emulator) rather than the real AWS cloud. This means:

- **No real AWS account is required.**
- **No credit card is needed.**
- **No internet access is needed after the initial downloads.**
- All AWS API calls remain local to the laptop, making them fast and free.

The AWS CLI is used in Labs 1, 3, and 5 to interact with emulated AWS services such as IAM, S3, Lambda, and more.

**Installation Process (Windows):**

1. Opened a web browser and searched for "install AWS CLI v2 Windows" as recommended in the lab cheatsheet.
2. Navigated to the official AWS CLI MSI installer download page on the AWS website.
3. Downloaded the `AWSCLIV2.msi` installer package for Windows (64-bit).
4. Ran the MSI installer, which installs AWS CLI v2 system-wide and adds it to the system PATH automatically.
5. No reboot was required — opened a new Git Bash terminal to verify the installation.

**Verification:**

```
$ aws --version
aws-cli/2.17.18 Python/3.12.6 Windows/10 exe/AMD64
```

The output confirms:

- **AWS CLI version**: 2.17.18 (the latest v2 release)
- **Python version**: 3.12.6 (the AWS CLI is bundled with its own Python runtime)
- **Platform**: Windows/10 exe/AMD64

![AWS CLI v2 Installation](images/install-aws-cli-v2.png)

_Screenshot shows the output of `aws --version` in a Git Bash terminal, confirming AWS CLI v2.17.18 is installed._

---

### Step 3: Install kind & kubectl

kind (Kubernetes-in-Docker) is a tool for running local Kubernetes clusters using Docker containers as cluster nodes. kubectl is the official Kubernetes command-line tool used to deploy and manage applications on Kubernetes clusters. These tools are used together:

- **kind** creates and manages the local Kubernetes cluster (it runs each cluster node as a Docker container).
- **kubectl** communicates with the cluster's API server to perform operations such as inspecting nodes, deploying pods, and configuring networking.

These tools are used in Labs 1, 2, and 4 for Kubernetes security exercises including Pod security policies, network policies, and secret management.

**Installation Process (Windows using Chocolatey):**

Chocolatey is a package manager for Windows that simplifies software installation from the command line. Both kind and kubectl were installed via Chocolatey:

```
$ choco install kind
```

This command downloads and installs the latest stable version of kind. Chocolatey handles PATH configuration automatically.

```
$ choco install kubernetes-cli
```

Similarly, this installs the kubectl binary and makes it available system-wide.

**Alternative Installation (without Chocolatey):**

If Chocolatey is not available, the cheatsheet provides alternative methods:

- **kind**: Download the binary from the kind GitHub releases page (`kind v0.23.0` for Linux/Mac)
- **kubectl**: On Linux, install via `sudo snap install kubectl --classic`

**Verification — kind:**

```
$ kind --version
kind v0.23.0 go1.22.5 windows/amd64
```

This confirms kind version 0.23.0, built with Go 1.22.5, running on Windows/AMD64.

**Verification — kubectl:**

```
$ kubectl version --client
Client Version: v1.30.3
Kustomize Version: v5.0.4-0.20230601165947-4ce0d4c5b0cf
```

The `--client` flag checks only the kubectl client version without contacting a server (since no cluster is running yet). This confirms:

- **Client version**: v1.30.3
- **Kustomize version**: v5.0.4 (Kustomize is built into kubectl for Kubernetes manifest customization)

![kind & kubectl Installation](images/install-kind-kubectl.png)

_Screenshot shows the output of `kind --version` and `kubectl version --client` in a Git Bash terminal._

---

### Step 4: Install Helper Tools (OpenSSL, oathtool, Trivy)

These helper tools provide specialized functionality used in specific labs:

#### OpenSSL

**Purpose (Lab 3):** OpenSSL is a robust, full-featured open-source toolkit implementing the Secure Sockets Layer (SSL) and Transport Layer Security (TLS) protocols. In Lab 3, it is used for:

- Generating encryption keys (public/private key pairs)
- Creating self-signed X.509 certificates
- Signing certificate signing requests (CSRs)
- Computing file hashes and digests

**Installation:** On Windows, OpenSSL comes bundled with Git Bash — no separate installation is required. It is located at `/usr/bin/openssl` within the Git Bash environment.

**Verification:**

```
$ openssl version
OpenSSL 3.2.1 30 Jan 2024 (Library: OpenSSL 3.2.1 30 Jan 2024)
```

This confirms OpenSSL version 3.2.1 (released 30 January 2024) is available.

#### oathtool

**Purpose (Lab 4):** oathtool (part of the OATH Toolkit) generates HMAC-based One-Time Passwords (HOTP) and Time-based One-Time Passwords (TOTP). In Lab 4, it is used to simulate MFA (Multi-Factor Authentication) codes for testing AWS IAM identity providers and access policies.

**Installation:** On Windows, oathtool can be used via WSL (Ubuntu). Alternatively, a phone authenticator app can be used as a substitute. For this setup, WSL will be used when required in Lab 4.

**Verification:** oathtool verification will be performed in Lab 4 when it is first needed.

#### Trivy

**Purpose (Lab 4):** Trivy is a comprehensive and versatile security scanner. In Lab 4, it scans Docker container images for:

- Operating system vulnerabilities (Alpine, Debian, Ubuntu, etc.)
- Application library vulnerabilities (npm, pip, gem, etc.)
- Misconfigurations in Infrastructure as Code (IaC) files

**Installation:** No installation is needed. Trivy is run directly via Docker in Lab 4 using the command:

```
docker run --rm aquasec/trivy image <image-name>
```

This approach keeps the host system clean and ensures the latest version of Trivy is always used.

**Combined Verification:**

![Helper Tools](images/helper-tools.png)

_Screenshot shows the output of `openssl version` confirming OpenSSL 3.2.1 is available in the environment._

---

### Step 5: Start & Verify LocalStack

LocalStack is a cloud service emulator that runs in a single Docker container. It provides a local, fully-functional AWS environment that mimics real AWS services (IAM, S3, Lambda, DynamoDB, etc.) without requiring any connection to the AWS cloud. This is the core of the course's "no cloud account needed" approach.

**Starting LocalStack:**

The LocalStack container is started with the following command:

```
$ docker run -d --name localstack -p 4566:4566 localstack/localstack
```

Let us break down what each part of this command does:

| Part | Meaning |
|------|---------|
| `docker run` | Creates and starts a new container from an image |
| `-d` | Detached mode — runs the container in the background |
| `--name localstack` | Assigns the name "localstack" to the container for easy reference |
| `-p 4566:4566` | Maps port 4566 on the host to port 4566 in the container (the LocalStack API port) |
| `localstack/localstack` | The Docker image name on Docker Hub |

**First-run behavior:** The first time this command runs, Docker pulls the `localstack/localstack` image from Docker Hub (approx. 600 MB). Subsequent starts use the cached image and complete almost instantly.

**Managing LocalStack:**

The cheatsheet provides several commands for managing the LocalStack container throughout the course:

| Action | Command |
|--------|---------|
| **Start** (if already created) | `docker start localstack` |
| **Stop** | `docker stop localstack` |
| **Remove completely** | `docker rm -f localstack` |

**Verification — Health Check:**

```
$ curl http://localhost:4566/_localstack/health
{"services": {...}, "version": "3.5.0"}
```

The `/health` endpoint returns a JSON response indicating:

- **services**: A list of all emulated AWS services and their status (running/available)
- **version**: The LocalStack version running (3.5.0)

A successful response confirms that:

1. The LocalStack container is running.
2. The container's port 4566 is correctly mapped to the host.
3. The LocalStack process inside the container has initialized all services.
4. The HTTP endpoint is reachable and responding to requests.

![Starting LocalStack](images/start-localstack.png)

_Screenshot shows the output of the `docker run` command for starting LocalStack, showing the container ID being returned._

![LocalStack Health Check](images/localstack-health.png)

_Screenshot shows the output of the `curl` health check command, returning the JSON health status from LocalStack._

---

### Step 6: One-Time AWS CLI Configuration

Since LocalStack emulates AWS services locally, it does not perform real authentication — it accepts any credentials. However, the AWS CLI requires some credentials to be configured before it can send requests. Dummy values are set once so the CLI stops prompting for them.

**Configuration Commands:**

```
$ aws configure set aws_access_key_id test
$ aws configure set aws_secret_access_key test
$ aws configure set region us-east-1
```

These commands write the following values to the AWS CLI configuration file (`~/.aws/config`) and credentials file (`~/.aws/credentials`):

| Setting | Value | Purpose |
|---------|-------|---------|
| `aws_access_key_id` | `test` | Dummy access key ID for LocalStack |
| `aws_secret_access_key` | `test` | Dummy secret access key for LocalStack |
| `region` | `us-east-1` | Default AWS region (LocalStack defaults to this) |

**Creating the Endpoint Variable:**

To avoid typing `--endpoint-url=http://localhost:4566` with every AWS CLI command, an environment variable is set in the current session:

```
$ EP='--endpoint-url=http://localhost:4566'
```

Now, any AWS CLI command can be run with `aws $EP <command>` instead of the full form.

**Tip:** The cheatsheet also mentions `awslocal` as an optional shortcut — a Python package (`pip install awscli-local`) that provides an `awslocal` command which automatically points to the local endpoint. Both approaches achieve the same result.

**Verification — Proving Communication with LocalStack:**

```
$ aws $EP sts get-caller-identity
{
    "UserId": "test",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:user/test"
}
```

The `sts get-caller-identity` AWS API call returns information about the currently authenticated identity. The response proves:

1. The AWS CLI is successfully communicating with **LocalStack** (not AWS), because:
   - The Account is `000000000000` (LocalStack's default account ID, not a real AWS account ID).
   - The UserId is `test` (our dummy credential).
   - The ARN follows the pattern `arn:aws:iam::000000000000:user/test`.
2. The endpoint URL configuration is correct.
3. LocalStack's IAM service emulation is functioning.

![AWS CLI Configuration](images/one-time-aws-cli-config.png)

_Screenshot shows the AWS CLI configuration commands and the successful `sts get-caller-identity` response from LocalStack._

---

### Step 7: Create & Verify Kubernetes Cluster (kind)

A Kubernetes cluster is created locally using kind (Kubernetes-in-Docker). This cluster runs entirely inside Docker containers — each cluster node is a Docker container that behaves like a virtual machine running Kubernetes.

**Why kind?**

- **Lightweight**: Runs on a laptop without requiring a full VM hypervisor.
- **Fast**: Creates a cluster in under 2 minutes.
- **Ephemeral**: Clusters can be created, used, and destroyed easily — perfect for lab exercises.
- **Compatible**: Supports all standard Kubernetes APIs, making it suitable for real Kubernetes security exercises.

**Creating the Cluster:**

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

The output shows each step kind performs:

| Step | What Happens |
|------|-------------|
| **Ensuring node image** | kind checks for the required node image (`kindest/node:v1.30.0`) locally and pulls it if missing |
| **Preparing nodes** | The node container(s) are created from the image |
| **Writing configuration** | The Kubernetes cluster configuration (kubeconfig) is written to `~/.kube/config` |
| **Starting control-plane** | The Kubernetes control plane components (API server, controller manager, etc.) are initialized |
| **Installing CNI** | A Container Network Interface plugin is installed for pod networking |
| **Installing StorageClass** | A default StorageClass is created for persistent volume claims |

The cluster is named `ccse` (CCSE = Cloud Computing Security Essentials) and a new kubectl context `kind-ccse` is automatically created and set as the current context.

**Verification — Cluster Info:**

```
$ kubectl cluster-info --context kind-ccse
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

This confirms:

- The Kubernetes API server is accessible at `https://127.0.0.1:6443` (localhost, no external network needed).
- The CoreDNS service (internal DNS for the cluster) is operational.
- The `--context kind-ccse` flag explicitly targets the `ccse` cluster (useful when multiple clusters are configured).

**Verification — Nodes:**

```
$ kubectl get nodes
NAME                   STATUS   ROLES           AGE   VERSION
ccse-control-plane    Ready    control-plane   75s   v1.30.0
```

The output shows:

- **Name**: `ccse-control-plane` (the single node acting as the cluster's control plane)
- **Status**: `Ready` — all cluster components are healthy and the node is accepting workloads
- **Roles**: `control-plane` — this node runs the Kubernetes control plane components
- **Age**: 75 seconds — the cluster was just created
- **Version**: `v1.30.0` — the Kubernetes version running on the node

**Managing the Cluster:**

| Action | Command |
|--------|---------|
| **Create cluster** | `kind create cluster --name ccse` |
| **List clusters** | `kind get clusters` |
| **Delete cluster** | `kind delete cluster --name ccse` |
| **Delete all clusters** | `kind delete clusters --all` |

![Creating Cluster](images/create-cluster.png)

_Screenshot shows the full output of the `kind create cluster --name ccse` command, confirming all steps completed successfully._

![Checking Cluster](images/check-cluster.png)

_Screenshot shows the output of `kubectl cluster-info --context kind-ccse` and `kubectl get nodes`, confirming the cluster is operational._

---

## Pre-Lab Verification Checklist

All items in the pre-lab verification checklist have been completed and confirmed passing:

| # | Check | Status | Verification Command |
|---|-------|--------|---------------------|
| 1 | `docker --version` prints a version, and `docker run hello-world` works | ✅ Pass | `docker --version` → `27.1.1`, `docker run --rm hello-world` → Hello from Docker! |
| 2 | `aws --version` prints `aws-cli/2.x` | ✅ Pass | `aws --version` → `aws-cli/2.17.18` |
| 3 | `kind --version` and `kubectl version --client` both work | ✅ Pass | `kind v0.23.0`, `kubectl v1.30.3` |
| 4 | LocalStack starts and `curl .../health` responds | ✅ Pass | Health endpoint returns `{"services": {...}, "version": "3.5.0"}` |
| 5 | `aws $EP sts get-caller-identity` returns an identity | ✅ Pass | Returns Account `000000000000`, ARN ending in `:user/test` |
| 6 | `kind create cluster` works and `kubectl get nodes` shows a node | ✅ Pass | Cluster `ccse` created, one `Ready` control-plane node |
| 7 | (Windows) Working inside Git Bash or WSL | ✅ Pass | All commands executed in **Git Bash** |

---

## Troubleshooting Reference

The following troubleshooting guide from the cheatsheet documents common issues and their resolutions. None of these issues were encountered during this setup, but they are documented here for future reference during lab sessions.

| Symptom | Fix |
|---------|-----|
| "Cannot connect to the Docker daemon" | Docker Desktop is not running — start it. On Linux, run the `usermod` command and re-login. |
| Docker won't start / very slow | Enable virtualization in BIOS. On Windows, enable WSL 2 + Virtual Machine Platform. |
| Port 4566 already in use | A LocalStack container is already running: `docker rm -f localstack`, then start again. |
| `aws`: "Could not connect to the endpoint URL" | LocalStack is not running, or you forgot `--endpoint-url` / `$EP`. Start LocalStack and retry. |
| "aws: command not found" / "kubectl not found" | The tool is not installed or not on PATH. Re-run the install step; open a new terminal. |
| heredoc / sha256sum errors on Windows | You are in PowerShell/CMD. Switch to Git Bash or WSL. |
| `kind create cluster` fails | Docker not running, or low memory. Ensure Docker has ≥ 4 GB RAM and is started. |
| Image download is slow in the lab | Ask the instructor to pre-pull images, or run the `docker run/pull` commands before class on Wi-Fi. |
| MFA/TOTP code always fails (Lab 4) | Your system clock is out of sync — enable automatic time synchronization, then retry. |
| NetworkPolicy not blocking (Lab 2) | The cluster needs Calico (the Lab 2 setup installs it). Wait until `calico-node` is Ready. |

---

## Handy One-Liners

The cheatsheet provides several convenient one-liner commands for managing the lab environment efficiently during lab sessions:

**Start everything for a session:**
```bash
docker start localstack 2>/dev/null || docker run -d --name localstack -p 4566:4566 localstack/localstack
EP='--endpoint-url=http://localhost:4566'
```
This command either starts an existing LocalStack container (if it was stopped) or creates a new one if it does not exist. The `2>/dev/null` suppresses the "No such container" error message. The `$EP` variable is then set for the session.

**See what is running:**
```bash
docker ps
kind get clusters
```
These commands provide a quick overview of running Docker containers and existing kind clusters.

**Clean up everything (frees disk space):**
```bash
docker rm -f localstack
kind delete clusters --all
docker system prune -f
```
This sequence removes the LocalStack container, deletes all kind clusters, and prunes unused Docker images, containers, and networks. The cheatsheet notes that removing containers and clusters is safe — they can be recreated from lab commands at any time. Lab report files (screenshots, outputs) are saved on the laptop, not inside containers.

---

## Security Tips & Best Practices

The cheatsheet provides several important security and usage recommendations:

1. **Git Bash / WSL Requirement**: All lab commands must be executed inside **Git Bash** or **WSL (Ubuntu)** on Windows. Command Prompt and PowerShell do not support bash features such as:
   - Heredocs (`cat << EOF > file`)
   - `sha256sum` checksums
   - Single-quoting behavior
   - Environment variable syntax (`$VAR`)

2. **LocalStack Credentials**: The dummy credentials (`test`/`test`) are acceptable because LocalStack runs locally and does not expose services to the internet. These are not real AWS credentials and pose no security risk.

3. **No Real AWS Account**: All labs are designed to work entirely with LocalStack on localhost. Students should never use real AWS credentials in this course. The `--endpoint-url=http://localhost:4566` flag ensures all AWS CLI commands target the local emulator.

4. **Container Cleanup**: Removing containers and clusters is entirely safe. All lab work and report files are saved on the local filesystem, not inside containers. Recreating containers from the provided commands takes less than a minute.

5. **Optional awslocal Shortcut**: For convenience, the `awslocal` command (`pip install awscli-local`) can be installed to further simplify AWS CLI usage. When installed, `awslocal` automatically includes the correct endpoint URL, so `awslocal sts get-caller-identity` works without the `$EP` variable.

---

## Summary

All components of the Lab 0 environment have been successfully installed, configured, and verified:

| Component | Version / Status | Verification |
|-----------|-----------------|--------------|
| **Docker** | v27.1.1 ✅ | `docker --version` and `docker run hello-world` |
| **AWS CLI v2** | v2.17.18 ✅ | `aws --version` |
| **kind** | v0.23.0 ✅ | `kind --version` |
| **kubectl** | v1.30.3 ✅ | `kubectl version --client` |
| **OpenSSL** | v3.2.1 ✅ | `openssl version` |
| **oathtool** | (WSL — Lab 4) ⏳ | Will be verified in Lab 4 |
| **Trivy** | (Docker — Lab 4) ⏳ | Will be verified in Lab 4 |
| **LocalStack** | v3.5.0 ✅ | `curl localhost:4566/_localstack/health` |
| **AWS CLI → LocalStack** | Connected ✅ | `aws $EP sts get-caller-identity` |
| **kind cluster (ccse)** | Kubernetes v1.30.0 ✅ | `kubectl get nodes` shows Ready |

**Conclusion:** The environment is fully prepared and verified for Labs 1 through 5. All 7 pre-lab checklist items pass, LocalStack is running and responding to AWS API calls, and the Kubernetes cluster is operational with a Ready control-plane node. The lab sessions can proceed without requiring any internet connectivity or real cloud accounts.
