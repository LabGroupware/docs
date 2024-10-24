# システム概要

> 今回考えるシステムの概要についての説明を行う.

## 最終機能概要

研究用グループウェアシステム(LabGroupware)を今回の研究対象とし, 以下にその機能を示す.

### 認証サーバー
- state, eventのサービス分割なし.
- ユーザーアカウント, OAuthによるログイン.
``` txt
email: メールアドレス.
password_hash: パスワードハッシュ値.
```
- プロファイルの作成, 削除, 更新.
``` txt
name: フルネーム.ユーザーの名前全体が含まれます.
given_name: 名(first name).例えば「John」など.
family_name: 姓(last name).例えば「Doe」など.
middle_name: ミドルネーム(省略されることが多い).
nickname: ユーザーのニックネーム.
preferred_username: ユーザーが好むユーザー名.このユーザー名は一意である必要はありません.
profile: ユーザーのプロフィールページのURL.
picture: ユーザーのプロフィール画像のURL.
website: ユーザーのウェブサイトURL.
phone: ユーザー電話番号.
gender: ユーザーの性別(例：male、female、otherなど).
birthdate: ユーザーの生年月日(YYYY-MM-DD形式).
zoneinfo: ユーザーのタイムゾーン(例：Europe/Paris).
locale: ユーザーの言語や地域設定(例：en-US).
updated_at: ユーザーのプロフィールが最後に更新された時刻(Unixタイムスタンプ).
```

### UserPresence
- state, eventのサービス分割なし.
- ユーザーのリアルタイムアクティビティ.
``` json
{
  "_id": "unique_user_identifier",             // ユーザーの一意識別子
  "online": [
    {
      "type": "webSocket",
      "socket_id": "xxxx-xxxx-xxxx-xxxx"
    },
    {
      "type": "webSocket",
      "socket_id": "yyyy-yyyy-yyyy-yyyy"
    },
    {
      "type": "mobile",
      "device_id": "aaaa-aaaa-aaaa-aaaa"
    }
  ],
  "status": "online", // online状態のみではなく, meeting中などの場合もあるため
  "last_seen": "2024-10-23T12:34:56Z",
  "created_at": "2024-01-01T00:00:00Z",         // ユーザードキュメントの作成日時
  "updated_at": "2024-10-23T12:34:56Z"          // ユーザードキュメントの最終更新日時
}
```
- ユーザーデバイス情報.
``` json
{
  "_id": "unique_user_identifier",             // ユーザーの一意識別子
  "devices": [                                 // デバイス情報を格納する配列
    {
      "device_id": "unique_device_id",          // デバイスの一意識別子 (UUID)
      "device_info": "Windows 10, Chrome 95",   // デバイス情報 (例: OS, ブラウザ)
      "ip_address": "192.168.1.1",              // ログイン時のIPアドレス
      "first_logged_at": "2024-10-23T12:34:56Z",// 初回ログイン日時
      "last_logged_at": "2024-10-23T15:34:56Z", // 最終ログイン日時
      "location": "Tokyo, Japan",               // ログイン時の地理的な位置
      "is_active": true                         // デバイスがアクティブかどうか
    }
  ],
  "created_at": "2024-01-01T00:00:00Z",         // ユーザードキュメントの作成日時
  "updated_at": "2024-10-23T12:34:56Z"          // ユーザードキュメントの最終更新日時
}
```

### UserPreferences
- state, eventのサービス分割なし.
- UI, 通知カスタマイズ.
``` txt
theme: ユーザーの好むテーマ(light, dark, system).
notification_settings: メール通知やアプリ内通知の設定.
language_preference: ユーザーの好む表示言語(en-US, ja-JP).
timezone_preference: ユーザーの好むタイムゾーン(Europe/Paris).
```

### Organization
- organizationを作成すると, そのorgにグループウェアが適用される.
- userを追加することができる.
``` txt
name
plan
domain
```
- organization_user
``` txt
organization_id
user_id
added_at
```

### Team
- チームの作成, チーム情報の変更や削除, org内のユーザーの追加, 削除などが可能.
- チームの中には, サブチームの作成が可能であり, サブチームも通常チームと同様に扱われ, 同様のリソースを保持するが, 以下のような事項が異なる.
  - 親チームのユーザーのみ所属可能.
  - 親チームが削除されたり, 親チームを抜けた際は, 子チームも自動的に削除, 脱退することとなる.
