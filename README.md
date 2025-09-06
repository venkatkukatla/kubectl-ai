
# EKS + kubectl-ai Setup (AWS) — Step-by-step README

> This README documents a repeatable, step-by-step process to install the AWS CLI, kubectl, eksctl, install krew (and the `ai` plugin), create an EKS cluster, and deploy a sample NGINX app behind an NLB. It also shows how to configure `kubectl ai` (GEMINI/OPENAI) and common troubleshooting steps.

---

## Table of contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Variables (change these for your environment)](#variables)
4. [Step-by-step installation & setup](#step-by-step-installation--setup)

   * Install AWS CLI
   * Configure AWS CLI
   * Install kubectl
   * Install eksctl
   * Create EKS cluster
   * Update kubeconfig
   * Deploy NGINX app (Deployment + Service)
   * Install krew and `ai` plugin
   * Configure LLM API keys for `kubectl ai`
5. [Usage examples (`kubectl ai`)](#usage-examples-kubectl-ai)
6. [Troubleshooting (common issues & fixes)](#troubleshooting-common-issues--fixes)
7. [Cleanup](#cleanup)
8. [Notes & Security](#notes--security)
9. [Contributing / License](#contributing--license)

---

## Overview

This README bundles the commands and manifests you provided into a clean, GitHub-ready format. Follow the steps below to go from a fresh Ubuntu-like environment to a running EKS cluster with a sample NGINX deployment that uses an AWS NLB (Network Load Balancer).

---

## Prerequisites

* An **AWS account** with permissions to create EKS clusters, EC2 instances, IAM roles, VPCs and related resources.
* Local machine running Linux (Ubuntu is assumed) with `curl`, `tar`, `unzip`, and `ssh` available.
* `sudo` access on your local machine to install binaries to `/usr/local/bin`.
* (Optional) `jq` for JSON parsing (helpful but not required).
* An SSH key pair for `eksctl` nodegroup SSH access (we use the key name `ec2key` in examples — replace it or create/upload your own).

---

## Variables (change these for your environment)

```bash
CLUSTER_NAME="my-cluster01"
REGION="us-east-1"
NODE_TYPE="t3.medium"
NODES=2
SSH_KEY_NAME="ec2key"   # Must exist in AWS or be imported
```

---

## Step-by-step installation & setup

> Each numbered block below is intended to be run sequentially. Copy/paste commands into your terminal.

### 1) Install AWS CLI v2

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt update && sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install

# verify
aws --version
```

### 2) Configure AWS CLI (credentials & default region)

```bash
aws configure
# Enter: AWS Access Key ID, AWS Secret Access Key, default region name (e.g. us-east-1), default output (e.g. json)
```

> If you prefer to import a keypair into EC2 for SSH access to nodes (used by eksctl `--ssh-public-key`), generate a key locally and import it:

```bash
ssh-keygen -t rsa -b 4096 -C "eks node key" -f ~/.ssh/${SSH_KEY_NAME}
aws ec2 import-key-pair --key-name "${SSH_KEY_NAME}" --public-key-material fileb://~/.ssh/${SSH_KEY_NAME}.pub --region ${REGION}
```

> Or create/upload the key in the AWS console and use that key name in the eksctl command.

### 3) Install `kubectl`

```bash
curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# verify
kubectl version --client
```

### 4) Install `eksctl`

```bash
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# verify
eksctl version
```

### 5) Create an EKS cluster with `eksctl`

> Replace `--ssh-public-key` value with the EC2 keypair name you created or imported.

```bash
eksctl create cluster \
  --name ${CLUSTER_NAME} \
  --region ${REGION} \
  --node-type ${NODE_TYPE} \
  --nodes ${NODES} \
  --nodes-min 1 \
  --nodes-max 2 \
  --with-oidc \
  --ssh-access \
  --ssh-public-key ${SSH_KEY_NAME} \
  --managed
```

> This command may take 10–20 minutes depending on AWS region and resource creation.

### 6) Update kubeconfig (so `kubectl` talks to your cluster)

```bash
aws eks --region ${REGION} update-kubeconfig --name ${CLUSTER_NAME}

# quick checks
kubectl get nodes
kubectl get svc -A
```

### 7) Deploy the sample NGINX application

Create **deployment.yaml** (or apply directly from the snippet below):

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Create **service.yaml** (NLB) — note the AWS annotation for NLB:

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Apply manifests:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# verify
kubectl get deployments
kubectl get svc nginx-service
kubectl get pods -l app=nginx
```

> When Service type=LoadBalancer is created, AWS will provision an NLB because of the annotation. Use `kubectl get svc nginx-service -o wide` or `kubectl describe svc nginx-service` to view the external IP/DNS.

### 8) Install `krew` (plugin manager) and `ai` plugin

Run the krew install block (this is the canonical sequence):

```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
```

Add krew to PATH (append to `~/.bashrc` or your shell rc file):

```bash
echo 'export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# verify
kubectl krew version
```

Install the `ai` plugin (provided `krew` has it):

```bash
kubectl krew install ai

# verify installed plugins
kubectl krew list
```

### 9) Configure your LLM API key(s) for `kubectl ai`

`kubectl ai` can use different LLM providers depending on the plugin and your environment. Example environment variables:

```bash
# Example for Gemini (or other Google-related endpoints)
export GEMINI_API_KEY="your_gemini_api_key_here"

# Example for OpenAI
export OPENAI_API_KEY="your_openai_api_key_here"

# Make them persistent in ~/.bashrc if you want
echo 'export GEMINI_API_KEY="your_gemini_api_key_here"' >> ~/.bashrc
echo 'export OPENAI_API_KEY="your_openai_api_key_here"' >> ~/.bashrc
source ~/.bashrc
```

> Replace the placeholder keys with your real keys. Different LLMs/plugins may require different variable names — consult the plugin docs if available.

---

## Usage examples (`kubectl ai`)

```bash
# Basic: ask to display pods (natural language)
kubectl ai "display pods"

# Explicitly choose LLM provider (if supported):
kubectl ai --llm-provider openai "display pods"

# Ask for explanation or summary
kubectl ai "Summarize the status of all deployments in the default namespace"

# Get a suggested kubectl command
kubectl ai "How do I scale nginx-deployment to 6 replicas?"
```

---

## Troubleshooting — common issues and fixes

* **`Error: unknown flag: --provider`**

  * Use `--llm-provider` (the plugin expects `--llm-provider` not `--provider`) or consult the plugin help: `kubectl ai --help`.

* **`OPENAI_API_KEY environment variable not set`**

  * Export `OPENAI_API_KEY` (or the provider-specific variable) before running the command: `export OPENAI_API_KEY="sk-..."`.

* **`ssh-public-key ec2key not found` (eksctl)**

  * Make sure the key name exists in the AWS EC2 Key Pairs for the region. Use `aws ec2 import-key-pair` or create it in the console.

* **`permission denied` when moving binaries**

  * Use `sudo` for `mv` into `/usr/local/bin` or install to a user-local bin and add to PATH.

* **`kubectl` cannot communicate with cluster\`**

  * Ensure `aws eks update-kubeconfig --region ... --name ...` completed successfully and the `~/.kube/config` contains the cluster context.

* **Long creation times for `eksctl create cluster`**

  * EKS cluster creation is expected to take several minutes — watch CloudFormation or `eksctl` logs. If it fails, inspect the stack in the AWS CloudFormation console.

* **`/bin/bash: warning: shell level (1000) too high, resetting to 1`**

  * This is a bash warning that can occur in some nested shell environments — usually harmless but indicates many nested shells. Restart terminal if needed.

---

## Cleanup

To delete the cluster and associated resources created by `eksctl`:

```bash
eksctl delete cluster --name ${CLUSTER_NAME} --region ${REGION}
```

> Note: deleting the cluster will remove managed nodegroups, but additional resources (like EBS volumes, load balancers, or manually created S3 buckets) might remain — review AWS console for orphaned resources.

---

## Notes & Security

* **Replace placeholder values** (API keys, cluster names, SSH key names) before running commands.
* **Do not commit secrets** (API keys, AWS credentials) into Git. Use environment variables or secret managers.
* When using `kubectl ai` with production clusters, be mindful of RBAC and what information is being sent to external LLM services.

---

## Contributing / License

Feel free to open a PR or suggest improvements to this README. Use a permissive license (e.g. MIT) if you plan to publish it in a public repo.

---

*End of README*
