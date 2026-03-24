# Guide: Creating a Custom RHEL 10 Bootc ISO with NVIDIA GPU Support

This guide provides the full configuration and commands to build a **RHEL 10 Bootc** image with NVIDIA GPU drivers (CUDA 13.1 / Driver 590+) and convert it into a bootable ISO for edge deployment.

## 1. Complete Containerfile Configuration

Below is the full content of `Containerfile.bootc`. This file uses a multi-repository approach to ensure all build dependencies are met for RHEL 10.

**File Path:** `scenarios/scenario-01-device-edge/bootc-image/containerfiles/Containerfile.bootc`

```dockerfile
# Containerfile for RHEL 10 Bootc OS Image with NVIDIA GPU Support
# Includes: Podman + podman-compose, NVIDIA drivers 590+ with CUDA 13.1, and SSH access

FROM registry.redhat.io/rhel10/rhel-bootc:10.0

# =============================================================================
# SECTION 1: System Admin & SSH Setup
# =============================================================================

# Create admin user and enable SSH
# Note: 'admin' user is created and added to the wheel group for sudo access.
RUN useradd -m -G wheel admin && \
    echo "admin:admin123" | chpasswd && \
    systemctl enable sshd

# To use an SSH Public Key (uncomment if you have id_rsa.pub in your build context)
# COPY id_rsa.pub /home/admin/.ssh/authorized_keys
# RUN chown -R admin:admin /home/admin/.ssh && chmod 700 /home/admin/.ssh && chmod 600 /home/admin/.ssh/authorized_keys

# =============================================================================
# SECTION 2: NVIDIA GPU Setup (Build-Time Installation)
# =============================================================================

# Install EPEL repository for RHEL 10
RUN dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm && \
    dnf -y clean all

# Install build tools from CentOS Stream 10 (RHEL 10 repos may be incomplete in early release)
RUN dnf swap -y \
    --repofrompath=centos10-baseos,https://mirror.stream.centos.org/10-stream/BaseOS/x86_64/os/ \
    --repofrompath=centos10-appstream,https://mirror.stream.centos.org/10-stream/AppStream/x86_64/os/ \
    --setopt=centos10-baseos.gpgcheck=0 \
    --setopt=centos10-appstream.gpgcheck=0 \
    openssl-fips-provider-so openssl-libs && \
    dnf install -y \
    --repofrompath=centos10-baseos,https://mirror.stream.centos.org/10-stream/BaseOS/x86_64/os/ \
    --repofrompath=centos10-appstream,https://mirror.stream.centos.org/10-stream/AppStream/x86_64/os/ \
    --setopt=centos10-baseos.gpgcheck=0 \
    --setopt=centos10-appstream.gpgcheck=0 \
    gcc \
    make \
    dkms && \
    dnf -y clean all

# X11/OpenGL/Vulkan dependencies
RUN dnf install -y \
    --repofrompath=centos10-baseos,https://mirror.stream.centos.org/10-stream/BaseOS/x86_64/os/ \
    --repofrompath=centos10-appstream,https://mirror.stream.centos.org/10-stream/AppStream/x86_64/os/ \
    --setopt=centos10-baseos.gpgcheck=0 \
    --setopt=centos10-appstream.gpgcheck=0 \
    libX11 libXext libglvnd libglvnd-egl libglvnd-gles libglvnd-glx libglvnd-opengl \
    libvdpau vulkan-loader ocl-icd opencl-filesystem && \
    dnf -y clean all

# Add NVIDIA CUDA repository
RUN dnf config-manager --add-repo \
    https://developer.download.nvidia.com/compute/cuda/repos/rhel10/x86_64/cuda-rhel10.repo

# Install kernel-devel matching the bootc image kernel
RUN KERNEL_VERSION=$(rpm -q kernel --queryformat '%{VERSION}-%{RELEASE}.%{ARCH}') && \
    dnf install -y \
    --repofrompath=centos10-baseos,https://mirror.stream.centos.org/10-stream/BaseOS/x86_64/os/ \
    --repofrompath=centos10-appstream,https://mirror.stream.centos.org/10-stream/AppStream/x86_64/os/ \
    --setopt=centos10-baseos.gpgcheck=0 \
    --setopt=centos10-appstream.gpgcheck=0 \
    "kernel-devel-${KERNEL_VERSION}" \
    "kernel-headers-${KERNEL_VERSION}" && \
    dnf -y clean all

# Install NVIDIA drivers and container toolkit (Pinned to 590.x)
RUN dnf install -y \
    "kmod-nvidia-latest-dkms-590*" \
    "nvidia-driver-590*" \
    "nvidia-driver-cuda-590*" \
    nvidia-container-toolkit \
    --setopt=install_weak_deps=False \
    --allowerasing && \
    dnf -y clean all

# Compile NVIDIA kernel modules at build time
RUN KERNEL_VERSION=$(rpm -q kernel --queryformat '%{VERSION}-%{RELEASE}.%{ARCH}') && \
    mkdir -p /var/lib/dkms && \
    dkms autoinstall -k ${KERNEL_VERSION}

RUN systemctl mask dkms.service

# Blacklist Nouveau
RUN echo "blacklist nouveau" > /etc/modprobe.d/blacklist-nouveau.conf && \
    echo "options nouveau modeset=0" >> /etc/modprobe.d/blacklist-nouveau.conf

# Load modules and persistency
RUN echo -e "nvidia\nnvidia-uvm\nnvidia-modeset" > /etc/modules-load.d/nvidia.conf
RUN systemctl enable nvidia-persistenced.service

# NVIDIA UVM & CDI services
COPY containerfiles/configs/nvidia-uvm-init.service /etc/systemd/system/nvidia-uvm-init.service
COPY containerfiles/configs/nvidia-cdi-generate.service /etc/systemd/system/nvidia-cdi-generate.service
RUN systemctl enable nvidia-uvm-init.service nvidia-cdi-generate.service

# =============================================================================
# SECTION 3: Container Runtime (Podman)
# =============================================================================

RUN dnf install -y podman podman-compose && \
    dnf -y clean all && \
    systemctl enable podman.service

COPY containerfiles/configs/containers.conf /etc/containers/containers.conf
RUN setsebool -P container_use_devices 1
RUN mkdir -p /etc/cdi /var/lib/containers

# =============================================================================
# Metadata Labels
# =============================================================================

LABEL org.opencontainers.image.title="MLOps Bootc RHEL 10 - GPU & SSH"
LABEL hardware.gpu.vendor="nvidia"
LABEL os.version="10"
```