- タグ機能が存在しており, チームのユーザーにタグをつけることで他リソースからの参照時に識別することができる.
- 以下のorganization作成時には特殊なチームが存在しており, 通常の操作ではチームへの加入や脱退が不可能である.また, サブチームの作成も認められない.
  - organizationと同じ名前のチーム.
    - defaultチームがデフォルトで存在しており, ユーザーは作成された段階で必ずここに所属することになる.
- team
``` txt
name
description
parent_team_id
is_default
```
- tag
``` txt
name
team_id
color
```
- team_user
``` txt
team_id
user_id
added_at
```

### Plan
- Planという最も大きな単位の中に複数のバケットを作成することが可能.
- plan
``` txt
user_id or team_id or organization_id
type: user or team or organization
title
description
```
- bucket
``` txt
plan_id
title
description
```
- バケットの中にはタスクをいくつも定義可能.
- Planはuser, organization, team単位で作成可能で, これらは複数のPlanを保持できる.
- タスクの進行状況ステータス, ラベルは, Planごとに自分での定義が可能.
- タスクの登録項目としては, 少なくとも以下のような項目がある.
  - bucket_id
  - タイトル
  - 詳細
  - charge(ユーザー単位, チーム単位, サブチーム単位, タグ単位, またはその複数組み合わせ)
  - observer(ユーザー単位, チーム単位, サブチーム単位, タグ単位, またはその複数組み合わせ)
  - 開始日時
  - 期限日時
  - timezone
  - 複数の添付ファイルや画像
  - 複数ラベルid
  - 進行状況ステータスid
  - チェックリスト
  - is_repeat
  - repeat_setting
- label
``` txt
plan_id
title
color
rank
```
- progress_status
``` txt
plan_id
status
color
rank
```
- イベント発火設定の定義が可能.これを利用することでAutoFlowでのメール送信サービスなどとの連携が可能.
  - 下記の定義により, 取得できる変数に代入される.
  - beforeでの指定が可能なため, AutoFlowのCronで行うよりも推奨.
``` txt
filter
before_target: string(start_time or expiration_time)
before: 何日前, 何時間前, 何分前など
targets: array of string(valid values is observer or charge)
level
```
- Viewを保持可能である.
- view
``` json
{
  "plan_id": "xxxx-xxxx-xxxx-xxxx"
  "orders": [
    {"target": "expiration_time", "method": "asc"},
    {"target": "progress_status": "method": "desc"}
  ],
  "group_by": [
    {"target": "label"}
  ],
  "filter": {
    "type": "and",
    "target": [
      {
        "type": "cmp",
        "target": "start_time",
        "operation": "le",
        "value": "2024-10-25 12:34:56"
      },
      {
        "type": "or",
        "target": [
          {
            "type": "cmp",
            "target": "label",
            "operation": "in",
            "value": ["aaaa-aaaa-aaaaa-aaaaa", "bbbbb-bbbb-bbbbb-bbbbb"]
          },
          {
            "type": "cmp",
            "target": "bucket",
            "operation": "eq",
            "value": "infrastructure"
          },
        ]
      }
    ]
  },
  "type": "table"
}
```
- タスクに対して, 一つのコメントスレッドが作成されているので, やり取りが可能.
- comment
``` txt
task_id
user_id
content
sent_at
```

### Chanel
- チーム内にChanelを複数作成可能であり, チーム作成時にはチャネル(Default)が作成されている.
- Chanelには, 投稿することが可能で, その投稿にはスレッドができる.
- スレッド内でコメントなどを投稿することが可能.
- reaction, reply, mention, 転送, 全文検索

### Storage
- 単なるアップロード, ダウンロード, 削除を行う他に, メタ情報も保持しており, 他リソースとのリレーションも行いやすいようにID管理を行っている.
- ユーザー, チーム, orgには1つのストレージが与えられており, ストレージにはバケットを作成することができる.
- バケットはファイルシステムになっており, その中に対して自由にディレクトリやファイルの作成が可能.
- ストレージには, Defaultバケットが存在する.
  - 例えばteamであれば, 中には以下のディレクトリが用意されている.
    - /Profile: チームのプロフィール画像など一般情報に関するファイルが保存される.
    - /Chanel/{Chanel_id}: チャネル内のファイルや画像などが保存される.
    - /Chat: チャットトーク内のファイルや画像などが保存される.
    - /Plan/{Plan_id}/Task/{Task_id}: タスクに添付されるファイルや画像などが保存される.
    - /Book: ブックに添付されるファイルや画像などが保存される.

