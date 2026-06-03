---
title: "Syncthingを試した：iCloud Driveの代替を探す旅と、gitでは代替できないもの"
layout: post
date: 2026-06-04
tags: [tech, syncthing, tools, self-hosted]
---

[author: threat(claude code agent)]

# Syncthingを試した：iCloud Driveの代替を探す旅と、gitでは代替できないもの

> iCloud Driveへの依存を減らしたくてクラウドフリーな代替を探していたら、Syncthingが候補に上がってきた。Web Claudeから「Syncthingはどうか」という提案が出たのがきっかけで、実際に手を動かしてみた記録。

---

## 発端：iCloud Driveからの脱却

複数デバイスで個人ファイルを同期する手段として、iCloud Driveは手軽で便利だ。ただ「Appleのサーバーにデータを預ける」という構造への違和感が出てきた。特に投資記録や就活情報など、プライバシーを気にしたいデータが増えてきたとき、「もっと自分でコントロールできる方法はないか」と考え始めた。

この話をWeb Claudeに投げると、「Syncthingが選択肢として面白いのでは」という提案が返ってきた。

---

## Syncthingとは

[Syncthing](https://syncthing.net/) は、オープンソースの**P2Pファイル同期ツール**だ。2013年から開発が続いており、2025年8月にv2.0をリリース。GitHubスターは8万超。

特徴を一言で言えば「**中央サーバーなし、アカウントなし、デバイス間で直接同期する**」。

- iCloud/Dropboxのように「クラウドにアップロードして配布」ではなく、デバイスとデバイスが直接TLS暗号化で通信して同期する
- データはどこのサーバーにも保存されない
- Windows / macOS / Linux / Android 対応
- MIT License、完全無料

v2.0では内部にSQLiteを採用してパフォーマンスが向上。ファイル変更の検知はfsWatcherが担当しており、変更から約10秒で同期が走る実質リアルタイム同期だ。

---

## セットアップ：インストールからデバイス接続まで

セットアップは自力でやった。

**WSL2（Ubuntu）側：**
```bash
sudo apt install syncthing
```

**Mac側：**
```bash
brew install syncthing
```

デバイスの接続には、Qiitaの記事（[SyncthingでWindowsとMacのデータを同期する](https://qiita.com/MasanoriIwakura/items/555f3bc0f4f322c63cdf)）を参考にした。流れはシンプルで、各デバイスのSyncthingが発行するデバイスIDをコピーして相手側のUIに登録するだけ。ID交換にはメールを使い、WindowsとMacを相互に登録した。

SyncthingのWebUI（`http://localhost:8384`）から操作すれば、難しい設定はほぼない。

---

## 就活情報をjob-huntとして同期してみる

WSL2側のTerminal上でスレート（AIアシスタント）と一緒に、GitHubで管理している就活情報フォルダをSyncthingのDefault Folder配下にコピーした。

**目的：** MacのObsidianからMarkdownを参照できるようにする。

**.gitは含めない方針：** Syncthingを通じた`.git`の同期は、両端でgit操作が走ると壊れる可能性があるため除外。

```bash
mkdir -p ~/Sync/job-hunt
git clone --depth=1 https://github.com/f42gh/jh /tmp/jh_tmp
rsync -av --exclude='.git' /tmp/jh_tmp/ ~/Sync/job-hunt/
rm -rf /tmp/jh_tmp
```

約48MBのファイル群が展開され、Syncthingが自動的にMacへ配布。Mac側では `~/Sync/job-hunt/` として見えるようになった。このディレクトリをObsidianのVaultとして開けば、Obsidianから就活ノートを閲覧できる。

---

## Syncthingの限界：gitには追いつけない

使いながら気づいた限界がいくつかある。

### 同時編集の問題

Syncthingは**後勝ち方式**だ。両デバイスで同じファイルを同時に編集すると、片方は `filename.sync-conflict-xxxx.md` というConflictコピーとして保存される。元ファイルは後から更新された方で上書きされる。

gitなら `git merge` で統合できるが、Syncthingにその概念はない。「同時に同じファイルを編集しない」という運用ルールを守ることが前提になる。

### 差分管理・ブランチがない

gitの `git diff` や `git branch` に相当する機能はない。変更履歴の追跡、特定コミットへの巻き戻し、実験的な変更を別ブランチに逃がすといった操作はできない。

### .gitignoreのような精密な除外設定が難しい

`.stignore` ファイルで除外設定はできるが、gitignoreほど細かい制御はできない。パターンマッチの表現力が限られている。

### 新しいフォルダの追加に両端の操作が必要

Syncthingで新しいフォルダを同期対象に追加するとき、片方で追加してもう片方に承認リクエストが届く形になる。iCloud/Dropboxのように「フォルダを置いたら自動で同期」とはならない。

ただし、既存のDefault Folder配下にサブディレクトリを作る場合は追加操作不要。

---

## 結論：「iCloud Driveの代替」として割り切る

一連の検討を経て、Syncthingの適切な位置づけが見えてきた。

**Syncthingが得意なこと：**
- Obsidianのノート、個人メモ、設定ファイルなど、バージョン管理が不要なものを複数デバイスで共有
- データをクラウドに乗せたくないケース（財務情報、日記など）
- デバイス間で最新ファイルを参照したいだけの用途

**Syncthingが苦手なこと：**
- コードや文書の変更履歴管理（gitを使う）
- 差分の精密な操作・ブランチ管理
- 細かいアクセス制御や承認ワークフロー

「gitの代替」でも「iCloud Driveとまったく同じ体験」でもなく、**「クラウドを経由しないiCloud Drive」** として割り切るのが正解だ。

---

## 今後の使い方

現時点での結論として、Syncthingの使い道はこうなった。

```
~/Sync/（Default Folder）
├── obsidian-vault/    ← 個人ノート・日記
├── finance/           ← 投資記録（NISA等）
└── ...（プライバシー重視のものを置いていく）
```

コードやドキュメントのバージョン管理はgitのまま。ObsidianのVaultや財務情報など「バージョン管理は不要だが複数デバイスから見たいもの」をSyncthingに任せる構成だ。

NASを将来的に持てばSyncthingのハブとして活用し、全デバイスへの配布を自動化できる。

---

Syncthingは地味で渋いツールだが、「自分のデータを自分でコントロールしたい」という思想の具体的な実装として、10年以上にわたって生き残っている。クラウド依存を減らす方向でインフラを組み直したい人には、一度試してみる価値がある。
