# Access Entry

> AWS-Auth Config Mapは非推奨となり, Access Entryが推奨となった.

## モードの有効化
`Authentication Mode`に`API`を追加している必要がある.

## タイプ

> クラスターのプラットフォームバージョンに関係なくこれらのロールのエントリを aws-auth ConfigMap に追加するため, マネージド型ノードグループや Fargate プロファイルに使用される IAM ロールのアクセスエントリを作成する必要はない.
> アクセスエントリの作成後にタイプを変更することはできない.

- `EC2 Linux`: Linux または Bottlerocket のセルフマネージド型ノードで使用される IAM ロールの場合
- `EC2 Windows`: Windows のセルフマネージド型ノードで使用される IAM ロールの場合
- `FARGATE_LINUX`: AWS Fargate (Fargate) で使用される IAM ロールの場合
- `STANDARD`: デフォルト

## Kubernetes RBAC

> アクセスエントリのタイプが STANDARD で, Kubernetes RBAC 認証を使用する場合は, アクセスエントリに 1 つ以上のグループ名を追加可能.
> 作成後のグループ名を追加および削除も可能.

1. クラスター上に Kubernetes RoleBinding または ClusterRoleBinding を作成し, グループ名を kind: Group の subject として指定.
2. kubernetes-groupsを複数指定したアクセスエントリーを作成する.ただしアクセスポリシーのアクセス許可の範囲は, 指定した Kubernetes 名前空間に限定することも可能.
3. バインディングの roleRef でも指定した Kubernetes Role または ClusterRole オブジェクトで指定したすべてのクラスターオブジェクトにアクセスすることを IAM プリンシパルに許可する.

タイプが EC2 Linux または EC2 Windows のアクセスエントリを作成する場合, アクセスエントリを作成する IAM プリンシパルには iam:PassRole アクセス許可が必要

## アソシエイトアクセスポリシー

> アクセスエントリーにAWS上で定義っされたポリシーでKubernetes権限を設定したい場合は別途専用のアソシエイトリソースを作成する.
> SAによる権限操作などを行いたい場合はアクセスエントリーで管理する.

### 前提条件
- アクセスエントリーが作成されていること.
- ListAccessEntries, DescribeAccessEntry, UpdateAccessEntry, ListAccessPolicies, AssociateAccessPolicy, および DisassociateAccessPolicy のアクセス許可を持つ AWS Identity and Access Management ロールまたはユーザーであること.

### 考慮点
- 各アクセスエントリには複数のアクセスポリシーを関連付け可能だが, 各ポリシーを1つのアクセスエントリに関連付けることができるのは1回だけとなる.
- アクセスポリシーをクラスター上のすべてのリソースに範囲指定することも, 1つ以上の Kubernetes 名前空間の名前を指定して範囲指定することも可能.
  - 名前空間の名前にはワイルドカード文字を使用可能.
- アクセスポリシーをアクセスエントリに関連付けた後, アクセスポリシーのアクセス範囲を変更可能.
  - 必要に応じて関連付けの名前空間を追加および削除可能.
- グループ名も指定されているアクセスエントリにアクセスポリシーを関連付けると, IAMプリンシパルには, 関連するすべてのアクセスポリシーのすべてのアクセス許可が付与される.

1. 利用可能なアクセスポリシーを表示.
   ``` sh
   aws eks list-access-policies --output table
   ```
1. 既存のアクセスエントリを表示.
    ``` sh
    aws eks list-access-entries --cluster-name my-cluster
    ```
1. アクセスポリシーをアクセスエントリに関連付けをする.

### アクセスポリシーでの許可
- Kubernetes verbs (アクセス許可) と resources を含む rulesがある.
- アクセスポリシーの内容は変更不可.
- 独自のアクセスポリシーを作成不可.

#### AmazonEKSAdminPolicy
リソースに対するほとんどのアクセス許可を IAM プリンシパルに付与するアクセス許可を含む.
アクセスエントリに関連付けられている場合, そのアクセス範囲は通常 1 つ以上の Kubernetes 名前空間となる.

#### AmazonEKSClusterAdminPolicy
IAM プリンシパル管理者にクラスターへのアクセス権を付与するアクセス許可を含む.
そのアクセス範囲は通常、Kubernetes 名前空間ではなくクラスターになる.