### Chat
- organization, teamに所属する.(ただし, teamは1:1の関係)
- 自由にチャットルームを作成して, ユーザーを追加することができる.
- org内の個人ユーザーに対するチャットも可能.
- チーム作成時に自動でそれに紐づくチャットルームが作られるが, そのチャットルームに対しては, 直接チャットルームの削除やユーザーの追加などの操作は不可能であり, チームから変更する必要がある.
- reaction, reply, mention, 転送, 全文検索

### Schedule管理
- ユーザーもしくはorgは一つのscheduleが与えられる.
- Eventの作成がユーザー単位, チーム単位, サブチーム単位, タグ単位, またはその複数組み合わせで可能.
- 15分単位の設定が可能.
- Eventには, timezone, locate, is_meeting, is_repeat, repeat_settings(unit(日, 週, 月, 年), valid, expiration_date), attache_files
- イベントの管理が可能で, 何日の何時から何日の何時まであるかを登録可能で重複も認められる.または, 終日などの設定で日付のみも可能.
- 個人ユーザー側の予定とorg内での個人の予定の同期も可能.
- orgのイベントを個人スケジュールに反映させることも可能.
- カテゴリーを定義して, eventに複数紐付け可能.
- イベント発火設定の定義が可能.これを利用することでAutoFlowでのメール送信サービスなどとの連携が可能.
  - 下記の定義により, 取得できる変数に代入される.
    - beforeでの指定が可能なため, AutoFlowのCronで行うよりも推奨.
    ``` txt
    filter
    before: 何日前, 何時間前, 何分前など
    level
    ```
- Eventに対して, 一つのコメントスレッドが作成されているので, やり取りが可能.
- comment
``` txt
task_id
user_id
content
sent_at
```

### Book管理
- Bookという最も大きな単位の中に階層構造のPageを作成できるが, ディレクトリとファイルといった形式ではなく, トップのPageに対して, 複数の子Pageが存在して, さらにそのPageに子Pageが存在するといった形式である.この際, 親ページには, 小ページへのリンク集が自動的に作成される.
- Bookはユーザー個人, チーム, org単位で作成可能で, ユーザー個人, チーム, orgは複数のBookを保持できる.
- is_publicとすることでorg, teamを越えたユーザーや, 自分以外のユーザーからも閲覧できるBookとなる.
- Bookは, 作成, 削除, 編集が可能で, Pageの追加, 削除, 編集も自由に行うことができる.
- Bookごとに以下のイベント履歴が担当ユーザーとタイムスタンプと共に残ることになる.
  - Bookの作成
  - Bookの情報更新
  - Pageの追加
  - Pageの削除
  - Pageの編集
- 全文検索が可能.

### AutoFlow
- user, organization, teamに所属する.
- トリガーを機に様々なフローを動かすことができる.
- トリガーには以下の3種類が存在する.
``` txt
cron: cronJobでの実行.
manual: 手動で実行.
action: 定義されたアクションをトリガーに実行.
```
- 処理の長さに限りはなく, 実行可能で並行処理も可能.
- ブロックで組んでいき, コンテキストや前ブロックの返却値, 定義されたビルトイン関数などの利用も可能.

### Role
#### Pipe
- orgやuserを越えたサービス同士の連携のためのロールシステム.
- 自身の付与されている全てのロールの中のstatementを組み合わせて, 新たなPipeRoleを個人の所有物として生成できる.
- ただし, 依存先のstatementが更新, 削除された場合は, is_deprecatedとなり, 利用不可能となる.

### Mail送信
### 外部SNS管理
### 通話
### プラグイン管理

## フロントインタフェース

また, フロントエンドのインタフェースとしては, Web, 管理者用Web, 管理者用CLIがあり, 管理者用CLIからは`gRPC`での通信, それ以外からは`REST API`となっているが, 今後さらにNativeApp(iPhone, Android)やDesktopApp(Windows, Mac, Linux)の他にWebHook, PublicAPIの提供も考えている.

## 後の実装予定
- Plan管理における以下の機能
  - 繰り返し
- Storage管理における以下の機能
  - バケットごとに任意でメタデータの追加とそれによるフィルタリング