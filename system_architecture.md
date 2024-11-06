# システムアーキテクチャ

> システムアーキテクチャ構成を定義する.

## 一般技術スタック
- **AWS**: nonlocalの環境ではAWSを利用.
  - **Network**: ネットワーク関連のサービスでL4レベルのアクセス制御を行う.
  - **Cognito**: テレメトリ監視などの管理者用サービスの認証・認可.
  - **EKS**: Kubernetesのクラスタ管理.
- **Kind**: local開発環境でのKubernetesクラスタ管理.
- **Istio/Kiali**: サービスメッシュとその可視化.
- **Prometheus/Grafana**: メトリクス監視とその可視化.
- **Jaeger**: 分散トレーシング.
- **Kubernetes/Helm/Helmfile**: OrchestratorとKubernetesリソース定義など.
- **Docker Compose**: 各サービス時のインフラ代用.
- **gRPC/Buf(BSR)**: サービス間同期通信とそのスキーマ定義レジストリ.
- **Kafka/Confluent Schema Registry(CSR)**: サービス間非同期通信とそのスキーマ定義レジストリ.
- **debezium**: CDCのための利用.
- **postgres/mongoDB**: データベース.
- **Buildpacks**: アプリケーションImageパッケージング.
- **EventStoreDB**: イベントソーシング用イベントストア.

## サービス分割

> 今回はBusiness Capabilitiesパターンで機能ごとの分割を行う.

### Domain
- [参考](./system_overview.md)

### WebSocketサービス (WebSocket Service)
- **責務**: リアルタイムでのイベント通知とデータ配信.
- **技術スタック**: Go, Redis, Kafka.
  - API gatewayは通らず, WebClientとの接続を保ち, SocketIDとUserID, Subscribeしたいレベルなどの結びつけを行っておく
  - Redisにてソケット情報とユーザーID, JobIDの紐付けを集権管理.
  - KafkaでJob管理サービスからのメッセージングをサブスクライブしておき, UserIDとClientのSubscribeレベルのフィルタリングによって該当接続Clientのソケットへメッセージを送信する
  - ソケットの接続の完了と切断に関して, publishを行う.
- **主な機能**:
  - クライアントとのWebSocket接続の維持.
  - イベント駆動型の通知の送信.
  - 他のサービス（特にJob管理サービス）からのイベントの購読.
  - 接続に関するpublish.

### Job管理サービス (Job Management Service)
- **責務**: 非同期タスクの管理と状態追跡.
- **技術スタック**: Go, Kafka, Redis.
  - 有効期限付きでJobを保存する, リアルタイム性が求められるためRedisを利用.
  - Jobの状態確認のリクエストはgRPCで返却.
  - JobID発行以降のjob状態更新リクエストはKafkaによるイベントディスパッチをリッスンしたもので対応.
  - データを変換後, Subscribeしているサービスにパブリッシュする
- **主な機能**:
  - ジョブIDの発行とジョブの進行状況の追跡.
  - 各サービス間のタスク完了や失敗の通知.
  - APIゲートウェイとの連携を通じてジョブの管理と更新を行う.
  - 各ドメインサービスからJobに関する更新, 完了, 失敗を保持.

### API Gateway
- **責務**: クライアントからの単一エンドポイント.
- **技術スタック**: Spring.
  - マネージドサービスやOSS, Ingressなどではカスタマイズが難しく, 特にAPI CompositionやKafkaへのメッセージングなど通常のルーティングのみで完結することではないためスクラッチ実装.
  - フロントからの呼び出しはRESTで受付.
  - JobIDの発行のためのJob管理サービス呼び出しやauthサービスの呼び出し, クエリ, コマンド呼び出しはgRPCを利用.
- **主な機能**:
  - API Compositionパターンの場合, データの収集と結合.
  - クエリであればgRPC, コマンドであればKafkaへのルーティング.
  - コマンドはJob管理サービスからJobIDを受け取り次第, メッセージングを行う.
  - トレーシングのためのID発行とコンテキストへの付与.
  - メッセージングを送信できるためのリクエスト簡易バリデーション.

## 流れ
### コマンドに関して
- APIゲートウェイはメッセージスキーマのためだけの最低限のバリデーションを行った後, Job管理サービスにJobIDを発行してもらう
- 中心となるドメインサービスにgRPCで送信後, このドメインサービスがSagaオーケストラとなり, Kafkaを用いてSaga実行. この時各プロセスの完了や失敗など全てをイベントパブリッシュする
- 全トランザクション完了時にはそのイベントもパブリッシュする
- これらのイベントパブリッシュにより, Job管理サービスや各関連ドメインサービス, Notificationサービス, CQRSの場合, クエリ専用ドメインサービスが同期や処理を開始する

### クエリに関して
- api compositionではapi gatewayがgrpcにて各ドメインサービスへのcompositionを行う
- cqrsではapi gatewayがgrpcにてクエリ専用サービスへプロキシする