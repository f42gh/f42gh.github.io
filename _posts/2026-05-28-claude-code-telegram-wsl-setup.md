---
title: "WSLでClaude Code × Telegramを動かすまで：ハマりポイント全記録"
layout: post
date: 2026-05-28
tags: [tech, claude-code]
---

# WSLでClaude Code × Telegramを動かすまで：ハマりポイント全記録

> Claude Code × Telegram連携をWSL上で動かそうとして盛大にハマった記録。同じ環境の人の参考になれば。

## TL;DR

- Claude CodeのTelegramプラグインは **Bun** が必須（Nodeでは動かない）
- WSLではBunのPATHがClaude Codeの子プロセスに渡らず、**サイレントにfailする**
- `.mcp.json` の `"command": "bun"` をフルパスに書き換えれば解決

---

## 前提環境

- Windows 11 + WSL2（Ubuntu）
- Claude Code（最新版）
- Telegram ボット（BotFatherで作成済み）

## 0. そもそもの話：PowerShellで `.sh` は動かない

WSL上の `boot.sh` を起動しようとしたところ、最初にPowerShell側で引っかかった。

PowerShellが実行できるスクリプトは `.ps1` のみ。`.sh` はBash用なので、WSLを起こしてからBashで実行する必要がある。

```powershell
wsl
./boot.sh
```

## 1. `cannot execute: required file not found`

WSL内で `./boot.sh` を実行すると：

```
cannot execute: required file not found
```

一見ファイルが見つからないように見えるが、**実際に見つからないのはインタプリタの方**。原因はWindowsの改行コード（CRLF）。

shebang行 `#!/bin/bash` の末尾に `\r`（CR）が付いていて、OSが `/bin/bash\r` という存在しないプログラムを探してしまう。

```bash
# 確認
head -1 boot.sh | cat -A
# #!/bin/bash^M$  ← ^M が見えたらCRLFが原因

# 修正
sed -i 's/\r$//' boot.sh
chmod +x boot.sh
./boot.sh
```

## 2. Bunのインストール

Telegramプラグインの MCP サーバーは Bun で動く。WSLの最小構成には `unzip` すら入っていないので、先にインストール：

```bash
sudo apt install unzip
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc
bun --version
```

## 3. 既存のTelegramボットとの競合

同じボットトークンを2つのサービスで同時に使うことはできない。Telegramのポーリング接続は**1トークンにつき1つだけ**。同時接続すると `409 Conflict` エラーになる。

**対策：BotFatherで新しいボットを作成して使い分ける。**

## 4. プラグインのインストール

Claude Code内で：

```
/plugin install telegram@claude-plugins-official
/reload-plugins
/telegram:configure <BOT_TOKEN>
```

exitして、チャンネルフラグ付きで再起動：

```bash
claude --channels plugin:telegram@claude-plugins-official
```

## 5. プラグインがfailedになる（本記事のメイン）

`/mcp` でステータスを確認すると：

```
plugin:telegram:telegram · ✘ failed
```

エラーメッセージは `ENOENT`。

### 原因

プラグインの `.mcp.json` で `bun` コマンドが相対指定になっている：

```json
{
  "mcpServers": {
    "telegram": {
      "command": "bun",
      "args": ["run", "--cwd", "${CLAUDE_PLUGIN_ROOT}", "--shell=bun", "--silent", "start"]
    }
  }
}
```

Claude Codeが子プロセスを起動する際に、`.bashrc` や `.profile` で設定した `PATH` が引き継がれないため、`bun` が見つからずサイレントにfailする。**エラーメッセージは `ENOENT` のみで、Bunが原因だとは教えてくれない。**

### 確認方法：手動でプラグインを起動してみる

```bash
cd ~/.claude/plugins/cache/claude-plugins-official/telegram/0.0.6/
bun run start
```

これでTelegramボットが動いてペアリングコードが送られてくるなら、プラグイン自体は正常。Claude Codeからの自動起動だけが問題。

### 解決方法：フルパスに書き換える

```bash
sed -i 's|"command": "bun"|"command": "/home/<ユーザー名>/.bun/bin/bun"|' \
  ~/.claude/plugins/cache/claude-plugins-official/telegram/0.0.6/.mcp.json
```

書き換え後、Claude Codeを再起動：

```bash
claude --channels plugin:telegram@claude-plugins-official
```

`/mcp` でtelegramが緑（connected）になっていれば成功。

## 6. ペアリングと初期設定

1. Telegramで新しいボットにメッセージを送信 → 6文字のペアリングコードが返ってくる
2. Claude Code内で：
   ```
   /telegram:access pair <ペアリングコード>
   ```
3. セキュリティのため、allowlistモードに切り替え：
   ```
   /telegram:access policy allowlist
   ```

これで自分以外のユーザーからのメッセージはサイレントに無視される。

## 7. プラグイン更新時の注意

プラグインがアップデートされると `.mcp.json` が上書きされ、`"command": "bun"` に戻る可能性がある。再度 `sed` を実行すればいいが、エイリアスにしておくと楽：

```bash
# ~/.bashrc に追記
alias fix-telegram-plugin='sed -i '\''s|"command": "bun"|"command": "/home/<ユーザー名>/.bun/bin/bun"|'\'' ~/.claude/plugins/cache/claude-plugins-official/telegram/*/.mcp.json'
```

## まとめ：ハマりポイント一覧

| 症状 | 原因 | 対処 |
|---|---|---|
| PowerShellで `.sh` が動かない | PowerShellは `.ps1` のみ対応 | WSL内のBashで実行 |
| `cannot execute: required file not found` | shebang行にCRLFの `\r` が混入 | `sed -i 's/\r$//' script.sh` |
| `error: unzip is required` | WSL最小構成にunzipがない | `sudo apt install unzip` |
| `409 Conflict` | 同一トークンで複数サービスがポーリング | ボットを新規作成して使い分け |
| プラグインが `ENOENT` でfail | Claude CodeからBunのPATHが見えない | `.mcp.json` の `command` をフルパスに書き換え |

---

WSL × Claude Code × Telegramの組み合わせは、動いてしまえば快適だけど、セットアップの地雷が多い。特にBunのPATH問題はエラーメッセージが不親切でかなり辛い。この記事が同じ罠にハマった人の助けになれば幸い。
