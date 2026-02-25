# DigitalOcean MarketplaceでOpenClawを本番運用するための完全ガイド

**読了時間：15分**

**カテゴリー：チュートリアル**

---

## はじめに

この記事では、[DigitalOcean Marketplace](https://marketplace.digitalocean.com/apps/openclaw)のOpenClaw 1-Click Dropletを使用して、本番環境を構築する方法を解説します。Marketplace版は、OpenClawがプリインストールされており、専用のヘルパースクリプトが用意されているため、素早くセットアップできます。

ただし、**3つの重要な課題**があり、これを解決しないと本番運用できません。この記事では、その課題と解決策を詳しく説明します。

### この記事で学べること

- DigitalOcean Marketplace OpenClaw Dropletの作成
- AIモデルの選択と設定（Anthropic Sonnet推奨）
- Marketplace版特有の設定方法（`/opt/openclaw-cli.sh`など）
- 3つの重要な課題の解決策
- セキュリティ強化（Web UI認証）

### 前提条件

- DigitalOceanアカウント（[200ドルの無料クレジット付きサインアップ](https://m.do.co/c/signup)）
- Anthropic APIキー（[https://console.anthropic.com/](https://console.anthropic.com/)で取得）
- 基本的なLinuxコマンドの知識
- 所要時間：約40分

### コスト

- **Droplet: 月額12ドル**（2 vCPU、2GB RAM、60GB SSD）- 推奨スペック
- **Anthropic API: 使用量による**
  - Claude 3.5 Sonnet: $3/MTok (入力), $15/MTok (出力)
  - Claude 3 Opus: $15/MTok (入力), $75/MTok (出力)

**推奨：Sonnetを使用**（理由は後述）

---

## ステップ1: Marketplace Dropletの作成

### 1.1 OpenClaw Dropletの作成

1. [DigitalOcean Marketplace - OpenClaw](https://marketplace.digitalocean.com/apps/openclaw)にアクセス
2. **Create OpenClaw Droplet**をクリック
3. 以下を選択：
   - **Region:** 最も近いリージョン（日本の場合はSingapore）
   - **Droplet Size:** 
     - **推奨:** Basic → Regular → **$12/mo** (2 vCPU, 2GB RAM, 60GB SSD)
     - 最小: $6/mo (1 vCPU, 1GB RAM) - Swapが必要
   - **Authentication:** SSHキー（推奨）
   - **Hostname:** わかりやすい名前（例：`openclaw-production`）
4. **Create Droplet**をクリック
5. IPアドレスをメモ

### 1.2 SSHで接続

```bash
ssh root@YOUR_DROPLET_IP
```

初回接続時、ウェルカムメッセージが表示されます：

```
Welcome to OpenClaw on DigitalOcean!

To get started:
1. Configure your AI model: /opt/openclaw-cli.sh config
2. Check status: /opt/status-openclaw.sh
3. View logs: journalctl -u openclaw -f

Documentation: https://docs.openclaw.ai
Support: https://discord.com/invite/clawd
```

---

## ステップ2: AIモデルの設定（重要！）

### 2.1 なぜAnthropicのSonnetを推奨するのか

OpenClawは複数のAIモデルプロバイダーをサポートしていますが、**Anthropic Claude 3.5 Sonnetを強く推奨**します：

**Sonnet推奨の理由：**

1. **レートリミットの問題**
   - **Claude 3 Opus:** 高性能だが、レートリミットが厳しく、本番環境で頻繁にエラーが発生
   - **Claude 3.5 Sonnet:** 適度なレートリミットで、実用的な速度で動作
   
2. **コストパフォーマンス**
   - Sonnet: $3/MTok (入力), $15/MTok (出力)
   - Opus: $15/MTok (入力), $75/MTok (出力) - Sonnetの5倍
   
3. **性能**
   - Sonnet 3.5は十分に高性能で、ほとんどのタスクで満足できる結果
   - Opusの性能向上は、レートリミットとコストに見合わない

**結論：本番環境ではSonnetを使用することを強く推奨します。**

### 2.2 Anthropic APIキーの取得

1. [Anthropic Console](https://console.anthropic.com/)にアクセス
2. アカウント作成/ログイン
3. **API Keys**セクションで新しいキーを作成
4. キーをコピー（`sk-ant-api03-...`で始まる）

### 2.3 環境変数ファイルの編集

Marketplace版では、環境変数は`/opt/openclaw.env`で管理されます：

```bash
# 環境変数ファイルを編集
nano /opt/openclaw.env
```

以下の行を見つけて、APIキーを設定：

```bash
# For Anthropic Claude (recommended):
ANTHROPIC_API_KEY=sk-ant-api03-YOUR_KEY_HERE
```

**保存:** `Ctrl+O` → `Enter` → `Ctrl+X`

### 2.4 モデル設定の変更（Sonnetに設定）

```bash
# デフォルトモデルをSonnetに設定
/opt/openclaw-cli.sh config set agents.defaults.model.primary "anthropic/claude-sonnet-3-5"

# 設定を確認
/opt/openclaw-cli.sh config get agents.defaults.model
```

**期待される出力：**
```json
{
  "primary": "anthropic/claude-sonnet-3-5"
}
```

### 2.5 OpenClawを再起動

```bash
/opt/restart-openclaw.sh
```

または

```bash
systemctl restart openclaw
```

### 2.6 動作確認

```bash
# ステータス確認
/opt/status-openclaw.sh

# ログで動作確認
journalctl -u openclaw -f
```

エラーがなければ成功です！✅

---

## ステップ3: Marketplace版の理解

### 3.1 重要なファイルとスクリプト

Marketplace版には、標準インストールとは異なる専用のヘルパースクリプトがあります：

#### `/opt/openclaw-cli.sh`
すべてのOpenClawコマンドのラッパー。`openclaw`ユーザーとして実行します。

```bash
# 標準インストール
openclaw status

# Marketplace版
/opt/openclaw-cli.sh status
```

#### `/opt/openclaw.env`
環境変数設定ファイル（APIキー、Gateway設定など）

```bash
# APIキーや設定はここで管理
nano /opt/openclaw.env

# 変更後は再起動が必要
systemctl restart openclaw
```

#### その他のヘルパースクリプト

```bash
/opt/openclaw-tui.sh           # TUI（Text User Interface）起動
/opt/restart-openclaw.sh       # OpenClawを再起動
/opt/status-openclaw.sh        # ステータス確認
/opt/update-openclaw.sh        # OpenClawをアップデート
/opt/setup-openclaw-domain.sh  # カスタムドメイン設定
```

### 3.2 ユーザーとパーミッション

Marketplace版では、OpenClawは専用の`openclaw`ユーザーで実行されます：

```bash
# OpenClawユーザーに切り替え
su - openclaw

# 設定ファイルの場所
ls -la /home/openclaw/.openclaw/

# rootに戻る
exit
```

**重要：** すべてのOpenClawコマンドは`/opt/openclaw-cli.sh`経由で実行してください。

---

## ステップ4: 3つの重要な課題と解決策

### 課題1: サンドボックス環境がインターネットにアクセスできない 🚫

**症状：**
エージェントが外部API（天気情報、ニュース、検索など）にアクセスできない。

**原因：**
OpenClawのデフォルト設定では、セキュリティのためにエージェントはサンドボックス環境（Dockerコンテナ）内で実行されます。このサンドボックスは外部ネットワークへのアクセスが制限されています。

**解決策：**

```bash
# ホストで実行するように設定
/opt/openclaw-cli.sh config set tools.exec.host gateway

# インタラクティブな確認を無効化
/opt/openclaw-cli.sh config set tools.exec.ask off

# フルセキュリティモードを有効化
/opt/openclaw-cli.sh config set tools.exec.security full

# OpenClawを再起動
/opt/restart-openclaw.sh
```

**設定の説明：**

- `tools.exec.host gateway`: サンドボックスではなく、ゲートウェイホスト上で実行
- `tools.exec.ask off`: コマンド実行前の確認を無効化（自動化に必要）
- `tools.exec.security full`: フルセキュリティチェックを有効化

**確認：**

```bash
# 設定を確認
/opt/openclaw-cli.sh config get tools.exec

# 期待される出力：
{
  "host": "gateway",
  "ask": "off",
  "security": "full"
}
```

**テスト：**
Telegramなどのチャネルで「今日の天気は？」と聞いてみてください。外部APIにアクセスできれば成功です。

---

### 課題2: エージェントがブラウザにアクセスできない 🌐

**症状：**
エージェントがWebページをスクレイピングしたり、ブラウザ自動化を実行できない。

**原因：**
ChromeやChromiumがインストールされていない、または正しく設定されていない。

**解決策：**

#### 2.1 Google Chromeのインストール

```bash
# Google Chromeをダウンロード・インストール
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
dpkg -i google-chrome-stable_current_amd64.deb

# 依存関係を修正
apt --fix-broken install -y

# 確認
google-chrome --version
```

**期待される出力：**
```
Google Chrome 145.0.7632.116
```

#### 2.2 OpenClawのブラウザ設定

```bash
# ブラウザを有効化
/opt/openclaw-cli.sh config set browser.enabled true

# Chromeのパスを設定
/opt/openclaw-cli.sh config set browser.executablePath /usr/bin/google-chrome

# ヘッドレスモードを有効化
/opt/openclaw-cli.sh config set browser.headless true

# サンドボックスを無効化（必須）
/opt/openclaw-cli.sh config set browser.noSandbox true

# OpenClawを再起動
/opt/restart-openclaw.sh
```

#### 2.3 動作確認

```bash
# ブラウザステータス確認
/opt/openclaw-cli.sh browser status

# ブラウザ起動テスト
/opt/openclaw-cli.sh browser start

# 簡単なテスト
/opt/openclaw-cli.sh browser open https://example.com
/opt/openclaw-cli.sh browser screenshot
```

**期待される出力：**

```
profile: openclaw
enabled: true
running: true
cdpPort: 18800
cdpUrl: http://127.0.0.1:18800
browser: unknown
detectedBrowser: custom
detectedPath: /usr/bin/google-chrome
profileColor: #FF4500
```

#### 2.4 トラブルシューティング

もしエラーが出る場合：

```bash
# システムライブラリの不足確認
ldd /usr/bin/google-chrome | grep "not found"

# 不足ライブラリをインストール
apt install -y \
  libatk1.0-0 \
  libatk-bridge2.0-0 \
  libcups2 \
  libxkbcommon0 \
  libxcomposite1 \
  libxdamage1 \
  libxrandr2 \
  libgbm1 \
  libpango-1.0-0 \
  libasound2
```

---

### 課題3: Web UIに認証がない 🔓

**症状：**
DigitalOcean Marketplace版は、デフォルトでパブリックIPアドレス経由でWeb UIにアクセスできる状態になっており、認証が一切ありません。これは**深刻なセキュリティリスク**です。

**原因：**
Caddyリバースプロキシが以下のように設定されています：

```caddy
YOUR_IP {
    tls { ... }
    reverse_proxy localhost:18789
}
```

誰でも `https://YOUR_IP/` にアクセスできてしまいます。

**解決策：Basic認証の追加**

#### 3.1 パスワードハッシュの生成

```bash
# パスワードをハッシュ化
caddy hash-password

# プロンプトが表示されたらパスワードを入力
# 出力されたハッシュをコピー
```

**例：**
```
Enter password: ●●●●●●●●
Confirm password: ●●●●●●●●
$2a$14$JuViLfdKLPjLUabGupYi.p5uGV0O.FXt67nQ04bqdoBiIO0GSRFi
```

このハッシュをメモしてください。

#### 3.2 Caddyfileの編集

```bash
# Caddyfileを編集
nano /etc/caddy/Caddyfile
```

**変更前：**

```caddy
YOUR_IP {
    tls {
        issuer acme {
            dir https://acme-v02.api.letsencrypt.org/directory
            profile shortlived
        }
    }
    reverse_proxy localhost:18789
    header X-DO-MARKETPLACE "openclaw"
}
```

**変更後：**

```caddy
YOUR_IP {
    tls {
        issuer acme {
            dir https://acme-v02.api.letsencrypt.org/directory
            profile shortlived
        }
    }
    
    # Basic認証を追加
    basicauth {
        # ユーザー名: admin（任意の名前に変更可）
        admin $2a$14$JuViLfdKLPjLUabGupYi.p5uGV0O.FXt67nQ04bqdoBiIO0GSRFi
    }
    
    reverse_proxy localhost:18789
    header X-DO-MARKETPLACE "openclaw"
}
```

**重要：** `admin` の部分は任意のユーザー名に変更できます。ハッシュは先ほど生成したものを使用してください。

**保存:** `Ctrl+O` → `Enter` → `Ctrl+X`

#### 3.3 Caddyの再起動

```bash
# 設定をリロード
systemctl reload caddy

# ステータス確認
systemctl status caddy
```

**エラーがなければ成功です！**

#### 3.4 動作確認

1. ブラウザで `https://YOUR_IP/` にアクセス
2. ユーザー名とパスワードのプロンプトが表示されるはずです
3. 認証情報を入力してログイン

**成功！** 🎉 これでWeb UIは認証で保護されました。

---

## ステップ5: チャネルの接続

### 5.1 Telegramボットの作成と接続

#### Telegramボットの作成

1. Telegramで[@BotFather](https://t.me/botfather)を開く
2. `/newbot` コマンドを送信
3. ボット名を入力（例：`My OpenClaw Bot`）
4. ユーザー名を入力（例：`my_openclaw_bot`）
5. ボットトークンをコピー（`1234567890:ABCdefGHIjklMNOpqrsTUVwxyz...`）

#### OpenClawに設定

```bash
# 環境変数ファイルにトークンを追加
nano /opt/openclaw.env
```

以下の行を見つけて、トークンを設定：

```bash
# Telegram Bot Token
TELEGRAM_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz...
```

**保存:** `Ctrl+O` → `Enter` → `Ctrl+X`

```bash
# OpenClawを再起動
/opt/restart-openclaw.sh
```

#### ペアリング

```bash
# ペアリングコードを取得
/opt/openclaw-cli.sh pairing list telegram
```

**出力例：**
```
Pairing Code: ABCD1234
Expires: 2026-02-25 18:00:00
```

1. Telegramで作成したボットを開く
2. コード（例：`ABCD1234`）を送信
3. OpenClawで承認：

```bash
/opt/openclaw-cli.sh pairing approve telegram ABCD1234
```

**成功！** Telegramボットが接続されました。ボットにメッセージを送ってテストしてください。

### 5.2 その他のチャネル

- **WhatsApp:** `/opt/openclaw-cli.sh channels login whatsapp`
- **Discord:** `/opt/openclaw.env`に`DISCORD_BOT_TOKEN`を追加
- **Slack:** `/opt/openclaw.env`に`SLACK_BOT_TOKEN`と`SLACK_APP_TOKEN`を追加

詳細は[公式ドキュメント](https://docs.openclaw.ai/channels)を参照してください。

---

## ステップ6: 高度な設定（オプション）

### 6.1 カスタムドメインの設定

パブリックIPの代わりに、独自ドメイン（例：`bot.example.com`）を使用できます：

```bash
# セットアップスクリプトを実行
/opt/setup-openclaw-domain.sh
```

プロンプトに従って：

1. **ドメイン名を入力：** `bot.example.com`
2. **メールアドレスを入力：** `admin@example.com`（Let's Encrypt通知用）

スクリプトが自動的に：
- Caddyfileを更新
- Let's EncryptでSSL証明書を取得
- OpenClawを再起動

**重要：** ドメインのDNS設定で、AレコードをDropletのIPアドレスに向けてください。

### 6.2 セキュリティ強化：IP制限

特定のIPアドレスのみアクセス許可：

```bash
nano /etc/caddy/Caddyfile
```

```caddy
YOUR_IP_OR_DOMAIN {
    tls { ... }
    
    # IPアドレス制限
    @blocked not remote_ip YOUR_HOME_IP YOUR_OFFICE_IP
    respond @blocked "Access Denied" 403
    
    basicauth { ... }
    reverse_proxy localhost:18789
}
```

### 6.3 セキュリティ強化：Tailscale VPN（最も安全）

```bash
# Tailscaleインストール
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# OpenClaw設定
/opt/openclaw-cli.sh config set gateway.bind tailnet

# OpenClawを再起動
/opt/restart-openclaw.sh
```

これで、Tailscaleネットワークからのみアクセス可能になります。

---

## ステップ7: 監視とメンテナンス

### 7.1 ログの確認

```bash
# OpenClawログ
journalctl -u openclaw -f

# Caddyログ
journalctl -u caddy -f

# システムリソース
htop
```

### 7.2 定期的なメンテナンス

```bash
# システムアップデート
apt update && apt upgrade -y

# OpenClawアップデート
/opt/update-openclaw.sh

# 再起動（必要に応じて）
/opt/restart-openclaw.sh
```

### 7.3 バックアップ

重要なファイル：

```bash
# OpenClaw環境変数
/opt/openclaw.env

# OpenClaw設定（openclawユーザーとして）
su - openclaw
tar -czf ~/openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/
exit

# バックアップをダウンロード
scp root@YOUR_IP:/home/openclaw/openclaw-backup-*.tar.gz ./
```

---

## トラブルシューティング

### 問題1: "Permission denied" エラー

**症状：** `/opt/openclaw-cli.sh` でコマンド実行時にエラー

**解決策：**

```bash
# rootユーザーであることを確認
whoami  # 出力: root

# スクリプトに実行権限があることを確認
ls -l /opt/openclaw-cli.sh

# openclawユーザーが存在することを確認
id openclaw
```

### 問題2: APIレートリミットエラー

**症状：** `Rate limit exceeded` エラーが頻繁に発生

**解決策：**

```bash
# 現在のモデルを確認
/opt/openclaw-cli.sh config get agents.defaults.model

# OpusからSonnetに変更（推奨）
/opt/openclaw-cli.sh config set agents.defaults.model.primary "anthropic/claude-sonnet-3-5"

# OpenClawを再起動
/opt/restart-openclaw.sh
```

**Opusのレートリミットは厳しいため、本番環境ではSonnetを使用してください。**

### 問題3: メモリ不足（OOM）

**症状：** OpenClawが頻繁にクラッシュする

**解決策：**

```bash
# メモリ使用量を確認
free -h

# Swapを追加（1GB RAMの場合）
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# 確認
free -h
```

### 問題4: ブラウザが起動しない

**症状：** `Failed to start Chrome CDP`

**解決策：**

```bash
# Chromeを手動でテスト
google-chrome --headless --no-sandbox --disable-gpu --dump-dom https://example.com

# エラーが出る場合、システムライブラリをインストール
apt install -y \
  libatk1.0-0 libatk-bridge2.0-0 libcups2 \
  libxkbcommon0 libxcomposite1 libxdamage1 \
  libxrandr2 libgbm1 libpango-1.0-0 libasound2

# 設定を確認
/opt/openclaw-cli.sh browser status
```

---

## まとめ

これで、DigitalOcean Marketplace版OpenClawを本番環境として運用する準備が整いました！

### 実現したこと ✅

- ✅ OpenClaw Marketplace Dropletの作成
- ✅ Anthropic Claude Sonnetの設定（レートリミット対策）
- ✅ サンドボックス環境からのインターネットアクセス
- ✅ ブラウザ自動化の有効化
- ✅ Web UIへの認証追加によるセキュリティ強化
- ✅ メッセージングチャネルの接続

### 重要なポイント 🎯

1. **Sonnetを使用する**
   - Opusはレートリミットが厳しく、本番環境では実用的ではない
   - Sonnetは十分に高性能で、コストパフォーマンスも良い

2. **Marketplace版の特徴を理解する**
   - `/opt/openclaw-cli.sh`を使用
   - `/opt/openclaw.env`で環境変数を管理
   - `openclaw`ユーザーで実行される

3. **セキュリティを忘れずに**
   - Web UIにBasic認証を追加
   - 可能であればTailscale VPNを使用
   - 定期的にバックアップを取る

### 次のステップ 🚀

- [OpenClaw公式ドキュメント](https://docs.openclaw.ai/)を読む
- [Skills](https://docs.openclaw.ai/skills)を追加してエージェントを拡張
- [Heartbeat](https://docs.openclaw.ai/automation/heartbeat)で定期的なタスクを設定
- [メモリ管理](https://docs.openclaw.ai/memory)でエージェントの記憶を永続化

### サポート 💬

- **公式Discord:** https://discord.com/invite/clawd
- **GitHub:** https://github.com/openclaw/openclaw
- **ドキュメント:** https://docs.openclaw.ai/
- **Marketplace:** https://marketplace.digitalocean.com/apps/openclaw

---

**著者について**

本記事は、実際のDigitalOcean Marketplace OpenClaw環境の構築と運用経験に基づいて執筆されました。

**OpenClawについて**

OpenClawは、AIアシスタントをメッセージングプラットフォームに接続し、自動化、メモリ管理、スキル拡張を可能にするオープンソースフレームワークです。

---

**関連記事**

- [OpenClawの基本概念](/)
- [Telegramボットの作成方法](/)
- [ブラウザ自動化の高度なテクニック](/)
- [Anthropic Claude APIの使い方](/)
