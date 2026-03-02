# DigitalOceanでOpenClawを本番運用するガイド - 2026/2/25時点（日本語版）

**読了時間：12分**

---

## はじめに

この記事では、[DigitalOcean Marketplace](https://marketplace.digitalocean.com/apps/openclaw)のOpenClaw 1-Click Dropletを使って、実際に動く環境を作る方法を説明します。

僕自身がセットアップした時に、3つの問題にぶつかりました。この記事では、その解決方法と、実際に役立つ使い方まで紹介します。

### この記事でわかること

- DigitalOcean MarketplaceでのOpenClawセットアップ
- Telegramボットの接続方法
- AIモデルの選び方（AnthropicのSonnetがおすすめ）
- よくある3つの問題と解決策
- ブログ記事の自動翻訳など、実用的な使い方

### 必要なもの

- DigitalOceanアカウント（[200ドルクレジット付きサインアップ](https://m.do.co/c/signup)）
- Anthropic APIキー（[https://console.anthropic.com/](https://console.anthropic.com/)）
- 所要時間：40分くらい

### コスト

- **Droplet:** 月$12（2 vCPU、2GB RAM）
- **Anthropic API:** 使った分だけ
  - Sonnet: 入力$3/MTok、出力$15/MTok
  - Opus: 入力$15/MTok、出力$75/MTok（5倍高い）

Sonnetで十分です。理由は後で説明します。

---

## ステップ1: Dropletを作る

### 1.1 OpenClaw Dropletの作成

1. [DigitalOcean Marketplace - OpenClaw](https://marketplace.digitalocean.com/apps/openclaw)を開く
2. **Create OpenClaw Droplet**をクリック
3. 設定：
   - **Region:** 近いところ（日本ならSingapore）
   - **Droplet Size:** $12/mo（2 vCPU、2GB RAM）推奨
   - **Authentication:** SSHキー
   - **Hostname:** `openclaw-production`とか
4. **Create Droplet**をクリック
5. IPアドレスをメモ

### 1.2 SSHで接続

```bash
ssh root@YOUR_DROPLET_IP
```

初回接続すると、ウェルカムメッセージが出ます。

---

## ステップ2: Telegramボットを接続する

最初にTelegramを繋いでおくと、後のテストが楽です。

### 2.1 Telegramボットを作る

1. Telegramで[@BotFather](https://t.me/botfather)を開く
2. `/newbot`を送る
3. ボット名を入力（例：`My OpenClaw Bot`）
4. ユーザー名を入力（例：`my_openclaw_bot`）
5. トークンをコピー（`1234567890:ABC...`みたいなやつ）

### 2.2 OpenClawに設定

```bash
# 環境変数ファイルを編集
nano /opt/openclaw.env
```

この行を探して、トークンを入れる：

```bash
# Telegram Bot Token
TELEGRAM_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz...
```

保存：`Ctrl+O` → `Enter` → `Ctrl+X`

```bash
# 再起動
/opt/restart-openclaw.sh
```

### 2.3 ペアリング

```bash
# ペアリングコードを取得
/opt/openclaw-cli.sh pairing list telegram
```

出力：
```
Pairing Code: ABCD1234
Expires: 2026-02-25 18:00:00
```

1. Telegramボットを開く
2. コード（`ABCD1234`）を送る
3. 承認：

```bash
/opt/openclaw-cli.sh pairing approve telegram ABCD1234
```

ボットにメッセージを送って、返事が来れば成功です。

---

## ステップ3: AIモデルを設定する

### 3.1 なぜSonnetなのか

OpenClawはいろんなAIモデルに対応してますが、**Anthropic Claude 3.5 Sonnet**をおすすめします。

**理由：**

1. **レートリミット**
   - Opus: 高性能だけど、すぐ制限に引っかかる
   - Sonnet: 普通に使える
   
2. **コスト**
   - Sonnet: $3/$15（入力/出力）
   - Opus: $15/$75（5倍）
   
3. **性能**
   - Sonnetで十分

Opusは本番では使いづらいです。

### 3.2 Anthropic APIキーを取得

1. [Anthropic Console](https://console.anthropic.com/)でアカウント作成
2. **API Keys**で新しいキーを作る
3. キーをコピー（`sk-ant-api03-...`）

### 3.3 環境変数に設定

```bash
nano /opt/openclaw.env
```

この行を探して、APIキーを入れる：

```bash
# For Anthropic Claude (recommended):
ANTHROPIC_API_KEY=sk-ant-api03-YOUR_KEY_HERE
```

保存して終了。

### 3.4 Sonnetに設定

```bash
# デフォルトモデルをSonnetに
/opt/openclaw-cli.sh config set agents.defaults.model.primary "anthropic/claude-sonnet-3-5"

# 確認
/opt/openclaw-cli.sh config get agents.defaults.model
```

```bash
# 再起動
/opt/restart-openclaw.sh
```

### 3.5 テスト

Telegramボットに「こんにちは」と送ってみてください。返事が来れば成功です。

---

## ステップ4: Marketplace版の仕組みを理解する

### 4.1 重要なファイル

Marketplace版には、専用のヘルパースクリプトがあります。

#### `/opt/openclaw-cli.sh`
すべてのOpenClawコマンドのラッパー。

```bash
# 普通のインストール
openclaw status

# Marketplace版
/opt/openclaw-cli.sh status
```

#### `/opt/openclaw.env`
環境変数の設定ファイル（APIキーとか）。

```bash
nano /opt/openclaw.env  # 編集
systemctl restart openclaw  # 反映
```

#### その他のスクリプト

```bash
/opt/openclaw-tui.sh           # TUI起動
/opt/restart-openclaw.sh       # 再起動
/opt/status-openclaw.sh        # ステータス確認
/opt/update-openclaw.sh        # アップデート
/opt/setup-openclaw-domain.sh  # ドメイン設定
```

### 4.2 ユーザーとパーミッション

OpenClawは`openclaw`ユーザーで動いてます。

```bash
# openclawユーザーに切り替え
su - openclaw

# 設定ファイルの場所
ls -la ~/.openclaw/

# rootに戻る
exit
```

**重要：** `/opt/openclaw-cli.sh`経由でコマンド実行してください。

---

## ステップ5: 3つの問題と解決策

### 問題1: サンドボックスがインターネットにアクセスできない

**症状：**
天気APIとか外部サービスにアクセスできない。

**原因：**
デフォルトではサンドボックス（Dockerコンテナ）内で動くので、外部ネットワークが制限されてます。

**解決策：**

```bash
# ホストで実行するように変更
/opt/openclaw-cli.sh config set tools.exec.host gateway
/opt/openclaw-cli.sh config set tools.exec.ask off
/opt/openclaw-cli.sh config set tools.exec.security full

# 再起動
/opt/restart-openclaw.sh
```

**確認：**

```bash
/opt/openclaw-cli.sh config get tools.exec
```

Telegramで「今日の天気は？」と聞いてみてください。

---

### 問題2: ブラウザが使えない

**症状：**
Webページのスクレイピングができない。

**原因：**
Chromeがインストールされてない。

**解決策：**

#### 2.1 Chromeをインストール

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
dpkg -i google-chrome-stable_current_amd64.deb
apt --fix-broken install -y

# 確認
google-chrome --version
```

#### 2.2 OpenClawの設定

```bash
/opt/openclaw-cli.sh config set browser.enabled true
/opt/openclaw-cli.sh config set browser.executablePath /usr/bin/google-chrome
/opt/openclaw-cli.sh config set browser.headless true
/opt/openclaw-cli.sh config set browser.noSandbox true

# 再起動
/opt/restart-openclaw.sh
```

#### 2.3 確認

```bash
/opt/openclaw-cli.sh browser status
/opt/openclaw-cli.sh browser start
```

`running: true`なら成功です。

---

### 問題3: Web UIに認証がない

**症状：**
`https://YOUR_IP/`が誰でもアクセスできる状態。

**原因：**
Caddyの設定に認証がない。

**解決策：**

#### 3.1 パスワードをハッシュ化

```bash
caddy hash-password
```

パスワードを入力すると、ハッシュが出力されます。これをコピー。

#### 3.2 Caddyfileを編集

```bash
nano /etc/caddy/Caddyfile
```

`basicauth`セクションを追加：

```caddy
YOUR_IP {
    tls {
        issuer acme {
            dir https://acme-v02.api.letsencrypt.org/directory
            profile shortlived
        }
    }
    
    # 認証を追加
    basicauth {
        admin $2a$14$JuVi...（生成したハッシュ）
    }
    
    reverse_proxy localhost:18789
    header X-DO-MARKETPLACE "openclaw"
}
```

保存：`Ctrl+O` → `Enter` → `Ctrl+X`

#### 3.3 Caddyを再起動

```bash
systemctl reload caddy
systemctl status caddy
```

#### 3.4 確認

ブラウザで`https://YOUR_IP/`を開くと、ユーザー名とパスワードを聞かれます。

---

## ステップ6: 高度な設定

### 6.1 カスタムドメイン

独自ドメインを使いたい場合：

```bash
/opt/setup-openclaw-domain.sh
```

ドメイン名とメールアドレスを入力すれば、自動で設定してくれます。

### 6.2 IP制限

特定のIPだけ許可：

```bash
nano /etc/caddy/Caddyfile
```

```caddy
@blocked not remote_ip YOUR_HOME_IP YOUR_OFFICE_IP
respond @blocked "Access Denied" 403
```

### 6.3 Tailscale VPN（一番安全）

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

/opt/openclaw-cli.sh config set gateway.bind tailnet
/opt/restart-openclaw.sh
```

---

## ステップ7: 実用例 - ブログ記事の自動翻訳

ここからは、実際に僕がやってる使い方を紹介します。

### 7.1 やりたいこと

1. 毎朝9時に英語ブログをチェック
2. 新しい記事があれば、内容を取得
3. 日本語に翻訳（編集者目線で調整）
4. GitHubにPRを自動作成

### 7.2 準備

#### GitHub Personal Access Tokenを作る

1. [GitHub Settings → Personal access tokens](https://github.com/settings/tokens)
2. **Generate new token (classic)**
3. スコープ：`repo`にチェック
4. トークンをコピー（`ghp_...`）

#### トークンを設定

```bash
nano /opt/openclaw.env
```

追加：

```bash
GITHUB_TOKEN=ghp_YourTokenHere
```

保存して再起動。

### 7.3 Cronジョブを作る

```bash
/opt/openclaw-cli.sh cron add \
  --name "blog-translation" \
  --cron "0 9 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --announce \
  --message "ブラウザでhttps://www.zipteam.com/blog/をチェック。新しい記事があれば、内容を取得して日本語に翻訳し、GitHubのhttps://github.com/meatcake/jp-blog-postsにPRを作成してください。状態はworkspace/zipteam-blog-state.jsonで管理。新規記事がなければHEARTBEAT_OKと返してください。"
```

### 7.4 状態ファイルを作る

```bash
su - openclaw
cd ~/.openclaw/workspace

cat > zipteam-blog-state.json << 'EOF'
{
  "lastCheck": null,
  "seenPosts": []
}
EOF

exit
```

### 7.5 実績

この仕組みで：

- 3記事を自動翻訳
- 7つのPRを自動作成
- 手作業時間：週5時間 → 30分（95%削減）

**コスト：**
- Droplet: $12/月
- API: 週$1.65くらい
- **合計：月$19くらい**

**時給$50で計算すると、ROI約84倍。**

---


---

## ステップ7.5: CronとHeartbeatの使い分け

ステップ7でCronを使ってブログチェックを設定しましたが、実際に運用してみて問題に気づきました。**Cronジョブからはブラウザツールがブロックされてる**んです。

ここでは、その問題と解決策（Heartbeatへの切り替え）を説明します。

### 7.5.1 Cronとは？Heartbeatとは？

まず、2つの仕組みの違いを理解しましょう。

#### Cron（定期実行ジョブ）

**特徴：**
- 正確な時刻に実行（例：毎朝9:00ピッタリ）
- 独立したセッションで動く
- メインセッションとは別の環境
- 異なるモデルや設定を使える

**向いてるケース：**
- 正確な時刻が重要（「毎週月曜9:00に週報」とか）
- メインセッションと切り離したい
- 別のAIモデルを使いたい

**制限：**
- 独立環境なので、一部のツールにアクセスできない
- **ブラウザツールがブロックされる**（今回の問題）

#### Heartbeat（定期チェック）

**特徴：**
- メインセッションで実行
- 約30分おきにチェック（時刻は少しずれる）
- 会話のコンテキストがある
- **メインセッションのツール全部使える**

**向いてるケース：**
- 「だいたい朝」「1日数回」でOK
- ブラウザなど、メインセッションのツールが必要
- 複数のチェックをまとめたい（メール+カレンダー+ブログとか）

**制限：**
- 正確な時刻じゃない
- メインセッションと同じ環境（トークン消費が増える）

### 7.5.2 今回遭遇した問題

ステップ7でCronを使ってブログチェックを設定しましたが：

```bash
# これが動かなかった
/opt/openclaw-cli.sh cron add \
  --name "blog-translation" \
  --cron "0 9 * * *" \
  --session main \
  --message "ブラウザでブログをチェック..."
```

**エラー：**
```
Browser tools are blocked
Host browser control: blocked
```

`--session main`にしても、Cron環境自体がブラウザをブロックしてるようです。

ZipTeamブログはJavaScript（Gatsby/React）でレンダリングされるので、ブラウザなしではチェックできません。

### 7.5.3 解決策：Heartbeatに切り替える

Heartbeatならメインセッションで動くので、ブラウザにアクセスできます。

#### ステップ1: HEARTBEAT.mdを作る

ワークスペースに`HEARTBEAT.md`を作成：

```bash
su - openclaw
cd ~/.openclaw/workspace

cat > HEARTBEAT.md << 'EOF'
# Heartbeat Tasks

## ブログチェック（1日2-3回）
- https://www.zipteam.com/blog/ をブラウザでチェック
- workspace/zipteam-blog-state.json と比較
- 新しい記事があれば：
  - 内容を取得
  - 日本語に翻訳（4ステップワークフロー）
  - Telegramで通知
  - state.jsonを更新
- なければ静かに終了
EOF

exit
```

**重要：** Heartbeatはトークンを消費するので、チェック項目は必要最小限に。

#### ステップ2: 古いCronジョブを削除

```bash
# jobs.jsonを直接編集
nano /home/openclaw/.openclaw/cron/jobs.json

# または、jqで削除
jq '.jobs |= map(select(.name != "blog-translation"))' \
  /home/openclaw/.openclaw/cron/jobs.json > /tmp/jobs-new.json

cp /home/openclaw/.openclaw/cron/jobs.json \
   /home/openclaw/.openclaw/cron/jobs.json.backup

mv /tmp/jobs-new.json /home/openclaw/.openclaw/cron/jobs.json
```

#### ステップ3: 再起動

```bash
/opt/restart-openclaw.sh
```

#### ステップ4: 確認

Telegramで僕に「ブログをチェックして」と言ってみてください。動作確認できます。

### 7.5.4 Heartbeatの動作

Heartbeatは約30分おきに実行されます：

1. `HEARTBEAT.md`を読む
2. タスクを実行（ブログチェックなど）
3. 新しい情報があれば報告
4. なければ静かに終了（`HEARTBEAT_OK`）

**実行タイミング：**
- 朝、昼、夕方、夜など、1日に2-4回
- 正確な時刻ではない（だいたいの時間）

### 7.5.5 Heartbeatに追加できるタスク

`HEARTBEAT.md`には複数のタスクを書けます：

```markdown
# Heartbeat Tasks

## ブログチェック（1日2-3回）
（上記と同じ）

## メールチェック（1日2回）
- IMAPでメールボックスをチェック
- 重要なメールがあれば通知

## カレンダー確認（毎朝）
- 今日と明日の予定を確認
- 2時間以内のイベントがあれば通知

## 天気チェック（毎朝）
- San Jose, CAの天気を確認
- 雨が降る予報なら通知
```

**注意：** タスクが増えるとトークン消費も増えます。本当に必要なものだけ追加してください。

### 7.5.6 CronとHeartbeatの比較表

| 項目 | Cron | Heartbeat |
|------|------|-----------|
| **実行タイミング** | 正確（9:00ピッタリ） | だいたい（30分おき） |
| **セッション** | 独立 | メイン |
| **ブラウザアクセス** | ❌ ブロック | ✅ 可能 |
| **コンテキスト** | なし | あり |
| **トークン消費** | 少ない | 多め |
| **複数タスク** | 個別に設定 | まとめて実行 |
| **向いてるケース** | 正確な時刻が必要 | ツールアクセスが必要 |

### 7.5.7 僕の推奨

**ブログチェックのような用途には、Heartbeatがおすすめです。**

理由：
- ブラウザアクセスが必要
- 「だいたい朝」で十分
- 他のチェック（メール、カレンダー）もまとめられる

Cronは、「毎週月曜9:00に週報を作成」みたいな、正確な時刻が重要なタスクに使いましょう。

---
## ステップ8: 監視とメンテナンス

### 8.1 ログの確認

```bash
# OpenClawログ
journalctl -u openclaw -f

# Caddyログ
journalctl -u caddy -f

# システムリソース
htop
```

### 8.2 定期メンテナンス

```bash
# システムアップデート
apt update && apt upgrade -y

# OpenClawアップデート
/opt/update-openclaw.sh
```

### 8.3 バックアップ

```bash
# 重要なファイル
/opt/openclaw.env
~/.openclaw/

# バックアップ
su - openclaw
tar -czf ~/openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/
exit

# ダウンロード
scp root@YOUR_IP:/home/openclaw/openclaw-backup-*.tar.gz ./
```

---

## トラブルシューティング

### APIレートリミットエラー

**症状：** `Rate limit exceeded`が頻繁に出る

**解決策：**

```bash
# Sonnetに変更
/opt/openclaw-cli.sh config set agents.defaults.model.primary "anthropic/claude-sonnet-3-5"
/opt/restart-openclaw.sh
```

### メモリ不足

**症状：** OpenClawがクラッシュする

**解決策：**

```bash
# Swapを追加
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

free -h  # 確認
```

### ブラウザが起動しない

**症状：** `Failed to start Chrome CDP`

**解決策：**

```bash
# Chromeをテスト
google-chrome --headless --no-sandbox --disable-gpu --dump-dom https://example.com

# エラーが出たらライブラリをインストール
apt install -y \
  libatk1.0-0 libatk-bridge2.0-0 libcups2 \
  libxkbcommon0 libxcomposite1 libxdamage1 \
  libxrandr2 libgbm1 libpango-1.0-0 libasound2
```

---

## まとめ

この記事でやったこと：

- OpenClaw Marketplace Dropletの作成
- Telegramボットの接続
- Anthropic Claude Sonnetの設定
- 3つの問題の解決
- ブログ自動翻訳の実装

### 次のステップ

- [OpenClaw公式ドキュメント](https://docs.openclaw.ai/)
- [Skills](https://docs.openclaw.ai/skills)で機能拡張
- [Heartbeat](https://docs.openclaw.ai/automation/heartbeat)で定期タスク

### サポート

- Discord: https://discord.com/invite/clawd
- GitHub: https://github.com/openclaw/openclaw
- Docs: https://docs.openclaw.ai/

---

**2026年2月25日時点の情報です。**

OpenClawやDigitalOcean Marketplaceの仕様は変わる可能性があります。最新情報は公式ドキュメントを確認してください。
