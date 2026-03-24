# RHEL 10 Bootc NVIDIA GPU 搭載カスタム ISO 作成ガイド

この手順書は、**RHEL 10 Bootc** をベースに NVIDIA GPU ドライバ（CUDA 13.1 / Driver 590+）を組み込み、エッジデバイスへのインストールが可能な ISO イメージを作成する一連の流れを解説します。

## 1. 事前準備とリポジトリの取得

まず、必要なソースコードをクローンし、作業ディレクトリへ移動します。

```bash
# リポジトリのクローン
git clone https://github.com/redhat-et/mlops-at-the-edge.git

# プロジェクトディレクトリへ移動
cd mlops-at-the-edge/scenarios/scenario-01-device-edge/bootc-image
```

### レジストリへのログイン
Red Hat 公式レジストリおよび、ビルドしたイメージを保存するレジストリ（Quay.io 等）にログインします。

```bash
sudo podman login registry.redhat.io
sudo podman login quay.io
```

---

## 2. Containerfile の作成と確認

`scenarios/scenario-01-device-edge/bootc-image/containerfiles/Containerfile.bootc` として以下の内容を用意します。

> **注意:** このファイルには RHEL 10 用のビルドツールを補完するために CentOS Stream 10 のリポジトリを参照する高度な設定が含まれています。

```dockerfile
# Containerfile for RHEL 10 Bootc OS Image with NVIDIA GPU Support
# Includes: Podman + podman-compose, NVIDIA drivers 590+ with CUDA 13.1, and SSH access

FROM registry.redhat.io/rhel10/rhel-bootc:10.0

# =============================================================================
# SECTION 1: システム管理 & SSH 設定
# =============================================================================

# 管理ユーザー 'admin' の作成、sudo 権限付与、SSH サービスの有効化
RUN useradd -m -G wheel admin && \
    echo "admin:admin123" | chpasswd && \
    systemctl enable sshd

# =============================================================================
# SECTION 2: NVIDIA GPU 設定 (ビルド時のドライバインストール)
# =============================================================================

# RHEL 10 用の EPEL リポジトリをインストール
RUN dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm && \
    dnf -y clean all

# CentOS Stream 10 からビルドツールをインストール（RHEL 10 リポジトリ補完用）
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

# X11/OpenGL/Vulkan 依存関係のインストール
RUN dnf install -y \
    --repofrompath=centos10-baseos,https://mirror.stream.centos.org/10-stream/BaseOS/x86_64/os/ \
    --repofrompath=centos10-appstream,https://mirror.stream.centos.org/10-stream/AppStream/x86_64/os/ \
    --setopt=centos10-baseos.gpgcheck=0 \
    --setopt=centos10-appstream.gpgcheck=0 \
    libX11 libXext libglvnd libglvnd-egl libglvnd-gles libglvnd-glx libglvnd-opengl \
    libvdpau vulkan-loader ocl-icd opencl-filesystem && \
    dnf -y clean all

# NVIDIA CUDA リポジトリの追加
RUN dnf config-manager --add-repo \
    https://developer.download.nvidia.com/compute/cuda/repos/rhel10/x86_64/cuda-rhel10.repo

# OS イメージのカーネルバージョンに合致する kernel-devel をインストール
RUN KERNEL_VERSION=$(rpm -q kernel --queryformat '%{VERSION}-%{RELEASE}.%{ARCH}') && \
    dnf install -y \
    --repofrompath=centos10-baseos,https://mirror.stream.centos.org/10-stream/BaseOS/x86_64/os/ \
    --repofrompath=centos10-appstream,https://mirror.stream.centos.org/10-stream/AppStream/x86_64/os/ \
    --setopt=centos10-baseos.gpgcheck=0 \
    --setopt=centos10-appstream.gpgcheck=0 \
    "kernel-devel-${KERNEL_VERSION}" \
    "kernel-headers-${KERNEL_VERSION}" && \
    dnf -y clean all

# NVIDIA ドライバと Container Toolkit のインストール (590系に固定)
RUN dnf install -y \
    "kmod-nvidia-latest-dkms-590*" \
    "nvidia-driver-590*" \
    "nvidia-driver-cuda-590*" \
    nvidia-container-toolkit \
    --setopt=install_weak_deps=False \
    --allowerasing && \
    dnf -y clean all

# ビルド時に NVIDIA カーネルモジュールをコンパイル (DKMS)
RUN KERNEL_VERSION=$(rpm -q kernel --queryformat '%{VERSION}-%{RELEASE}.%{ARCH}') && \
    mkdir -p /var/lib/dkms && \
    dkms autoinstall -k ${KERNEL_VERSION}

RUN systemctl mask dkms.service

# Nouveau ドライバの無効化設定
RUN echo "blacklist nouveau" > /etc/modprobe.d/blacklist-nouveau.conf && \
    echo "options nouveau modeset=0" >> /etc/modprobe.d/blacklist-nouveau.conf

# モジュールの自動ロードと永続化設定
RUN echo -e "nvidia\nnvidia-uvm\nnvidia-modeset" > /etc/modules-load.d/nvidia.conf
RUN systemctl enable nvidia-persistenced.service

# NVIDIA UVM および CDI 初期化サービスの追加
COPY containerfiles/configs/nvidia-uvm-init.service /etc/systemd/system/nvidia-uvm-init.service
COPY containerfiles/configs/nvidia-cdi-generate.service /etc/systemd/system/nvidia-cdi-generate.service
RUN systemctl enable nvidia-uvm-init.service nvidia-cdi-generate.service

# =============================================================================
# SECTION 3: コンテナランタイム (Podman)
# =============================================================================

RUN dnf install -y podman podman-compose && \
    dnf -y clean all && \
    systemctl enable podman.service

COPY containerfiles/configs/containers.conf /etc/containers/containers.conf
RUN setsebool -P container_use_devices 1
RUN mkdir -p /etc/cdi /var/lib/containers

# =============================================================================
# メタデータラベル
# =============================================================================

LABEL org.opencontainers.image.title="MLOps Bootc RHEL 10 - GPU & SSH"
LABEL hardware.gpu.vendor="nvidia"
LABEL os.version="10"
```

