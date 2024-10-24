# Structure

> AWSのEKS上にマイクロサービスを構築する上でのアーキテクチャレイヤとそれぞれで使用するツールを定義する.

## インフラストラクチャ層

> AWSのセットアップから, Kubernetesのインフラよりのセットアップまでを担当する.
> IaCを利用することでコード管理し, 全てを1レポジトリで取り扱う.
> CI/CDは取り入れず, 必ず`terraform plan`などで確認してから変更を行う.
> この段階でのKubernetesやHelmの管理はmanifestやhelmfileではなく, 全てTerraformのプロバイダで利用する.

### Repo/Dir Style

```
infra-repo
├── composition
│   ├── pattern1-infra
│   │   └── ap-northeast-1
│   │         └── prod
│   │             ├── backend.config
│   │             ├── clusterrolebinding_k8s_developer.yaml
│   │             ├── data.tf
│   │             ├── main.tf
│   │             ├── outputs.tf
│   │             ├── providers.tf
│   │             ├── terraform.tfvars
│   │             ├── variables.tf
│   │             └── versions.tf
│   ├── pattern2-infra
│   │   └── ap-northeast-1
│   │         └── prod
│   │             ├── backend.config
│   │             ├── clusterrolebinding_k8s_developer.yaml
│   │             ├── data.tf
│   │             ├── main.tf
│   │             ├── outputs.tf
│   │             ├── providers.tf
│   │             ├── terraform.tfvars
│   │             ├── variables.tf
│   │             └── versions.tf
│   └── terraform-remote-backend
│       └── ap-northeast-1
│   │         └── prod
│   │             ├── data.tf
│   │             ├── main.tf
│   │             ├── outputs.tf
│   │             ├── providers.tf
│   │             ├── terraform.tfvars
│   │             └── variables.tf
├── infrastructure_modules
│   ├── pattern1-eks
│   │   ├── data.tf
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   ├── template
│   │   │   └── ssm_document_cleanup_docker_images.yaml
│   │   └── variables.tf
│   ├── pattern2-eks
│   │   ├── data.tf
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   ├── template
│   │   └── variables.tf
│   ├── terraform-remote-backend
│   │   ├── data.tf
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   └── variables.tf
│   └── vpc
│       ├── data.tf
│       ├── main.tf
│       ├── outputs.tf
│       ├── provider.tf
│       └── variables.tf
└── resource_modules
     ├── compute
     │   ├── ec2
     │   ├── key-pair
     │   ├── lambda
     │   └── sg
     ├── container
     │   ├── ecr
     │   ├── ecs
     │   └── eks
     ├── database
     │   ├── dynamodb
     │   │    ├── README.md
     │   │    ├── autoscaling.tf
     │   │    ├── examples
     │   │    │   ├── autoscaling
     │   │    │   │   ├── README.md
     │   │    │   │   ├── main.tf
     │   │    │   │   ├── outputs.tf
     │   │    │   │   ├── variables.tf
     │   │    │   │   └── versions.tf
     │   │    │   ├── basic
     │   │    │   │   ├── README.md
     │   │    │   │   ├── main.tf
     │   │    │   │   ├── outputs.tf
     │   │    │   │   ├── variables.tf
     │   │    │   │   └── versions.tf
     │   │    │   ├── global-tables
     │   │    │   │   ├── README.md
     │   │    │   │   ├── main.tf
     │   │    │   │   ├── outputs.tf
     │   │    │   │   ├── variables.tf
     │   │    │   │   └── versions.tf
     │   │    │   └── s3-import
     │   │    │       ├── README.md
     │   │    │       ├── files
     │   │    │       │   ├── sample.csv
     │   │    │       │   └── sample.json
     │   │    │       ├── main.tf
     │   │    │       ├── outputs.tf
     │   │    │       ├── variables.tf
     │   │    │       └── versions.tf
     │   │    ├── main.tf
     │   │    ├── outputs.tf
     │   │    └── variables.tf
     │   ├── elasticache
     │   └── rds
     ├── identity
     │   ├── iam
     │   └── kms
     ├── integration
     │   └── msk
     ├── management
     │   └── cloudwatch
     ├── network
     │   ├── route53
     │   └── vpc
     ├── security
     │   └── sm
     └── storage
         ├── efs
         └── s3
```

### Technology Stack
- Terraform
  - AWS
  - Kubernetes
  - Helm

### Test
- terraform planによる確認.
- test環境などでのシステムテスト.

### Responsibilities
- awsネットワーク自体のセットアップ
- Namespaceの作成.
- 全namespaceが関連するconfigMapやSecret.
- ELBとingressのセットアップ.
- 各ネームスペースの操作可能なIAMアカウントなどの作成(aws-auth).
- 必要なpodを操作するためのserviceAccount(IRSA).
- ネームスペース単位のアクセス制御を行うNetworkPolicy.
- サービス間通信用のメッセージブローカーのセットアップ.
- EBS, EFSのPersistentVolume.
- S3のセットアップ.
- 各サービスの利用する外部データベースのセットアップ.
- ログ, メトリクス, 分散トレースのための中央集権サーバーのセットアップ.
- terraform自体の状態管理.

