# Deployment

## アップデート戦略
- recreate→ダウンタイムは発生するが, 切り替えが早い.
- rolling_update→許容する不足数, 超過pod数(それぞれパーセンテージでも可)を設定可能(デフォルト).