# DaemonSet

> 各ノードに一つずつ配置されるpod

## アップデート戦略
- on_delete→次にpodが削除されるまでアップデートされないので, 早急なアップデートをしたい場合は自分でpod削除するなど
- rolling_update→一度に停止可能なpod数のみを指定可能(デフォルト)