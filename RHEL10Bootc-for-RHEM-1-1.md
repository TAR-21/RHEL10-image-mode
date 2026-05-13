# RHEL 10 Bootc NVIDIA GPU & Edge Manager 1.1 Custom ISO Creation Guide

This guide describes how to create an installable ISO for edge devices based on **RHEL 10 Bootc**, integrating an NVIDIA GPU environment, the Edge Manager agent (FlightCtl), and automatic rollback protection via Greenboot.

## 1. Prerequisites

Create a working directory and prepare the necessary configuration files.

```bash
# Move to the working directory
cd mlops-at-the-edge/scenarios/scenario-01-device-edge/bootc-image

# Place the configuration file for Early Binding
# ※ Ensure you have the config.yaml generated from Edge Manager
cp /path/to/flightctl-config.yaml ./config.yaml

# Log in to registries
sudo podman login registry.redhat.io
sudo podman login quay.io

```

## 2. Creating the Containerfile (`Containerfile.bootc`)

Create the file with the following content. `--nogpgcheck` is applied to external repositories to avoid GPG signature validation errors due to RHEL 10's strict security policies.

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:10.0

# =============================================================================
# SECTION 1: Red Hat Edge Manager (FlightCtl) & Greenboot Configuration
# =============================================================================
RUN dnf config-manager --set-enabled edge-manager-1.1-for-rhel-10-x86_64-rpms && \
    dnf -y install --nogpgcheck flightctl-agent greenboot greenboot-default-health-checks && \
    dnf -y clean all && \
    systemctl enable flightctl-agent.service greenboot-healthcheck.service && \
    systemctl mask bootc-fetch-apply-updates.timer

ADD config.yaml /etc/flightctl/

# =============================================================================
# SECTION 2: System Admin & SELinux Disabled
# =============================================================================
RUN useradd -m -G wheel admin && echo "admin:admin123" | chpasswd && systemctl enable sshd

RUN mkdir -p /usr/lib/bootc/bound-kargs.d && \
    echo "selinux=0" > /usr/lib/bootc/bound-kargs.d/00-disable-selinux.kargs && \
    sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

# =============================================================================
# SECTION 3: NVIDIA GPU Configuration (GPG Check Bypassed)
# =============================================================================
RUN dnf install -y --nogpgcheck https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm

RUN dnf swap -y --repofrompath=cs10-base,https://mirror.stream.centos.org/10-stream/BaseOS/x86_64/os/ --setopt=cs10-base.gpgcheck=0 openssl-fips-provider-so openssl-libs && \
    dnf install -y --repofrompath=cs10-app,https://mirror.stream.centos.org/10-stream/AppStream/x86_64/os/ --setopt=cs10-app.gpgcheck=0 gcc make dkms libX11 libXext libglvnd libglvnd-opengl && \
    dnf -y clean all

RUN dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel10/x86_64/cuda-rhel10.repo && \
    dnf config-manager --set-opt cuda-rhel10-x86_64.gpgcheck=0

RUN KERNEL_VERSION=$(rpm -q kernel --queryformat '%{VERSION}-%{RELEASE}.%{ARCH}') && \
    dnf install -y --repofrompath=cs10-base,https://mirror.stream.centos.org/10-stream/BaseOS/x86_64/os/ --setopt=cs10-base.gpgcheck=0 "kernel-devel-${KERNEL_VERSION}" && \
    dnf install -y --nogpgcheck "kmod-nvidia-latest-dkms-590*" "nvidia-driver-590*" "nvidia-driver-cuda-590*" nvidia-container-toolkit && \
    dkms autoinstall -k ${KERNEL_VERSION} && \
    systemctl mask dkms.service

RUN echo "blacklist nouveau" > /etc/modprobe.d/blacklist-nouveau.conf && \
    echo -e "nvidia\nnvidia-uvm\nnvidia-modeset" > /etc/modules-load.d/nvidia.conf && \
    systemctl enable nvidia-persistenced.service

# =============================================================================
# SECTION 4: Container Runtime (Podman)
# =============================================================================
RUN dnf install -y podman podman-compose && systemctl enable podman.service
COPY containerfiles/configs/containers.conf /etc/containers/containers.conf
RUN mkdir -p /etc/cdi /var/lib/containers

# =============================================================================
# SECTION 5: Greenboot Health Check Configuration (Rollback Protection)
# =============================================================================
RUN mkdir -p /usr/lib/greenboot/check/required.d && \
    echo '#!/bin/bash' > /usr/lib/greenboot/check/required.d/01_flightctl_agent_check.sh && \
    echo 'echo "Checking Flightctl Agent status..."' >> /usr/lib/greenboot/check/required.d/01_flightctl_agent_check.sh && \
    echo 'systemctl is-active flightctl-agent.service' >> /usr/lib/greenboot/check/required.d/01_flightctl_agent_check.sh && \
    chmod +x /usr/lib/greenboot/check/required.d/01_flightctl_agent_check.sh

LABEL com.redhat.edge-manager="1.1"
LABEL org.opencontainers.image.title="MLOps Edge RHEL 10 - FlightCtl & GPU"

```

## 3. Building and Pushing the Container Image

```bash
# Set version to v1.0.1
export VERSION=v1.0.1
export REGISTRY_USER="<YOUR_REGISTRY_USER>"
export OCI_IMAGE_REPO="quay.io/${REGISTRY_USER}/mlops-bootc-rhel10-nvidia"

# Build the image
sudo podman build --platform=linux/amd64 -f Containerfile.bootc -t ${OCI_IMAGE_REPO}:${VERSION} .

# Push to registry
sudo podman push ${OCI_IMAGE_REPO}:${VERSION}

```

## 4. Generating the Installable ISO Image

```bash
mkdir -p ~/bootc-build/output
cd ~/bootc-build

# Execute ISO generation
sudo podman run --rm -it --privileged \
  --security-opt label=type:unconfined_t \
  -v "$(pwd)/output":/output \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  quay.io/centos-bootc/bootc-image-builder:latest \
  build --type iso \
  ${OCI_IMAGE_REPO}:${VERSION}

```

## 5. Operational Note: Automatic Rollback via Greenboot

This image includes `Greenboot` to ensure system stability.

* **Mandatory Health Check**: The script `01_flightctl_agent_check.sh` defined in `/usr/lib/greenboot/check/required.d/` is treated as a mandatory condition for a successful boot.
* **Rollback Behavior**: If `flightctl-agent` fails to start, the system will attempt to reboot several times. If it remains unhealthy, the system will **automatically roll back to the previous stable version**.
* **Debugging**: If a rollback occurs, check the logs of the failed boot attempt with the following command:
`sudo journalctl -u greenboot-healthcheck -b -1`
