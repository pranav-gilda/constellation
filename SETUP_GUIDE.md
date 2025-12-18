* This is your "Engineering Diary." It documents the **how-to** and the **debugging battles** you won. This is gold for interviewers.

# 🛠️ Setup & Engineering Guide

This document details the infrastructure configuration, deployment steps, and "War Stories" (troubleshooting) encountered during the construction of Project Constellation.

---

## 1. AWS Infrastructure Setup

### Compute Layer (EC2)
* **Instance Type:** `g5.2xlarge` (1x NVIDIA A10G, 24GB VRAM).
* **AMI:** Deep Learning OSS Nvidia Driver AMI GPU PyTorch 2.x (Ubuntu 22.04).
    * *Why:* Pre-installed NVIDIA Drivers (535+) and Docker avoid dependency hell.
* **Region:** `us-east-1` (To minimize latency with Bedrock).

### Storage & Filesystem
* **Root Volume:** 150GB `gp3` SSD.
* **Crucial Step:** The AMI defaults to ~65GB. We manually resized the volume in AWS Console and expanded the filesystem live:
    ```bash
    sudo growpart /dev/nvme0n1 1
    sudo resize2fs /dev/nvme0n1p1
    ```

### Networking & Security
* **Security Group:**
    * **Port 22 (SSH):** Restricted to `My IP`.
    * **Port 8000 (vLLM API):** Restricted to `My IP`.
* **SSH Tunneling:** To avoid exposing the API to the open internet, we map the remote port to localhost:
    ```bash
    ssh -i "constellation-hackathon.pem" -L 8000:127.0.0.1:8000 ubuntu@<EC2_PUBLIC_IP>
    ```

---

## 2. Software Deployment (The Safety Node)

We use **Docker** to run vLLM, ensuring a reproducible environment.

### Prerequisites
* HuggingFace Token with access accepted for `meta-llama/Llama-3.2-3B-Instruct`.

### Launch Command
```bash
export HF_TOKEN="your_token_here"

sudo docker run -d \
  --runtime nvidia \
  --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  --env "HUGGING_FACE_HUB_TOKEN=$HF_TOKEN" \
  -p 8000:8000 \
  --ipc=host \
  --name safety_guardrail \
  vllm/vllm-openai:latest \
  --model meta-llama/Llama-3.2-3B-Instruct \
  --gpu-memory-utilization 0.6 \
  --max-model-len 4096
```

---

### Verification
Wait for the **logs** to show the **server** is up:

```bash
sudo docker logs -f safety_guardrail
# Look for: "Uvicorn running on [http://0.0.0.0:8000](http://0.0.0.0:8000)"
```

---

### ⚠️ Cost Warning
* EC2 (g5.2xlarge): ~$1.21/hour. Stop the instance when not in use.

* Storage (EBS): ~$0.08/GB/month. Terminate the volume after the project is complete.