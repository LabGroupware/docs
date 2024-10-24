# Job
- n並列で指定回数の正常終了を保証する.
- 許容する失敗回数の指定も可.
- 成功回数のみ後から変更不可.
- replica_setのようなlabelとselectorの指定は推奨されていない(k8sがuuidを自動生成).
- podのspecのrestart_policyはneverではpodの障害時に新規pod作成, on_failureでは再度同一podでjobの再開, ただしvolumeのマウントなどしてなければデータは消失.

# CronJob
- jobの定期実行を行い, 同期実行に関してallowなら制御なし, forbidなら前のjobが終了してないなら行わない, replaceなら前のjobをキャンセルして開始.