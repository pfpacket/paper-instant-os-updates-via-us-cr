# 先端 2017/10/26
Instant OS Updates via Userspace Checkpoint-and-Restart

Sanidhya Kashyap Changwoo Min Byoungyoung Lee Taesoo Kim Pavel Emelyanov †
Georgia Institute of Technology † CRIU & Odin, Inc.

---

## 概要
* カーネルを再起動なしに迅速に更新するインスタントアップデート
* KUP = kernel live patching + CRIU
* CRIUはアプリケーションの状態を保持の為
* 恐らく、技術的には、再起動している
* kernel live patching + CRIUを用いたKUPツールについての論文

---

## KUP 利点
* kpatchやksliceと比べて、カーネルの内部データ構造を変更する様なパッチでも最小ダウンタイムで更新
* 更新失敗時は、失敗前にrollback
