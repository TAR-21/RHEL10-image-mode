## RHEL 10 bootc + NVIDIA AI Server Build Guide

This guide describes how to build a "Golden Image" (ISO) that is immutable, secure, and ready for AI workloads out of the box.

### 1. Prerequisites (Build Host)

Prepare your workstation (RHEL 9/10 or Fedora) to build the container image and the ISO.

```bash
# Install Podman
sudo dnf install -y podman

# Login to Red Hat Registry (for base images)
sudo podman login registry.redhat.io

# Login to your Image Registry (e.g., Quay.io)
podman login quay.io

```

---

### 2. Configuration Files

Create a project directory `~/bootc-gpu-ai` and prepare the following two files.

#### ① `ollama.service`

This unit file generates the **CDI (Container Device Interface)** at runtime to ensure the GPU is detected by Podman before Ollama starts.

```ini
[Unit]
Description=Ollama GPU Container
After=network-online.target podman.service

[Service]
Restart=always
# Generate CDI spec on the fly based on the actual hardware (fixes build-time errors)
ExecStartPre=/usr/bin/nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
# Clean up previous containers
ExecStartPre=-/usr/bin/podman rm -f ollama

# Start Ollama with GPU access
ExecStart=/usr/bin/podman run --name ollama \
  --device nvidia.com/gpu=all \
  --security-opt label=disable \
  -v ollama_data:/root/.ollama \
  -p 11434:11434 \
  docker.io/ollama/ollama

[Install]
WantedBy=multi-user.target

```

#### ② `Containerfile`

The "Blueprint" for your OS. It includes security patches and excludes deprecated RHEL 10 packages.

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest

# 1. Apply Security Updates
# This addresses the High-severity vulnerabilities (setuptools, urllib3, etc.)
RUN dnf -y update --security && dnf clean all

# 2. Add NVIDIA Repository (RHEL 9 repo is compatible with RHEL 10)
RUN dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo

# 3. Install Required Packages
# Removed 'network-scripts' as it is fully deprecated in RHEL 10
RUN dnf install -y --nogpgcheck \
    nvidia-driver-cuda-libs \
    libnvidia-cfg \
    nvidia-container-toolkit \
    nfs-utils iputils && \
    dnf clean all

# 4. User and SSH Configuration (Day 01 Standard)
RUN useradd -m -G wheel admin && \
    echo 'admin:$6$rounds=4096$yourpasswordhash...' | chpasswd -e
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

# 5. Register Systemd Services
COPY ollama.service /etc/systemd/system/ollama.service
RUN systemctl enable ollama.service

```

---

### 3. Build, Push, and ISO Generation

Convert your blueprint into a bootable ISO.

```bash
# 1. Build the OS Container Image
podman build -t quay.io/<YOUR_ID>/rhel10-gpu-ai:v1 .

# 2. Push to Registry
podman push quay.io/<YOUR_ID>/rhel10-gpu-ai:v1

# 3. Generate the Installation ISO
mkdir -p output
sudo podman run --rm -it --privileged \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  -v $(pwd)/output:/output \
  registry.redhat.io/rhel10/bootc-image-builder:latest \
  build --type iso quay.io/<YOUR_ID>/rhel10-gpu-ai:v1

```

---

### 4. Post-Installation Verification

Once the installation from `output/bootiso/install.iso` is complete, log in as `admin` and run these checks:

1. **GPU Recognition:** `nvidia-smi` (Verify RTX 2000 Ada is listed).
2. **Container Status:** `podman ps` (Verify `ollama` is up and running).
3. **Bootc Status:** `bootc status` (Verify the system is managed by the immutable image).
