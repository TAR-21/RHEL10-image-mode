# RHEL 10 Bootc NVIDIA GPU 搭載カスタム ISO 作成手順書

本ドキュメントは、NVIDIA GPU (CUDA 13.1 / Driver 590+) をサポートした RHEL 10 Bootc イメージをビルドし、エッジデバイス用のインストール ISO を作成する手順をまとめたものです。

## 1. 事前準備

### リポジトリのクローンと移動
作業用ディレクトリに必要なソースコード一式を取得します。

```bash
git clone https://github.com/redhat-et/mlops-at-the-edge.git
cd mlops-at-the-edge/scenarios/scenario-01-device-edge/bootc-image
```

### レジストリへのログイン
イメージをプッシュするためのレジストリ（Quay.io や Docker Hub 等）および Red Hat 認定レジストリへログインします。

```bash
sudo podman login registry.redhat.io
sudo podman login quay.io
```

---

## 2. Bootc コンテナイメージのビルド

`Containerfile.bootc` を使用して、OS のルートファイルシステムとなるコンテナイメージを構築します。

### 環境変数の設定
`REGISTRY_USER` には、ご自身のレジストリユーザー名を指定してください。

```bash
export VERSION=v1.0.0
export REGISTRY_USER="<YOUR_REGISTRY_USER>"
export OCI_IMAGE_REPO="quay.io/${REGISTRY_USER}/mlops-bootc-rhel10-nvidia"
```

### ビルドの実行
このプロセスでは、RHEL 10 上で NVIDIA ドライバを DKMS (Dynamic Kernel Module Support) によりビルドします。

```bash
sudo podman build --platform=linux/amd64 \
  -f containerfiles/Containerfile.bootc \
  -t ${OCI_IMAGE_REPO}:${VERSION} .
```

### レジストリへのプッシュ
ビルドしたイメージをリモートレジストリに保存します。これは、後の ISO 生成プロセスで `bootc-image-builder` がイメージを参照するために必要です。

```bash
sudo podman push ${OCI_IMAGE_REPO}:${VERSION}
```

---

## 3. インストール用 ISO イメージの生成

`bootc-image-builder` を使用して、コンテナイメージを物理マシンにインストール可能な ISO 形式へ変換します。

### 出力ディレクトリの準備
```bash
mkdir -p ~/bootc-build/output
cd ~/bootc-build
```

### ISO 生成コマンドの実行
`--privileged` 権限と適切なセキュリティラベルを付与して実行します。

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

## 4. 成果物の確認とデプロイ

処理が完了すると、以下のパスに ISO ファイルが生成されます。

* **生成ファイル:** `~/bootc-build/output/iso/install.iso`

### イメージに含まれる主要な設定
* **デフォルトユーザー:** `admin` (パスワード: `admin123`)
* **GPU サポート:** NVIDIA Driver 590+ / CUDA 13.1
* **コンテナ環境:** Podman & podman-compose 導入済み
* **自動設定:** 起動時に NVIDIA UVM および CDI (Container Device Interface) を自動生成する systemd ユニットを構成済み

---

## 補足：トラブルシューティング
ビルドした ISO で起動後、GPU が正しく認識されているかは以下のコマンドで確認できます。
* `nvidia-smi` : ドライバと GPU のステータス確認
