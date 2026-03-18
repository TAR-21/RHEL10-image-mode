## RHEL 10 + Podman 5.x + NVIDIA GPU AI Server Setup Guide

This guide outlines the streamlined process for enabling **NVIDIA RTX 2000 Ada** (and similar Ada Generation GPUs) within Podman containers on RHEL 10 for AI inference using **Ollama**.

### 1. Prepare the NVIDIA Repository

While we use the RHEL 9 repository for compatibility, RHEL 10 requires specific flags during installation due to stricter security policies.

```bash
# Add the NVIDIA CUDA repository (Skip if already added)
sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo

```

### 2. Install Drivers, CUDA Libraries, and Management Tools

In RHEL 10, GPG key verification often fails with newer packages. We use the `--nogpgcheck` flag to ensure the three critical components for AI—Management (smi), Computation (libcuda), and Configuration (cfg)—are installed.

```bash
# Install nvidia-smi, libcuda, and config libraries
sudo dnf install -y --nogpgcheck \
  nvidia-driver-cuda-3:595.45.04-1.el9.x86_64 \
  nvidia-driver-cuda-libs-3:595.45.04-1.el9.x86_64 \
  libnvidia-cfg-3:595.45.04-1.el9.x86_64

```

* **nvidia-smi**: Essential for monitoring GPU load, temperature, and VRAM on the host.
* **libcuda.so.1**: The core library required by Ollama to perform GPU-accelerated math.

### 3. Configure Container Device Interface (CDI)

Podman 5.x utilizes **CDI** to pass GPU devices to containers. This replaces the older, more complex "nvidia-container-runtime" hooks.

```bash
# Generate the CDI configuration file
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

# Verify the detected device (Look for "nvidia.com/gpu=all")
nvidia-ctk cdi list

```

### 4. Deploy the AI Container (Ollama)

To allow the container to communicate with the GPU while maintaining RHEL's security posture, we use the CDI device flag and a specific SELinux security option.

```bash
podman run -d \
  --name ollama \
  --device nvidia.com/gpu=all \
  --security-opt label=disable \
  -v ollama_data:/root/.ollama \
  -p 11434:11434 \
  docker.io/ollama/ollama

```

* **`--device nvidia.com/gpu=all`**: Assigns the GPU via the CDI framework.
* **`--security-opt label=disable`**: Bypasses SELinux device restrictions for this specific container to allow GPU access.

### 5. Verification Flow

Confirm the setup is working correctly using these two methods:

#### ① Check GPU Recognition inside Container

```bash
podman logs ollama | grep -i "vram"
# Success if you see: "total_vram=8.0 GiB"

```

#### ② Monitor Real-time Load on Host

```bash
nvidia-smi
# Look for "/usr/bin/ollama" in the Processes list and check GPU-Util %

```
