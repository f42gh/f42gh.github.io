---
title: "Syncthingを試した：Dropboxの代替を探す旅と、gitでは代替できないもの"
layout: post
date: 2026-06-04
tags: [tech, syncthing, tools, self-hosted]
---

[author: threat(claude code agent)]

# Syncthingを試した：Dropboxの代替を探す旅と、gitでは代替できないもの

> 投資記録の管理方法を考えていたら、気づいたらSyncthingを試していた。Web Claudeから「Syncthingはどうか」という提案が出たのがきっかけで、実際に手を動かしてみた記録。

---

## 発端：投資情報をどこで管理するか

個人的な投資記録を整理しようとしたとき、真っ先に思い浮かぶのはGitHub（プライベートリポジトリ）だ。手元のMarkdownをcommitしてpushすれば、バージョン管理・複数デバイスからのアクセス・バックアップが一気に解決する。

ただ、金融情報をGitHub（＝Microsoftのサーバー）に乗せることへの引っかかりがある。もっとプライバシーを担保した方法はないか、と考え始めた。

この話をWeb Claudeに投げると、「Syncthingが選択肢として面白いのでは」という提案が返ってきた。

---

## Syncthingとは

[Syncthing](https://syncthing.net/) は、オープンソースの**P2Pファイル同期ツール**だ。2013年から開発が続いており、2025年8月にv2.0をリリース。GitHubスターは8万超。

特徴を一言で言えば「**中央サーバーなし、アカウントなし、デバイス間で直接同期する**」。

- Dropboxのように「クラウドにアップロードして配布」ではなく、デバイスとデバイスが直接TLS暗号化で通信して同期する
- データはどこのサーバーにも保存されない
- Windows / macOS / Linux / Android 対応
- MIT License、完全無料

v2.0では内部にSQLiteを採用してパフォーマンスが向上。ファイル変更の検知はfsWatcherが担当しており、変更から約10秒で同期が走る実質リアルタイム同期だ。

---

## 既に動いていた

「試してみよう」と言いながらWSL2の環境を確認したところ、Syncthing v1.27.2がすでにインストールされており、MacBookとWindowsマシンの2台がデバイスとして登録済みだった。

```
/home/f42/Sync/    ← Default Folder（2台間で同期済み）
```

以前に設定して忘れていたのか、記憶にない。ともかく環境は整っていた。

---

## 就活情報をjob-huntとして同期してみる

Syncthingのリポジトリ（GitHubで管理している就活情報）をSyncthingで同期してみることにした。目的は「MacのObsidianからMarkdownを参照できるようにする」こと。

ただし.gitディレクトリは含めない方針で進めた。Syncthingを通じた.gitの同期は、両端でgit操作が走ると壊れる可能性があるためだ。

```bash
mkdir -p /home/f42/Sync/job-hunt
git clone --depth=1 https://github.com/f42gh/jh /tmp/jh_tmp
rsync -av --exclude='.git' /tmp/jh_tmp/ /home/f42/Sync/job-hunt/
rm -rf /tmp/jh_tmp
```

これで `/home/f42/Sync/job-hunt/` に約48MBのファイル群が展開され、Syncthingが自動的にMacへ配布した。

Mac側では `~/Sync/job-hunt/` として見える。このディレクトリをObsidianのVaultとして開けばObsidianから就活ノートを閲覧できる。

---

## Syncthingの限界：gitの再現は無理

使いながら気づいた限界がいくつかある。

### 同時編集の問題

Syncthingは**後勝ち方式**だ。両デバイスで同じファイルを同時に編集すると、片方は `filename.sync-conflict-xxxx.md` というConflictコピーとして保存される。元ファイルは後から更新された方で上書きされる。

gitなら `git merge` で統合できるが、Syncthingにその概念はない。「同時に同じファイルを編集しない」という運用ルールを守ることが前提になる。

### .gitignoreのような精密な除外設定が難しい

`.stignore` ファイルで除外設定はできるが、gitignoreほど細かい制御はできない。パターンマッチの表現力が限られている。

### ブランチがない

当然だが、Syncthingにブランチの概念はない。「開発中の変更を別ブランチに逃がす」ようなことはできない。

### 新しいフォルダの追加に両端の操作が必要

Syncthingで新しいフォルダを同期対象に追加するとき、片方で追加してもう片方に承認リクエストが届く形になる。Dropboxのように「フォルダを置いたら自動で同期」とはならない。

ただし、既存のDefault Folder配下にサブディレクトリを作る場合は追加操作不要。

---

## 結論：「Dropboxの代替」として割り切る

一連の検討を経て、Syncthingの適切な位置づけが見えてきた。

**Syncthingが得意なこと：**
- Obsidianのノート、個人メモ、設定ファイルなど、バージョン管理が不要なものを複数デバイスで共有
- データをクラウドに乗せたくないケース（医療情報、財務情報など）
- デバイス間で最新ファイルを参照したいだけの用途

**Syncthingが苦手なこと：**
- コードや文書の変更履歴管理（gitを使う）
- 同時多数人での編集・コラボレーション
- 細かいアクセス制御や承認ワークフロー

「gitの代替」ではなく「Dropboxの代替」として使い分けるのが正解だ。

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

NASを持てばSyncthingのハブとして活用し、全デバイスへの配布を自動化できる。それは就職後の課題として残している。

---

Syncthingは地味で渋いツールだが、「自分のデータを自分でコントロールしたい」という思想の具体的な実装として、10年以上にわたって生き残っている。クラウド依存を減らす方向でインフラを組み直したい人には、一度試してみる価値がある。
