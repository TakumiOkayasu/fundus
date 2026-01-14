# GitHub Actions Self-hosted Runner 構築ガイド (Proxmox)

## 概要

GitHub Actions の Self-hosted Runner を Proxmox VM 上に構築し、カーネルビルド等の重いCIジョブを高速化する。

**期待効果:**
- カーネルビルド: 50-60分 → 10-15分 (16コア時)
- ccacheがローカルに残るため、2回目以降はさらに高速化

---

## 必要要件

### ハードウェア要件 (VM)

| 項目 | 最小 | 推奨 | 備考 |
|------|------|------|------|
| vCPU | 4コア | 16コア | カーネルビルドは並列度が効く |
| RAM | 8GB | 16-32GB | カーネルビルドで8GB以上消費 |
| ディスク | 50GB | 100GB+ | ccache + ソース + ビルド成果物 |
| ネットワーク | 1Gbps | 1Gbps | GitHubへのアウトバウンド接続 |

### ソフトウェア要件

| 項目 | バージョン | 備考 |
|------|-----------|------|
| OS | Ubuntu LTS (最新版) | GUIなし (CLIのみ) |
| Docker | 最新安定版 | VyOSビルドコンテナ用 |
| Git | 2.30+ | - |

> **注意**: デスクトップ環境 (GNOME等) は不要。リソースの無駄なので入れない。
>
> **GUIなしのUbuntuを用意する方法:**
> - **Ubuntu Server ISO** を使う (最初からGUIなし)
> - **Ubuntu Cloud Image (LTS)** を使う (Proxmox向け、最小構成) - https://cloud-images.ubuntu.com/
> - 既存のDesktop環境を削除: `sudo apt remove --purge ubuntu-desktop gnome-shell && sudo apt autoremove`

### ネットワーク要件

**アウトバウンド接続 (必須):**
- `github.com` (443)
- `api.github.com` (443)
- `*.actions.githubusercontent.com` (443)
- `codeload.github.com` (443)
- `*.pkg.github.com` (443)
- `ghcr.io` (443)

**インバウンド接続:**
- 不要 (RunnerがGitHubにポーリング)

---

## 構築手順

### Step 1: Proxmox VM 作成

```bash
# Proxmox Web UI または CLI で VM 作成
# 推奨スペック: 16 vCPU, 32GB RAM, 100GB SSD

# Ubuntu 22.04 LTS をインストール
# ユーザー名: runner (任意)
```

### Step 2: 基本セットアップ

```bash
# SSH接続後

# システム更新
sudo apt update && sudo apt upgrade -y

# 必要パッケージのインストール
sudo apt install -y \
  curl \
  git \
  jq \
  build-essential \
  libssl-dev \
  libffi-dev \
  python3 \
  python3-pip \
  ca-certificates \
  gnupg \
  lsb-release
```

### Step 3: Docker インストール

```bash
# Docker GPG キー追加
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Docker リポジトリ追加
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Docker インストール
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# runner ユーザーを docker グループに追加
sudo usermod -aG docker $USER

# 確認 (再ログイン後)
docker run hello-world
```

### Step 4: GitHub Actions Runner インストール

#### 4.1 GitHub でトークン取得

1. リポジトリの Settings → Actions → Runners → New self-hosted runner
2. 表示されるトークンをコピー (有効期限: 1時間)

#### 4.2 Runner ダウンロード・設定

```bash
# 作業ディレクトリ作成
mkdir -p ~/actions-runner && cd ~/actions-runner

# 最新バージョンを取得 (2026年1月時点)
RUNNER_VERSION=$(curl -s https://api.github.com/repos/actions/runner/releases/latest | jq -r '.tag_name' | sed 's/v//')
curl -o actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz -L \
  https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz

# 展開
tar xzf actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz

# 設定 (トークンは GitHub の画面からコピー)
./config.sh --url https://github.com/<OWNER>/<REPO> --token <YOUR_TOKEN>

# 対話形式で以下を入力:
# - Runner group: [Enter] (デフォルト)
# - Runner name: proxmox-runner (任意)
# - Labels: self-hosted,linux,x64,proxmox (カンマ区切り)
# - Work folder: [Enter] (デフォルト: _work)
```

#### 4.3 サービスとして登録 (自動起動)