#### AmazonEKSAdminViewPolicy
IAM プリンシパルにクラスターの全リソースをリスト/表示するアクセス権を付与するアクセス許可を含む.
Secretリソースも含むことに注意する.

#### AmazonEKSEditPolicy
IAM プリンシパルが Kubernetes リソースを編集できるようにするアクセス許可を含む.

#### AmazonEKSViewPolicy
IAM プリンシパルがほとんどの Kubernetes リソースを表示できるようにするアクセス許可を含む.

## 注意事項
- 1アクセスエントリには1つのIAM プリンシパルかつ1つのIAM プリンシパルに1つまでのアクセスエントリ.
- アクセスエントリのタイプが STANDARD 以外の場合はARN はクラスターと同じ AWS アカウントである必要がある.
- アクセスエントリの作成後に IAM プリンシパルを変更不可(Terraformでは再作成).
- ARN を含む IAM プリンシパルを削除しても, アクセスエントリは自動的に削除されない.
- アクセスエントリのタイプが STANDARD の場合, アクセスエントリのユーザー名を指定可能.
  - ユーザー名の値を指定しない場合, Amazon EKS はアクセスエントリのタイプと, 指定した IAM プリンシパルが IAM ロールか IAM ユーザーかに応じて, 以下のいずれかの値を設定.
  - 独自のユーザー名を指定する場合
    - system:, eks:, aws:, amazon:, または iam: で始めることはできない
  - アクセスエントリの作成後にユーザー名を変更可能.

## ベストプラクティス
- 短期認証情報を持つ IAM ロールを使用.
- 独自のユーザー名を指定する特別な理由がない限り, ユーザー名を指定せず, Amazon EKS に自動生成させる.

| IAM プリンシパルタイプ | タイプ | Amazon EKS が自動的に設定するユーザー名の値
| -- | -- | -- |
| ユーザー | STANDARD | ユーザーの ARN. 例：arn:aws:iam::111122223333:user/my-user |
| ロール | STANDARD | 想定されるロールの STS ARN. {{SessionName}} がロールに追加される.例：arn:aws:sts::111122223333:assumed-role/my-role/{{SessionName}} |
| ロール | EC2 Linux, または EC2 Windows | system:node:{{EC2PrivateDNSName}} |
| ロール | FARGATE_LINUX | system:node:{{SessionName}} |
- ユーザー名が IAM ロール用の場合は, ユーザー名の末尾に {{SessionName}} を追加すること.
  - ユーザー名に {{SessionName}} を追加する場合, ユーザー名の {{SessionName}} の前にコロンを含める必要がある.
  - 自動生成の場合はこの要件を満たしている.

# AWS-Auth

> Amazon EKS クラスターを作成すると, このクラスターを作成した IAM エンティティユーザーまたはロール (例: フェデレーティッドユーザー) は, コントロールプレーンのクラスターの RBAC 設定で system:masters のアクセス許可が自動的に付与されます。この IAM エンティティは, ConfigMap やその他の表示可能な設定には表示されません。そのため, もともとクラスターを作成した IAM エンティティを必ず追跡してください。クラスターを操作する権限を他の AWS ユーザーやロールに付与するには, Kubernetes 内の aws-auth ConfigMap を編集する必要があります。

## クラスタの作成者は?
クラスタを作成したIAMエンティティユーザーなどはデフォルトで`system:masters`(なんでもできる)アクセス許可が付与されている.

## 詳細に設定したい場合は?
他のIAMユーザーやロールにアタッチしたり, 権限を詳細に設定したい場合は, kubernetes_config_mapリソースを作成し, それをClusterRoleにBindする.

## Terraformでの実装

### config map
``` hcl
resource "kubernetes_config_map" "sample_aws_auth" {
  metadata {
    name = "aws-auth"
    namespace = "kube-system"
  }

  data = {
    mapRoles = <<YAML
    - rolearn: <紐付けたいIAM RoleのARN> 
    username: teama_developer
    groups:
    - teama:view
YAML
  }
}
```

### cluster role
``` hcl
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: developer-view-binding
subjects:
  - kind: Group
    name: teama:view # aws-authに登録したグループ
roleRef:
  kind: ClusterRole
  name: view # Kubernetesがデフォルトで持っているClusterRole
  apiGroup: rbac.authorization.k8s.io
```