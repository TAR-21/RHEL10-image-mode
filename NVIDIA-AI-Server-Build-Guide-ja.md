## RHEL 10 bootc + NVIDIA AI サーバー構築 全手順

### 1. 作業環境の準備 (ビルドホスト)

イメージを作成するためのツールをインストールし、レジストリにログインします。

```bash
# ツールインストール
sudo dnf install -y podman

# 各レジストリへのログイン
sudo podman login registry.redhat.io
podman login quay.io

```

---

### 2. 設定ファイルの作成

作業ディレクトリ `~/bootc-gpu-ai` を作成し、2つの重要ファイルを用意します。

#### ① `ollama.service`

実機のGPU環境に合わせてCDI（GPU定義）を起動時に自動生成し、Ollamaを起動するユニットファイルです。

```ini
[Unit]
Description=Ollama GPU Container
After=network-online.target podman.service

[Service]
Restart=always
# 起動直前に実機のGPUに合わせてCDIファイルを生成（ビルド時のエラー回避）
ExecStartPre=/usr/bin/nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
ExecStartPre=-/usr/bin/podman rm -f ollama

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

OSの設計図です。RHEL 10で廃止されたパッケージを除外し、セキュリティアップデートを組み込んでいます。

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest

# 1. セキュリティアップデートの適用
RUN dnf -y update --security && dnf clean all

# 2. NVIDIAリポジトリ追加
RUN dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo

# 3. 必要パッケージのインストール (RHEL 10互換構成)
RUN dnf install -y --nogpgcheck \
    nvidia-driver-cuda-libs \
    libnvidia-cfg \
    nvidia-container-toolkit \
    nfs-utils iputils && \
    dnf clean all

# 4. ユーザー・SSH設定
RUN useradd -m -G wheel admin && \
    echo 'admin:$6$rounds=4096$examplehash...' | chpasswd -e
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

# 5. サービスの登録
COPY ollama.service /etc/systemd/system/ollama.service
RUN systemctl enable ollama.service

```

---

### 3. ビルド・プッシュ・ISO化

コンテナイメージをビルドし、最終的にインストールメディア（ISO）へ変換します。

```bash
# イメージのビルド
podman build -t quay.io/tmorisu/rhel10-gpu-ai:v1 .

# レジストリへプッシュ
podman push quay.io/tmorisu/rhel10-gpu-ai:v1

# インストール用 ISO の生成
mkdir -p output
sudo podman run --rm -it --privileged \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  -v $(pwd)/output:/output \
  registry.redhat.io/rhel10/bootc-image-builder:latest \
  build --type iso quay.io/tmorisu/rhel10-gpu-ai:v1

```

---

### 4. インストール後の動作確認

生成された `output/bootiso/install.iso` で実機を起動し、以下のコマンドで正常性を確認します。

1. **GPU認識:** `nvidia-smi` (RTX 2000 Ada が表示されること)
2. **コンテナ稼働:** `podman ps` (Ollama が Up であること)
3. **セキュリティ:** `bootc status` (OSイメージが管理下にあること)
