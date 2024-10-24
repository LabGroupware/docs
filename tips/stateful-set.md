# StatefulSet

> ステートフルなワークロードに対応しており, persistent_volumeを使う場合にpodの再起動時, 同じディスクを利用して再作成されるため, 永続化に適した仕組みを所持.
> podのスケールインは最新に作られたものから削除されるため, 一番古いもの(0インデックス)をマスタノードとするような冗長構成に向いている.

## アップデート戦略
- DaemonSetと同様
- rolling_updateの停止可能なpod数など指定不可で常にインデックスの大きい順に一つずつ更新され, partitionの指定によりそのインデックスより小さいインデックスのpodに関して更新を行わず, 部分的な更新による様子見が可能.

## PersistentVolumeClaim
- StatefulSetに限り, その中のプロパティでPVCを設定可能.