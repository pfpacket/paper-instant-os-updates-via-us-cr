# 先端 2017/10/26
[Instant OS Updates via Userspace Checkpoint-and-Restart](https://www.usenix.org/sites/default/files/atc16_full_proceedings.pdf)

Sanidhya Kashyap Changwoo Min Byoungyoung Lee Taesoo Kim Pavel Emelyanov †
Georgia Institute of Technology † CRIU & Odin, Inc.

副題: 如何にプロセスマイグレーションに応用できるか。

---

## 概要
Linuxカーネルに対するライブパッチングの新たなツール _KUP_ に関する論文。

KUPは、セキュリティから機能追加までLinuxカーネルに対するあらゆる種類のパッチを現在実行中のカーネルに動的にパッチをあててカーネルを更新するツール。

既存のkspliceやkpatchなどは、カーネル内部のデータ構造に変更を加えないパッチのみ利用可能であった。

---

### 動機
* セキュリティが最大懸案事項たる現代、機能追加も含めて更新(update)は不可避
* しかし、その修正を適用するため、再起動するのは、商用システムでは大きなダウンタイムと損失を伴う
* カーネルに対するライブパッチングが不可欠
* しかし既存のライブパッチツールは、肝心なセキュリティ修正パッチさえ、うまく適用できないものが存在
* KUPであらゆるパッチをダウンタイム無しで実行中のカーネルに適用可能にする

---

## KUPの実現目標
* 実行中のカーネルを瞬時に更新
* 観測可能ないかなるダウンタイムも回避
* カーネルに対する変更を皆無に
* あらゆるパッチに対応  
既存のkernel live patchingはカーネル内部のデータ構造に対する変更を許さなかったが、KUPではこれを許可する

---

### システムの更新
特にカーネルの更新は通常再起動を要する。しかし、アプリケーションの状態を破棄することになる。

e.g. Facebookのmemcachedサーバ(キャッシュサーバ)は、144GBのRAMにおよそ120GBのデータをキャッシュしており、90から100分の準備時間が必要になる。

さらに、システムの更新が失敗したときのフォールバックは致命的。さらに、余分な作業とダウンタイムを生む。

+++

### これまでの対策方法
大きく以下の２つ

* 動的hot-patching(=live patching)
* ローリングアップデート  
はじめに小さな集団にその変更を適用して問題がないことを確認してから、本番環境に適用する。
更新失敗とダウンタイムのリスク軽減の為、入念な計画を必要とする。
* マイクロカーネル的手法
マイクロカーネルではカーネルの各コンポーネントが分離して、IPCとインターフェースで抽象化。
問題のあるコンポーネントを修正済みコンポーネントと置換しても問題がない。
既存のカーネルにおいては膨大な変更が必要、現実的でない。

---

### KUPライフサイクル
1. プロセスをダンプ (checkpointing)  
後述CRIUでプロセスをcheckpoint
2. ダンプしたデータを不揮発記憶装置に保存 (archiving)
カーネル入れ替え後、ダンプしたプロセスを復帰(restore)するため
3. 新旧カーネル入れ替え (switching kernel)  
メモリに新カーネルをロード。
4. 新カーネルブート  
完全な再起動ではないため、BIOSのPOST処理など時間のかかる処理をすっ飛ばす。
5. システムサービス初期化
これはsystemdなどのinitが管理するのでcheckpointしない
6. プロセス復帰 (resotring)
CRIUでダンプしたプロセスをCRIUで復帰させる

---

## [CRIU](https://criu.org/Main_Page)
### Checkpoint/Restore In Userspace

![CRIU](assets/criu_logo.jpg)

+++

## [CRIU](https://criu.org/Main_Page)
### Checkpoint/Restore In Userspace

```
# CRIUでPID 15798をダンプして、~/tmp/criu_dump/15798 に
sudo criu dump --images-dir ~/tmp/criu_dump/15798 --leave-stopped --tree 15798

# 何らかの方法でリモートに送信

# CRIUでディスクに保存されたプロセスを復帰させる
sudo criu restore --tree 15798 --images-dir ~/tmp/criu_dump/15798/ --shell-job
```

PID namespaceを切ってやるとPID衝突が起こらない!

---

## On-demand restore
CRIUでディスクにダンプされたデータを順にリストアすると、許容不可能なダウンタイムが生じる。

KUPでは、メモリの内容の総てをリストアせずして、プロセスがある特定のアドレスにアクセスしようとしたとき、これをロードする。
全体としてのダウンタイムの大幅な短縮を達成。

---

## 分散システムへの応用
ネットワーク上のノード間のプロセスマイグレーションに利用可能。
プロセスマイグレーションは計算資源共有と耐障害性に益する所多し。
詳しくは、LTスライド参照。

KUPのカーネル更新部分は無関係。

KUPのプロセスの効率的高速なC/Rは、直接プロセスマイグレーションに応用可能。