## コンテナオーケストレーション層

> ImageをPushされたことをトリガーにCI/CD Pipelineを動かし, サービスの更新を行う.
> リポジトリはパターンのみ分割しており, 各サービスに関してはmonorepo構成で行う.
> Helmによる外部helmなどの依存関係の解決やvalue.yamlを使用した環境の切り替えなどを可能にする.
> helmfileで管理することで, 複数のserviceのhelmを一括管理可能.

### Repo/Dir Style

```
pattern1-kubernetes-repo/
├── base/                           # 共通のベース設定(Infrastructure層でカバーしきれない範囲)
│   ├── ingress.yaml
│   └── networkpolicy.yaml
├── environments/                   # 各環境ごとの設定
│   ├── dev/                        # 開発環境のリソース
│   │   ├── values-dev.yaml         # 開発環境の設定ファイル (共通)
│   │   └── helmfile.yaml           # 開発環境のHelmfile (サービスごとのリリース設定)
│   └── prod/                       # 本番環境のリソース
│       ├── values-prod.yaml        # 本番環境の設定ファイル (共通)
│       └── helmfile.yaml           # 本番環境のHelmfile (サービスごとのリリース設定)
├── services/                       # 各サービスのHelmチャート
│   ├── service-a/
│   │   ├── helm/
│   │   │   ├── Chart.yaml          # Helmチャート
│   │   │   └── values.yaml         # サービスA用の共通設定
│   │   └── deployment.yaml         # Kubernetesリソース (必要なら)
│   └── service-b/
│       ├── helm/
│       │   ├── Chart.yaml          # Helmチャート
│       │   └── values.yaml         # サービスB用の共通設定
│       └── deployment.yaml         # Kubernetesリソース (必要なら)
└── ci-cd/                          # CI/CDパイプライン設定
    └── pipelines/


pattern2-kubernetes-repo/
├── base/                           # 共通のベース設定(Infrastructure層でカバーしきれない範囲)
│   ├── ingress.yaml
│   └── networkpolicy.yaml
├── environments/                   # 各環境ごとの設定
│   ├── dev/                        # 開発環境のリソース
│   │   ├── values-dev.yaml         # 開発環境の設定ファイル (共通)
│   │   └── helmfile.yaml           # 開発環境のHelmfile (サービスごとのリリース設定)
│   └── prod/                       # 本番環境のリソース
│       ├── values-prod.yaml        # 本番環境の設定ファイル (共通)
│       └── helmfile.yaml           # 本番環境のHelmfile (サービスごとのリリース設定)
├── services/                       # 各サービスのHelmチャート
│   ├── service-a/
│   │   ├── helm/
│   │   │   ├── Chart.yaml          # Helmチャート
│   │   │   └── values.yaml         # サービスA用の共通設定
│   │   └── deployment.yaml         # Kubernetesリソース (必要なら)
│   └── service-b/
│       ├── helm/
│       │   ├── Chart.yaml          # Helmチャート
│       │   └── values.yaml         # サービスB用の共通設定
│       └── deployment.yaml         # Kubernetesリソース (必要なら)
└── ci-cd/                          # CI/CDパイプライン設定
    └── pipelines/
```

helmfile.yaml
``` yaml
releases:
  - name: service-a
    namespace: service-a
    chart: ../../services/service-a/helm
    values:
      - ../../services/service-a/helm/values.yaml
      - values-dev.yaml

  - name: service-b
    namespace: service-b
    chart: ../../services/service-b/helm
    values:
      - ../../services/service-b/helm/values.yaml
      - values-dev.yaml

```

### Technology Stack
- Kubernetes
- Helm

### Test
- Helmデプロイ前の確認.

### Responsibilities
- 各ネームスペースでのDeployment, Service, (Cron)Job, StatefulSet(dbなど), DaemonSetなどのマニフェスト.
- Helmチャートの作成, Operatorの依存解決.
- リソース制限やスケーリングの設定.
- istioなどのサービスメッシュの追加.
- 環境/パターンごとのKubernetesへのデプロイ.

## アプリケーション層

> コーディングを行った後, Dockerのイメージビルドまでを担当する.
> 各ドメインサービス以外にも, OAuthサービスやGateway, Job管理サービスなど, スクラッチ実装するサービスは全て行う.
> サービスごとのレポジトリを構成し, 環境は別ディレクトリやリポジトリで管理するのではなく, 環境変数により指定できるようにする.(ただし, 異なるパターンに関しては別サービスという程で, 全く別のリポジトリに置く)

### Repo/Dir Style

```
pattern1-service-a-repo/
├── pipeline/
...

pattern1-service-b-repo/
├── pipeline/
...

pattern2-service-a-repo/
├── pipeline/
...

pattern2-service-b-repo/
├── pipeline/
...
```

### Technology Stack
- 各プログラミング言語
  - Java Spring
  - Go
- Docker
- Docker Compose
  - Database
  - Message Broker
- Jenkins

### Test
- クラスやメソッドのUnit Test
- Docker Composeを利用したIntegration Test

### Responsibilities
- 各サービスなどのコーディング
- Docker Imageのビルド
- CIでの自動テスト, ECRへのプッシュ
- サービス稼働の保証