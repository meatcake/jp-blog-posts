# DigitalOceanでOpenClawを本番運用するための完全ガイド

**読了時間：10分**

**カテゴリー：チュートリアル**

---

## はじめに

OpenClawは、AI assistantをTelegram、WhatsApp、Discordなどのメッセージングプラットフォームに接続できる強力なフレームワークです。この記事では、DigitalOcean上でOpenClawを本番環境として構築し、よくある3つの課題とその解決策を紹介します。

### この記事で学べること

- DigitalOceanでのOpenClaw初期セットアップ
- サンドボックス環境のインターネットアクセス問題の解決
- ブラウザ自動化の設定
- Web UIへの認証追加によるセキュリティ強化

### 前提条件

- DigitalOceanアカウント（[200ドルの無料クレジット付きサインアップ](https://m.do.co/c/signup)）
- SSHキーペア（または、パスワード認証）
- 基本的なLinuxコマンドの知識
- 所要時間：約30分

### コスト

- **月額6ドル**（1 vCPU、1GB RAM、25GB SSD）
- または、**月額4ドル**（リザーブドプライシング）

---

## ステップ1: Dropletの作成

### 1.1 基本設定

1. [DigitalOcean](https://cloud.digitalocean.com/)にログイン
2. **Create → Droplets**をクリック
3. 以下を選択：
   - **Region:** 最も近いリージョン（日本の場合はSingaporeまたはSan Francisco）
   - **Image:** Ubuntu 24.04 LTS
   - **Size:** Basic → Regular → **$6/mo** (1 vCPU, 1GB RAM, 25GB SSD)
   - **Authentication:** SSHキー（推奨）またはパスワード
4. **Create Droplet**をクリック
5. IPアドレスをメモする

### 1.2 SSHで接続

```bash
ssh root@YOUR_DROPLET_IP
```

---

## ステップ2: システムの準備

### 2.1 システムアップデート

```bash
apt update && apt upgrade -y
```

### 2.2 Swapの追加（1GB RAMの場合は必須）

メモリが1GBしかないため、Swapを追加してOOM（Out of Memory）を防ぎます：

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# 確認
free -h
```

### 2.3 Node.js 22のインストール

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs

# 確認
node --version  # v22.x.x
npm --version   # 10.x.x
```

---

## ステップ3: OpenClawのインストール

### 3.1 OpenClawのインストール

```bash
curl -fsSL https://openclaw.ai/install.sh | bash

# 確認
openclaw --version
```

### 3.2 オンボーディング

```bash
openclaw onboard --install-daemon
```

ウィザードが以下を案内します：

- **モデル認証**（APIキーまたはOAuth）
- **チャネル設定**（Telegram、WhatsApp、Discordなど）
- **Gateway token**（自動生成）
- **Daemon installation**（systemd）

### 3.3 動作確認

```bash
# ステータス確認
openclaw status

# サービス確認
systemctl --user status openclaw-gateway.service

# ログ表示
journalctl --user -u openclaw-gateway.service -f
```

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
openclaw config set tools.exec.host gateway

# インタラクティブな確認を無効化
openclaw config set tools.exec.ask off

# フルセキュリティモードを有効化
openclaw config set tools.exec.security full

# Gatewayを再起動
openclaw gateway restart
```

**設定の説明：**

- `tools.exec.host gateway`: サンドボックスではなく、ゲートウェイホスト上で実行
- `tools.exec.ask off`: コマンド実行前の確認を無効化
- `tools.exec.security full`: フルセキュリティチェックを有効化

**確認：**

```bash
# 設定を確認
openclaw config get tools.exec

# エージェントに外部アクセスをテスト
# （Telegramなどのチャネルで）「今日の天気は？」と聞いてみる
```

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

#### 2.2 OpenClawのブラウザ設定

```bash
# ブラウザパスを設定
openclaw config set browser.enabled true
openclaw config set browser.executablePath /usr/bin/google-chrome
openclaw config set browser.headless true
openclaw config set browser.noSandbox true

# Gatewayを再起動
openclaw gateway restart
```

#### 2.3 動作確認

```bash
# ブラウザステータス確認
openclaw browser status

# ブラウザ起動テスト
openclaw browser start

# 簡単なテスト
openclaw browser open https://example.com
openclaw browser screenshot
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

もしエラーが出る場合、以下を確認：

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
DigitalOceanテンプレートは、デフォルトでパブリックIPアドレス経由でWeb UIにアクセスできる状態になっており、認証が一切ありません。これは**深刻なセキュリティリスク**です。

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
        admin <生成したハッシュをここに貼り付け>
    }
    
    reverse_proxy localhost:18789
    header X-DO-MARKETPLACE "openclaw"
}
```

**保存：** `Ctrl+O` → `Enter` → `Ctrl+X`

#### 3.3 Caddyの再起動

```bash
# 設定をリロード
systemctl reload caddy

