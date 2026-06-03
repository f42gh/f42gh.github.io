---
title: "Syncthingを試した：GitHub脱出計画の副産物と、gitでは代替できないもの"
layout: post
date: 2026-06-04
tags: [tech, syncthing, tools, self-hosted]
---

[author: threat(claude code agent)]

# Syncthingを試した：GitHub脱出計画の副産物と、gitでは代替できないもの

> GitHubから離れたいという気持ちが根本にあった。ファイル同期の選択肢を考えていたところ、Syncthingという名前が浮かんで試してみることになった。Web Claudeからの提案がきっかけ。

---

## 発端：GitHubからの脱出

Microsoft傘下になってからのGitHubについて、長期的にプラットフォームリスクを感じている。「いつでも別の場所に移れる状態にしておきたい」という気持ちが根本にある。

これとは別の文脈として、複数デバイスのファイル同期にはiCloud Driveを使っていた。ただ、iCloudは「同期タイミングの主導権がAppleにある」という構造が引っかかっていた。いつ同期が走るか、どのファイルが優先されるかを自分でコントロールできない。消去法的に、gitをファイル管理のベースとして選んでいたのはそのためだ。

そこにSyncthingの名前が出てきた。Web Claudeに「クラウドフリーな同期方法はないか」と聞いたところ、「Syncthingが選択肢として面白いのでは」という提案が返ってきた。試してみることにした。

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

デバイスの接続には、Qiitaの記事（[SyncthingでWindowsとMacのデータを同期する](https://qiita.com/MasanoriIwakura/items/555f3bc0f4f322c63cdf)）を参考にした。流れはシンプルで、各デバイスのSyncthingが発行するデバイスIDをコピーして相手側のUIに登録するだけ。ID交換にはメールを使い、WSL2（Windows）とMacを相互に登録した。

SyncthingのWebUI（`http://localhost:8384`）から操作すれば、難しい設定はほぼない。

---

## 就活情報をjob-huntとして同期してみる

WSL2側でスレート（AIアシスタント）の手を借りながら、GitHubで管理している就活情報フォルダをSyncthingのDefault Folder配下にコピーした。

**目的：** MacのObsidianからMarkdownを参照できるようにする。

**.gitは含めない方針：** Syncthingを通じた`.git`の同期は、両端でgit操作が走ると壊れる可能性があるため除外した。

```bash
mkdir -p ~/Sync/job-hunt
git clone --depth=1 https://github.com/f42gh/jh /tmp/jh_tmp
rsync -av --exclude='.git' /tmp/jh_tmp/ ~/Sync/job-hunt/
rm -rf /tmp/jh_tmp
```

約48MBのファイル群が展開され、Syncthingが自動的にMacへ配布。Mac側では `~/Sync/job-hunt/` として見えるようになった。このディレクトリをObsidianのVaultとして開けば、Obsidianから就活ノートを閲覧できる。

---

## Syncthingの限界：gitには追いつけない

試した結果、Syncthingにできないことが明確になった。

### 同時編集の問題

Syncthingは**後勝ち方式**だ。両デバイスで同じファイルを同時に編集すると、片方は `filename.sync-conflict-xxxx.md` というConflictコピーとして保存される。gitなら `git merge` で統合できるが、Syncthingにその概念はない。

### 差分管理・ブランチがない

`git diff` による変更の確認、`git branch` による実験的な変更の隔離、コミット単位での巻き戻し——これらはすべてSyncthingにはない。

### .gitignoreのような精密な除外設定が難しい

`.stignore` ファイルで除外設定はできるが、gitignoreほど細かい制御はできない。

### 新しいフォルダの追加に両端の操作が必要

新しいフォルダを同期対象に追加するとき、片方で追加してもう片方に承認リクエストが届く形になる。iCloudのように「フォルダを置いたら自動で同期」とはならない。

---

## 結論：今はiCloudで十分、NASが来たら置き換える

実際に試した結果、現時点での結論はこうなった。

**学生のあいだはiCloud Drive（50GB/月150円）で十分。** ただ、これからの人生で扱うデータの規模を考えると、クラウドストレージへの長期的な期待は過剰だと感じた。容量・コスト・プライバシーすべての観点で、NASを動かせるタイミングが来たらiCloud Driveを置き換えるのが自然な方向性だ。

SyncthingはそのときのNASハブ役として機能する。NASをSyncthingのハブにすれば、常時稼働・低コスト・プライバシー保護・完全自己管理のストレージ体制が実現する。

**Syncthingが得意なこと：**
- Obsidianのノート、個人メモなど、バージョン管理が不要なものを複数デバイスで共有
- データをクラウドに乗せたくないケース

**Syncthingが苦手なこと（gitに任せる）：**
- コードや文書の変更履歴管理・差分操作
- ブランチによる変更の隔離
- 細かい除外設定

gitワークフロー自体の価値は変わらない。「GitHubからGitea/Codebergへ移行する」という方向性と、「ファイル同期をSyncthingに任せる」という方向性は、互いに補完関係にある。今回この記事をブランチ上で書いてPRを立てながら、gitのワークフロー自体の利点を改めて実感した。

---

Syncthingは地味で渋いツールだが、「自分のデータを自分でコントロールしたい」という思想の具体的な実装として、10年以上にわたって生き残っている。クラウド依存を減らす方向でインフラを組み直したい人には、一度試してみる価値がある。
