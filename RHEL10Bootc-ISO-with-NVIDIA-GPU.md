# Guide: Creating a Custom RHEL 10 Bootc ISO with NVIDIA GPU Support

This guide outlines the workflow for building a **RHEL 10 Bootc** image equipped with NVIDIA GPU drivers (CUDA 13.1 / Driver 590+) and converting it into a bootable ISO for edge device deployment.

## 1. Prerequisites

### Clone the Repository
Prepare the working directory and fetch the necessary source files.

```bash
git clone https://github.com/redhat-et/mlops-at-the-edge.git
cd mlops-at-the-edge/scenarios/scenario-01-device-edge/bootc-image
```

### Registry Authentication
Log in to the Red Hat Registry and your target OCI registry (e.g., Quay.io or Docker Hub) to allow pushing and pulling images.

```bash
sudo podman login registry.redhat.io
sudo podman login quay.io
```

---

## 2. Build the Bootc Container Image

Use the `Containerfile.bootc` to build the containerized OS image. This process compiles the NVIDIA drivers using DKMS during the build phase.

### Set Environment Variables
Replace `<YOUR_REGISTRY_USER>` with your actual registry username.

```bash
export VERSION=v1.0.0
export REGISTRY_USER="<YOUR_REGISTRY_USER>"
export OCI_IMAGE_REPO="quay.io/${REGISTRY_USER}/mlops-bootc-rhel10-nvidia"
```

### Execute the Build
The build uses CentOS Stream 10 repositories as a temporary source for build tools not yet fully available in the RHEL 10 beta/early release repos.

```bash
sudo podman build --platform=linux/amd64 \
  -f containerfiles/Containerfile.bootc \
  -t ${OCI_IMAGE_REPO}:${VERSION} .
```

### Push to Registry
The image must be available in a remote registry for the `bootc-image-builder` tool to pull and process it into an ISO.

```bash
sudo podman push ${OCI_IMAGE_REPO}:${VERSION}
```

---

## 3. Generate the Bootable ISO

Use the `bootc-image-builder` utility to transform your container image into a physical installation medium (ISO).

### Prepare Output Directory
```bash
mkdir -p ~/bootc-build/output
cd ~/bootc-build
```

### Run the ISO Builder
Run the builder in privileged mode with the necessary security labels and volume mounts.

```bash
sudo podman run --rm -it --privileged --pull=newer \
  --security-opt label=type:unconfined_t \
  -v "$(pwd)/output":/output \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  quay.io/centos-bootc/bootc-image-builder:latest \
  build --type iso \
  ${OCI_IMAGE_REPO}:${VERSION}
```

---

## 4. Verification and Deployment

Once the process completes, the installer ISO will be located at:

* **Artifact Path:** `~/bootc-build/output/iso/install.iso`

### Included Features & Configuration
* **Default User:** `admin` (Password: `admin123`)
* **GPU Support:** NVIDIA Driver 590+ / CUDA 13.1 (DKMS-based)
* **Container Runtime:** Podman & Podman-compose pre-installed
* **Edge Optimization:** Automatic NVIDIA UVM and CDI (Container Device Interface) generation services are configured to run on boot.

---

## Appendix: Post-Installation Checks
After booting from the ISO and installing the OS, verify the GPU functionality with:
* `nvidia-smi` : To check driver and GPU hardware status.

---
