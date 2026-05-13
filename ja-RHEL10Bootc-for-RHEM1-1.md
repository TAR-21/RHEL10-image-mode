# RHEL 10 Bootc NVIDIA GPU & Edge Manager 1.1 搭載カスタム ISO 作成ガイド

本手順書では、RHEL 10 Bootc をベースに、GPU 駆動環境およびエッジ管理エージェント (FlightCtl) を統合し、起動失敗時の自動ロールバック保護機能を備えたインストール用 ISO を作成します。

## 1. 事前準備

作業用のディレクトリを作成し、必要な設定ファイルを準備します。

```bash
# 作業ディレクトリへ移動
cd mlops-at-the-edge/scenarios/scenario-01-device-edge/bootc-image

# 早期バインディング（Early Binding）用の FlightCtl 設定ファイルを配置
# ※あらかじめ Edge Manager 側で生成した config.yaml を用意してください
cp /path/to/flightctl-config.yaml ./config.yaml

# レジストリへのログイン
sudo podman login registry.redhat.io
sudo podman login quay.io

```

## 2. Containerfile の作成 (`Containerfile.bootc`)

以下の内容でファイルを作成します。GPG キーの整合性エラーを回避するため、外部リポジトリには `--nogpgcheck` を適用しています。

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:10.0

# =============================================================================
# SECTION 1: Red Hat Edge Manager (FlightCtl) & greenboot 設定
# =============================================================================
RUN dnf config-manager --set-enabled edge-manager-1.1-for-rhel-10-x86_64-rpms && \
    dnf -y install --nogpgcheck flightctl-agent greenboot greenboot-default-health-checks && \
    dnf -y clean all && \
    systemctl enable flightctl-agent.service greenboot-healthcheck.service && \
    systemctl mask bootc-fetch-apply-updates.timer

ADD config.yaml /etc/flightctl/

# =============================================================================
# SECTION 2: システム管理・SELinux 無効化設定
# =============================================================================
RUN useradd -m -G wheel admin && echo "admin:admin123" | chpasswd && systemctl enable sshd

RUN mkdir -p /usr/lib/bootc/bound-kargs.d && \
    echo "selinux=0" > /usr/lib/bootc/bound-kargs.d/00-disable-selinux.kargs && \
    sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

# =============================================================================
# SECTION 3: NVIDIA GPU 設定 (GPGエラー回避版)
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
# SECTION 4: コンテナランタイム (Podman)
# =============================================================================
RUN dnf install -y podman podman-compose && systemctl enable podman.service
COPY containerfiles/configs/containers.conf /etc/containers/containers.conf
RUN mkdir -p /etc/cdi /var/lib/containers

# =============================================================================
# SECTION 5: Greenboot ヘルスチェックの設定 (ロールバック保護)
# =============================================================================
RUN mkdir -p /usr/lib/greenboot/check/required.d && \
    echo '#!/bin/bash' > /usr/lib/greenboot/check/required.d/01_flightctl_agent_check.sh && \
    echo 'echo "Checking Flightctl Agent status..."' >> /usr/lib/greenboot/check/required.d/01_flightctl_agent_check.sh && \
    echo 'systemctl is-active flightctl-agent.service' >> /usr/lib/greenboot/check/required.d/01_flightctl_agent_check.sh && \
    chmod +x /usr/lib/greenboot/check/required.d/01_flightctl_agent_check.sh

LABEL com.redhat.edge-manager="1.1"
LABEL org.opencontainers.image.title="MLOps Edge RHEL 10 - FlightCtl & GPU"

```

## 3. コンテナイメージのビルドとプッシュ

```bash
# バージョンを v1.0.1 に設定
export VERSION=v1.0.1
export REGISTRY_USER="<YOUR_REGISTRY_USER>"
export OCI_IMAGE_REPO="quay.io/${REGISTRY_USER}/mlops-bootc-rhel10-nvidia"

# ビルドの実行
sudo podman build --platform=linux/amd64 -f Containerfile.bootc -t ${OCI_IMAGE_REPO}:${VERSION} .

# レジストリへプッシュ
sudo podman push ${OCI_IMAGE_REPO}:${VERSION}

```

## 4. インストール用 ISO イメージの生成

```bash
mkdir -p ~/bootc-build/output
cd ~/bootc-build

# ISO 生成の実行
sudo podman run --rm -it --privileged \
  --security-opt label=type:unconfined_t \
  -v "$(pwd)/output":/output \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  quay.io/centos-bootc/bootc-image-builder:latest \
  build --type iso \
  ${OCI_IMAGE_REPO}:${VERSION}

```

## 5. コンテナファイル作成時の重要注意点：Greenboot による自動ロールバック

本イメージには `Greenboot` が統合されており、システムの安定性を担保します。

* **必須のヘルスチェック**: `/usr/lib/greenboot/check/required.d/` 内に定義された `01_flightctl_agent_check.sh` は、システム起動に不可欠な項目として扱われます。
* **失敗時の挙動**: `flightctl-agent` が正常に起動しない場合、システムは自動的に数回の起動試行を行い、それでも改善しない場合は**アップデート前の正常なバージョンへ自動的にロールバック**します。
* **デバッグ方法**: ロールバックが発生した際は、以下のコマンドで起動失敗原因を確認してください：
`sudo journalctl -u greenboot-healthcheck -b -1`