```bash
# サービスインストール
sudo ./svc.sh install

# サービス開始
sudo ./svc.sh start

# ステータス確認
sudo ./svc.sh status

# 自動起動有効化
sudo systemctl enable actions.runner.<OWNER>-<REPO>.<RUNNER_NAME>.service
```

### Step 5: ビルド環境の最適化

```bash
# ccache インストール
sudo apt install -y ccache

# ccache 設定 (大容量キャッシュ)
ccache --max-size=20G
ccache --set-config=cache_dir=/home/runner/.ccache

# VyOS ビルドコンテナを事前プル
docker pull vyos/vyos-build:current

# カーネルビルド用の追加パッケージ
sudo apt install -y \
  flex \
  bison \
  bc \
  libelf-dev \
  libncurses-dev \
  dwarves \
  kmod \
  cpio \
  rsync \
  debhelper \
  fakeroot \
  dpkg-dev
```

### Step 6: ワークフローの修正

リポジトリの `.github/workflows/build-vyos-custom-iso.yml` を修正:

```yaml
jobs:
  build-custom-iso:
    # 変更前
    # runs-on: ubuntu-latest

    # 変更後: Self-hosted Runner を使用
    runs-on: [self-hosted, linux, x64, proxmox]
    timeout-minutes: 120  # 短縮
```

---

## 運用・管理

### Runner の状態確認

```bash
# サービス状態
sudo ./svc.sh status

# ログ確認
journalctl -u actions.runner.<OWNER>-<REPO>.<RUNNER_NAME>.service -f
```

### Runner の更新

```bash
# GitHub が自動更新を推奨
# 手動更新が必要な場合:
cd ~/actions-runner
sudo ./svc.sh stop
# 新バージョンをダウンロード・展開
sudo ./svc.sh start
```

### トラブルシューティング

#### Runner がオフライン表示

```bash
# サービス再起動
sudo ./svc.sh stop
sudo ./svc.sh start

# ネットワーク確認
curl -I https://github.com
```

#### Docker 権限エラー

```bash
# runner ユーザーが docker グループに入っているか確認
groups runner

# 入っていない場合
sudo usermod -aG docker runner
# VM を再起動
```

#### ディスク容量不足

```bash
# Docker イメージ削除
docker system prune -af

# ccache クリア (必要に応じて)
ccache -C

# ビルド成果物削除
rm -rf ~/actions-runner/_work/*
```

---

## セキュリティ考慮事項

### 推奨設定

1. **専用VM**: Runner 専用の VM を使用 (他サービスと分離)
2. **ファイアウォール**: アウトバウンドのみ許可
3. **定期更新**: OS と Runner を定期的に更新
4. **シークレット管理**: GitHub Secrets を使用 (VM に直接保存しない)

### 非推奨

- パブリックリポジトリでの Self-hosted Runner 使用 (セキュリティリスク)
- root ユーザーでの Runner 実行
- 複数リポジトリでの Runner 共有 (分離推奨)

---

## 参考情報

### 公式ドキュメント

- [About self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners)
- [Adding self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/adding-self-hosted-runners)
- [Using self-hosted runners in a workflow](https://docs.github.com/en/actions/hosting-your-own-runners/using-self-hosted-runners-in-a-workflow)

### 期待されるビルド時間

| 環境 | コア数 | カーネルビルド | 全体 (ISO含む) |
|------|--------|---------------|---------------|
| GitHub Actions | 4 | ~50分 | ~70分 |
| Self-hosted (8コア) | 8 | ~25分 | ~40分 |
| Self-hosted (16コア) | 16 | ~12分 | ~25分 |
| Self-hosted (16コア+ccache) | 16 | ~5-8分 | ~18分 |

---

## チェックリスト

- [ ] Proxmox VM 作成 (16 vCPU, 32GB RAM, 100GB SSD)
- [ ] Ubuntu 22.04/24.04 LTS インストール
- [ ] Docker インストール・動作確認
- [ ] GitHub Actions Runner インストール・設定
- [ ] サービス登録・自動起動設定
- [ ] ccache 設定
- [ ] VyOS ビルドコンテナ事前プル
- [ ] ワークフロー修正 (`runs-on: [self-hosted, ...]`)
- [ ] テストビルド実行・時間計測