# ステータス確認
systemctl status caddy
```

#### 3.4 動作確認

1. ブラウザで `https://YOUR_IP/` にアクセス
2. ユーザー名とパスワードのプロンプトが表示されるはずです
3. 認証情報を入力してログイン

**成功！** 🎉 これでWeb UIは認証で保護されました。

---

## ステップ5: チャネルの接続

### Telegramの接続

```bash
# ペアリングコードの確認
openclaw pairing list telegram

# Telegramボットでコードを送信し、承認
openclaw pairing approve telegram <CODE>
```

### WhatsAppの接続

```bash
# QRコードでログイン
openclaw channels login whatsapp

# QRコードをスキャン
```

その他のチャネル（Discord、Slackなど）については、[公式ドキュメント](https://docs.openclaw.ai/channels)を参照してください。

---

## ステップ6: 高度な設定（オプション）

### セキュリティ強化オプション

#### オプション1: IP制限

特定のIPアドレスのみアクセス許可：

```caddy
YOUR_IP {
    tls { ... }
    
    # IPアドレス制限
    @blocked not remote_ip YOUR_HOME_IP YOUR_OFFICE_IP
    respond @blocked "Access Denied" 403
    
    basicauth { ... }
    reverse_proxy localhost:18789
}
```

#### オプション2: Tailscale VPN

最も安全な方法：

```bash
# Tailscaleインストール
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# OpenClaw設定
openclaw config set gateway.bind tailnet
openclaw gateway restart
```

これで、Tailscaleネットワークからのみアクセス可能になります。

---

## ステップ7: 監視とメンテナンス

### ログの確認

```bash
# Gatewayログ
journalctl --user -u openclaw-gateway.service -f

# Caddyログ
journalctl -u caddy -f

# システムリソース
htop
```

### 定期的なメンテナンス

```bash
# システムアップデート
apt update && apt upgrade -y

# OpenClawアップデート
npm update -g openclaw

# 再起動（必要に応じて）
openclaw gateway restart
```

### バックアップ

重要なファイル：

```bash
# OpenClaw設定
~/.openclaw/openclaw.json
~/.openclaw/agents/main/agent.json

# ワークスペース
~/.openclaw/workspace/

# バックアップコマンド例
tar -czf openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/
```

---

## トラブルシューティング

### 問題: メモリ不足（OOM）

**症状：** Gatewayが頻繁にクラッシュする

**解決策：**

```bash
# Swapを確認
free -h

# メモリ使用量を確認
ps aux --sort=-%mem | head -10

# より大きなDropletにアップグレード
# または、APIベースのモデルを使用
```

### 問題: ブラウザが起動しない

**症状：** `Failed to start Chrome CDP`

**解決策：**

```bash
# Chromeを手動でテスト
google-chrome --headless --no-sandbox --disable-gpu --dump-dom https://example.com

# システムライブラリを確認
ldd /usr/bin/google-chrome | grep "not found"

# 設定を確認
openclaw browser status
```

### 問題: Caddyの設定エラー

**症状：** 認証が機能しない

**解決策：**

```bash
# Caddy設定をテスト
caddy validate --config /etc/caddy/Caddyfile

# ログを確認
journalctl -u caddy -f

# 手動でリロード
systemctl reload caddy
```

---

## まとめ

これで、DigitalOcean上でOpenClawを本番環境として運用する準備が整いました！

### 実現したこと

✅ OpenClawのインストールと設定  
✅ サンドボックス環境からのインターネットアクセス  
✅ ブラウザ自動化の有効化  
✅ Web UIへの認証追加によるセキュリティ強化  
✅ メッセージングチャネルの接続  

### 次のステップ

- [OpenClaw公式ドキュメント](https://docs.openclaw.ai/)を読む
- [Skills](https://docs.openclaw.ai/skills)を追加してエージェントを拡張
- [Heartbeat](https://docs.openclaw.ai/automation/heartbeat)で定期的なタスクを設定
- [メモリ管理](https://docs.openclaw.ai/memory)でエージェントの記憶を永続化

### コストの最適化

- **より安価なオプション：** [Hetzner](https://docs.openclaw.ai/install/hetzner)（月額4ユーロ）
- **無料オプション：** [Oracle Cloud](https://docs.openclaw.ai/platforms/oracle)（ARM、Always Free Tier）

### サポート

- **公式Discord：** https://discord.com/invite/clawd
- **GitHub：** https://github.com/openclaw/openclaw
- **ドキュメント：** https://docs.openclaw.ai/

---

**著者について**

本記事は、実際のDigitalOcean + OpenClaw本番環境構築経験に基づいて執筆されました。

**OpenClawについて**

OpenClawは、AIアシスタントをメッセージングプラットフォームに接続し、自動化、メモリ管理、スキル拡張を可能にするオープンソースフレームワークです。

---

**関連記事**

- [OpenClawの基本概念](/)
- [Telegramボットの作成方法](/)
- [ブラウザ自動化の高度なテクニック](/)