---

## 2. Execution Steps

### Step 1: Preparation
Clone the repository and login to the required container registries.

```bash
git clone https://github.com/redhat-et/mlops-at-the-edge.git
cd mlops-at-the-edge/scenarios/scenario-01-device-edge/bootc-image

sudo podman login quay.io
sudo podman login registry.redhat.io
```

### Step 2: Build and Push the Bootc Image
Define your environment variables and build the OS container image.

```bash
export VERSION=v1.0.0
export REGISTRY_USER="<YOUR_REGISTRY_USER>"
export OCI_IMAGE_REPO="quay.io/${REGISTRY_USER}/mlops-bootc-rhel10-nvidia"

# Build the image
sudo podman build --platform=linux/amd64 \
  -f containerfiles/Containerfile.bootc \
  -t ${OCI_IMAGE_REPO}:${VERSION} .

# Push to your registry
sudo podman push ${OCI_IMAGE_REPO}:${VERSION}
```

### Step 3: Generate the Installation ISO
Create the bootable medium using the `bootc-image-builder`.

```bash
# Prepare output directory
mkdir -p ~/bootc-build/output
cd ~/bootc-build

# Pull the latest image locally (precautionary)
sudo podman pull ${OCI_IMAGE_REPO}:${VERSION}

# Run the ISO generation
sudo podman run --rm -it --privileged --pull=newer \
  --security-opt label=type:unconfined_t \
  -v "$(pwd)/output":/output \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  quay.io/centos-bootc/bootc-image-builder:latest \
  build --type iso \
  ${OCI_IMAGE_REPO}:${VERSION}
```

---

## 3. Results and Verification

The resulting file is located at `~/bootc-build/output/iso/install.iso`.

* **Credentials:** User `admin` / Password `admin123`.
* **Verification:** Once installed, run `nvidia-smi` to confirm the driver is active.

