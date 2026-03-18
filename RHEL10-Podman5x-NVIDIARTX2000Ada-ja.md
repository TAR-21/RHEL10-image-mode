## RHEL 10 + Podman 5.x + NVIDIA GPU AIサーバー構築ガイド

このガイドは、**RTX 2000 Ada** 等の最新GPUを、RHEL 10 上の Podman コンテナから 100% 活用するための最短ルートをまとめたものです。

### 1. NVIDIA リポジトリの準備

RHEL 9 用のリポジトリを流用しますが、RHEL 10 の厳格なセキュリティに対応させる必要があります。

```bash
# リポジトリの追加 (実行済みであればスキップ可)
sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo

```

### 2. ドライバ・計算ライブラリ・管理ツールのインストール

RHEL 10 では署名（GPG）エラーが発生しやすいため、`--nogpgcheck` を使用して、AI推論に必要な 3 つの要素を確実にインストールします。

```bash
# nvidia-smi(ツール)、libcuda(計算)、cfg(構成)をまとめてインストール
sudo dnf install -y --nogpgcheck \
  nvidia-driver-cuda-3:595.45.04-1.el9.x86_64 \
  nvidia-driver-cuda-libs-3:595.45.04-1.el9.x86_64 \
  libnvidia-cfg-3:595.45.04-1.el9.x86_64

```

* **nvidia-smi**: ホスト側で GPU の負荷や温度を監視するために必須。
* **libcuda.so.1**: Ollama 等の AI が GPU 計算を行うための心臓部。

### 3. Container Device Interface (CDI) の設定

Podman 5.x から採用された新しいデバイス割り当て方式「CDI」を設定します。これにより、従来の複雑なフックなしで GPU をコンテナに渡せます。

```bash
# CDI 設定ファイルの生成
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

# 認識されているデバイス名の確認 (nvidia.com/gpu=all が出ればOK)
nvidia-ctk cdi list

```

### 4. AIコンテナ (Ollama) の起動

SELinux の制限を考慮しつつ、生成した CDI デバイスを指定してコンテナを立ち上げます。

```bash
podman run -d \
  --name ollama \
  --device nvidia.com/gpu=all \
  --security-opt label=disable \
  -v ollama_data:/root/.ollama \
  -p 11434:11434 \
  docker.io/ollama/ollama

```

* **`--device nvidia.com/gpu=all`**: CDI を利用した GPU 割り当て。
* **`--security-opt label=disable`**: SELinux によるデバイス拒否を回避し、GPU との通信を許可。

### 5. 正常動作の確認フロー

セットアップが成功したか、以下の 2 段階で確認します。

#### ① コンテナ内での GPU 認識確認

```bash
podman logs ollama | grep -i "vram"
# 「total_vram="8.0 GiB"」のように表示されれば、AIがGPUを掴んでいます。

```

#### ② ホスト側での負荷確認

```bash
nvidia-smi
# Processes 欄に「/usr/bin/ollama」が表示され、GPU-Util が上昇していれば完璧です。

```

---

### トラブルシューティング・メモ

* **nvidia-smi が見つからない場合**: `dnf provides "*/nvidia-smi"` でパッケージ名を確認し、再インストールしてください。
* **ライブラリが見つからない場合**: `/usr/lib64/libcuda.so.1` が存在するか確認してください。
* **再起動後の注意**: ドライバー更新後は `sudo nvidia-ctk cdi generate` を再実行して設定を更新することをお勧めします。
