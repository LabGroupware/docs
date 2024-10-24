# IRSA

## 設定なしの認証(OIDC利用可能状態)
1. まずPodのリソースYAMLでService Accountを指定すると, Podの代わりのService AccountがAPI serverに認証するためのトークンを, コンテナ内のファイルにVolumeがマウントされる(`/var/run/secrets/kubernetes.io/serviceaccount/token`).
2. Podはコンテナ内のファイルにVolumeがマウントされたトークンでAPI serverに認証される.

## 設定なしの認可
1. PodはEKSワーカーノード上に起動し, そのEC2インスタンスにはAWS IAM instance profileがアタッチされている
2. つまりPod内のコンテナはEC2のインスタンスメタデータのURL（`169.254.169.254/latest/meta-data/iam/security-credentials/IAM_ROLE_NAME`）から, 一時的なIAM credentialsを取得することができる.

Pod上から以下のコマンドで確認可能.
``` sh
curl 169.254.169.254/latest/meta-data/iam/security-credentials
```

## なぜだめなのか?
ここで問題なのが最小権限のルールが破られて, どのPodも同じEC2のAWS Instance ProfileにアタッチされたAWS IAM Roleにアクセスができること.（つまり, ノードレベルのAWS認可）

## IRSA(IAM Role for Service Account, またはPodレベルのIAM認可)アーキテクチャの解剖
1. `kubectl apply -f`コマンドを使ってPodを起動する時, YAMLがAPI Serverに送られ, API ServerにあるAmazon EKS Pod Identity webhookが, 常時YAMLリソースにService AccountとそのService AccountのAnnotationにAWS IAM Role ARNがあるか見ている.
2. `irsa-service-account`というService Accountに eks.amazonaws.com/role-arn annotationがあるので, webhookがAWS_ROLE_ARNやAWS_WEB_IDENTITY_TOKEN_FILEという環境変数をPodにInjectする(aws-iam-token projected volumeがマウントされる)
3. Service accountがOIDC経由でAWS IAMから認証が取れた後, JWT トークンをOIDCから受け取り `AWS_WEB_IDENTITY_TOKEN_FILE`に保存する.
4. コンテナが`aws s3 ls`などのAWS CLIコマンドを実行すると, Podが`AWS_WEB_IDENTITY_TOKEN_FILE`に保存されたトークンを使って`sts:assume-role-with-web-identity`コマンドを実行し, AWS IAM roleをAssumeする. そのIAM Roleの一時的なCredentialsを使って`aws s3 ls`が実行される.


## Terraformでの実装

1. EKSクラスターのOIDCプロバイダーを作成
これは, マネージドノードの場合, 自動的に作成される.
``` hcl
resource "aws_iam_openid_connect_provider" "oidc_provider" {
  count           = var.enable_irsa && var.create_eks ? 1 : 0
  client_id_list  = [local.sts_principal]
  thumbprint_list = [var.eks_oidc_root_ca_thumbprint]
  url             = flatten(concat(aws_eks_cluster.this[*].identity[*].oidc.0.issuer, [""]))[0]
}
```
2. IAM Roleを作成してPolicyにOIDCプロバイダーの情報を付与する
``` hcl
resource "aws_iam_role" "s3_service_account" {
  name = "s3_sa"
  assume_role_policy = data.aws_iam_policy_document.s3_service_account.json
}

locals {
  eks_namespace = "dev"
}

data "aws_iam_policy_document" "s3_service_account" {
  statement {
    sid = "1"

    actions = [
      "s3:List*",
      "s3:Get*",
    ]

    resources = [
      "arn:aws:s3:::*",
    ]
  }

  // この辺が参考になると思います
  // https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html
  statement {
    effect = "Allow"
    actions = [
      "sts:AssumeRoleWithWebIdentity"
    ]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.oidc_provider.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "${replace(aws_iam_openid_connect_provider.oidc_provider.url, "https://", "")}:sub"
      values = [
        "system:serviceaccount:${local.eks_namespace}:${aws_iam_role.s3_service_account.name}"
      ]
    }
  }
}
```
3. ServiceAccountを作成する
``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: <Terraformで作成したaws_iam_role.s3_service_accountのARN>
  name: s3-sa
```
4. PodにServiceAccountを付与する
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  selector:
    matchLabels:
      app: my-app
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: s3-sa # 作成したServiceAccountを指定
...
```