---

## 3. コンテナイメージのビルドとプッシュ

環境変数を設定し、Podman でイメージをビルドしてレジストリへプッシュします。

```bash
# 環境変数の設定（レジストリユーザー名は適宜書き換えてください）
export VERSION=v1.0.0
export REGISTRY_USER="<YOUR_REGISTRY_USER>"
export OCI_IMAGE_REPO="quay.io/${REGISTRY_USER}/mlops-bootc-rhel10-nvidia"

# ビルドの実行（amd64 アーキテクチャ指定）
sudo podman build --platform=linux/amd64 \
  -f containerfiles/Containerfile.bootc \
  -t ${OCI_IMAGE_REPO}:${VERSION} .

# イメージのプッシュ
sudo podman push ${OCI_IMAGE_REPO}:${VERSION}
```

---

## 4. インストール用 ISO イメージの生成

`bootc-image-builder` を使用して、プッシュしたコンテナイメージをブート可能な ISO 形式に変換します。

```bash
# 出力ディレクトリの作成
mkdir -p ~/bootc-build/output
cd ~/bootc-build

# 最新イメージの取得（確認用）
sudo podman pull ${OCI_IMAGE_REPO}:${VERSION}

# ISO 生成の実行
sudo podman run --rm -it --privileged --pull=newer \
  --security-opt label=type:unconfined_t \
  -v "$(pwd)/output":/output \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  quay.io/centos-bootc/bootc-image-builder:latest \
  build --type iso \
  ${OCI_IMAGE_REPO}:${VERSION}
```

---

## 5. 成果物の確認

ビルドが完了すると、以下の場所に ISO イメージが生成されます。

* **成果物:** `~/bootc-build/output/iso/install.iso`

### システム構成の要約
* **ログイン:** ユーザー名 `admin` / パスワード `admin123`
* **GPU:** NVIDIA Driver 590+ / CUDA 13.1 導入済み
