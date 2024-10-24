# Lab Groupware API設計
> BFF実装はサブドメインによって分けることとし, その内Web APIの記述をする.
## URI設計
- すべて小文字
- 単語はハイフンでつなげる
- リソース名は複数系で使用する
- リソースを特定するためのidなどは`クエリパラメータ`ではなく、`パスパラメータ`を使用で統一
- その他もクエリパラメータかパスパラメータかの使い分けは以下の判断基準で行う
    - 一意なリソースを表すのに必要な場合はパスパラメータ
    - 省略可能な場合はクエリパラメータ
## リクエスト形式
- データ構造
    - jsonでスネーク形式でリクエストを受け付ける
## レスポンス形式
- データ構造
    - jsonでスネーク形式でリクエストを受け付ける
    - リスト取得APIの実施について
    - orderパラメータでソート順を指定(r_*は逆順となる)
    - paginationクエリパラメータでページネーションタイプ(numbered, cursor, none)を指定
    - 番号付きページネーションは`offset`と`limit`クエリパラメータで行う
    - カーソルページネーションは`cursor`と`limit`クエリパラメータで行う
    - with_countで総数を取得するかどうかを指定
    - 日付のフォーマットはRFC3339(W3C-DTF)
    - `2023-09-24T17:00:00+09:00`のような形式
    - 以下のような同一の構造に固定する(この構造はすべてのAPIで共通)
    ``` json
    {
        "success": boolean,
        "data": object | array | null,
        "code": string,
        "caption": string,
        "error_attributes": array | null
    }
    ```
    - ただし, 非同期の場合は以下のような構造を返すこととする.
    ``` json
    {
        "success": boolean, // 非同期処理すべてが完了したかどうか
        "process": boolean, // 非同期処理が実行中かどうか
        "is_valid": boolean, // JobIDが有効かどうか(期限切れでないかどうか)
        "data": object | array | null, // 処理全体として返却されるデータ
        "actions_queue": array, // 非同期処理の未処理アクションコードが配列に格納される(この段階で実行すべきアクションが順番に格納されており, アクション開始と同時にそのアクションが削除される)
        "pending_action_code": string | null, // 非同期処理の実行中アクションコードが格納される(この段階ではnull)
        "actions_stack": array, // 非同期処理の実行済みアクションが格納される(この段階ではnull)
        "code": string, // 非同期処理実行中を示すステータスコード | 非同期処理完了を示すステータスコード | 非同期処理エラーを示すステータスコード
        "caption": string, // 非同期処理実行中を示すキャプション | 非同期処理完了を示すキャプション | 非同期処理エラーを示すキャプション
        "error_attributes": array | null,
        "job_id": string,
        "job_uri": string,
        "progress": number, // 進行状況を示す数値(0-100), 計算方法としては, 処理されるべきアクション数に対して, 実行されたアクション数(data.length)を割合で示す(この段階では0),
        "started_at": string | null, // 非同期処理の開始日時
        "expired_at": string | null // 非同期処理の有効期限を示す日時(通常は24時間後)
    }
    ```
    - 処理の進行状況を確認するためのURIからの返却も上記と同じ構造で返す
    - dataは処理完了時にまとめて格納され, リクエストによって形式は異なる
    - actionのフォーマットは以下のようにする
    ``` json
    {
        "action_code": string, // アクション概要のわかるコード
        "success": boolean, // アクションの成功かどうか
        "data": object | array | null, // アクションデータ
        "code": string, // アクションのステータスコード
        "caption": string, // キャプション
        "error_attributes": array | null // エラーの詳細
    }
    ```
    - 例えば, ユーザーの作成を行う場合, ユーザーの作成アクション途中に, jobURLでの進行確認があった場合, 以下のような構造を返す
    ``` json
    {
        "success": false,
        "process": true,
        "is_valid": true,
        "data": null,
        "actions_queue": ["belong_personal_chat_room", "attach_default_authority"],
        "pending_action": "create_personal_chat_room",
        "actions_stack": [
            {
                "action_code": "create_user",
                "success": true,
                "data": {
                    "user_id": "12345",
                    "name": "blank",
                    "email": "blank@example.com",
                    "created_at": "2023-09-24T17:00:00+09:00",
                    "updated_at": "2023-09-24T17:00:00+09:00"
                },
                "code": "100",
                "caption": "ok",
                "error_attributes": null
            }
        ],
        "code": "801", // 非同期処理中を示すコード
        "caption": "processing asynchronous job",
        "error_attributes": null,
        "job_id": "xxxx-xxxx-xxxx-xxxx",
        "job_uri": "https://job.api.example/process/xxxx-xxxx-xxxx-xxxx",
        "progress": 25,
        "started_at": "2023-09-24T17:00:00+09:00",
        "expired_at": "2023-09-25T17:00:00+09:00"
    }
    ```
    - 例えば, ユーザーの作成を行う場合, ユーザーの作成アクション完了時に, jobURLでの進行確認があった場合, 以下のような構造を返す
    ``` json
    {
        "success": true,
        "process": false,
        "is_valid": true,
        "data": {
            "user_id": "12345",
            "personal_chat_room": {
                "chat_room_id": "aaaaa",
                "name": "blank personal room",
                "is_personal": true,
                "created_at": "2023-09-24T17:00:00+09:00",
                "updated_at": "2023-09-24T17:00:00+09:00"
            },
            "name": "blank",
            "email": "blank@example.com",
            "created_at": "2023-09-24T17:00:00+09:00",
            "updated_at": "2023-09-24T17:00:00+09:00"
        },
        "actions_queue": [],
        "pending_action": null,
        "actions_stack": [
            {
                "action_code": "attach_default_authority",
                "success": true,
                "data": {
                    "user_id": "12345",
                    "authority_id": "xxxxx",
                    "created_at": "2023-09-24T17:00:00+09:01",
                    "updated_at": "2023-09-24T17:00:00+09:01"
                },
                "code": "100",
                "caption": "ok",
                "error_attributes": null
            },
            {
                "action_code": "belong_personal_chat_room",
                "success": true,
                "data": {
                    "user_id": "12345",
                    "chat_room_id": "aaaaa",
                    "created_at": "2023-09-24T17:00:00+09:00",
                    "updated_at": "2023-09-24T17:00:00+09:00"
                },
                "code": "100",
                "caption": "ok",
                "error_attributes": null
            },
            {
                "action_code": "create_personal_chat_room",
                "success": true,
                "data": {
                    "chat_room_id": "aaaaa",
                    "name": "blank personal room",
                    "is_personal": true,
                    "created_at": "2023-09-24T17:00:00+09:00",
                    "updated_at": "2023-09-24T17:00:00+09:00"
                },
                "code": "100",
                "caption": "ok",
                "error_attributes": null
            },
            {
                "action_code": "create_user",
                "success": true,
                "data": {
                    "user_id": "12345",
                    "name": "blank",
                    "email": "blank@example.com",
                    "created_at": "2023-09-24T17:00:00+09:00",
                    "updated_at": "2023-09-24T17:00:00+09:00"
                },
                "code": "100",
                "caption": "ok",
                "error_attributes": null
            }
        ],
        "code": "800", // 非同期処理完了を示すコード
        "caption": "asynchronous job completed",
        "error_attributes": null,
        "job_id": "xxxx-xxxx-xxxx-xxxx",
        "job_uri": "https://job.api.example/process/xxxx-xxxx-xxxx-xxxx",
        "progress": 100,
        "started_at": "2023-09-24T17:00:00+09:00",
        "expired_at": "2023-09-25T17:00:00+09:00"
    }
    ```
    - 例えば, ユーザーの作成を行う場合, ユーザーの作成アクションエラー時に, jobURLでの進行確認があった場合, 以下のような構造を返す
    ``` json
    {
        "success": false,
        "process": false,
        "is_valid": true,
        "data": null,
        "actions_queue": [],
        "pending_action": null,
        "actions_stack": [
            {
                "action_code": "create_user",
                "success": false,
                "data": null,
                "code": "400",
                "caption": "bad request",
                "error_attributes": [
                    {
                        "field": "email",
                        "caption": "email is already taken"
                    }
                ]
            }
        ],
        "code": "802", // 非同期処理エラーを示すコード
        "caption": "asynchronous job error",
        "error_attributes": [
            {
                "action_code": "create_user",
                "data": [
                    {
                        "field": "email",
                        "caption": "email is already taken"
                    }
                ]
            }
        ],
        "job_id": "xxxx-xxxx-xxxx-xxxx",
        "job_uri": "https://job.api.example/process/xxxx-xxxx-xxxx-xxxx",
        "progress": 0,
        "started_at": "2023-09-24T17:00:00+09:00",
        "expired_at": "2023-09-25T17:00:00+09:00"
    }
    ```
    - expired_at以降にjob_uriにアクセスすると, 以下のデータ構造で404(Not Found)を返す
    ``` json
    {
        "success": false,
        "process": false,
        "is_valid": false,
        "data": null,
        "actions_queue": [],
        "pending_action": null,
        "actions_stack": [],
        "code": "804", // JobIDが有効期限切れもしくは存在しないことを示すコード
        "caption": "job not found",
        "error_attributes": null,
        "job_id": "xxxx-xxxx-xxxx-xxxx",
        "job_uri": "https://job.api.example/process/xxxx-xxxx-xxxx-xxxx",
        "progress": 0,
        "started_at": null,
        "expired_at": null
    }
    ```
## ステータスコード
- 2xx
    - 作成成功を示す201(Created)はリソースの作成に成功した場合に返すが, ヘッダーのLocationに新しいリソースへのurlは含めない
    - 更新成功を示す200(OK)はリソースの更新に成功した場合に返す
    - 削除成功を示す204(No Content)は使わず, 200(OK)で返す
    - リソースの取得に成功した場合は200(OK)を返す
    - 非同期ジョブを受け付けたことを示す202(Accepted)を返す場合, 処理の進行状況を確認するためのURIやジョブIDを受け取る.なお, このJobIDはWebsocketなどでの通知に使用する
- 4xx
    - リクエストのエラー
    - 特定できないものは400, それ以外はそれぞれで定義されたエラーを返すが, フロント側はエラーの場合, ステータスコードではなく, body内のアプリケーション用のエラーコードを見るようにする
- 5xx
    - サーバー上のエラーもしくは予期しないエラー
## アプリケーションコード
- フロント側のエラーハンドリング用のアプリケーション独自コード.
- ステータスコードより細かい定義であるため, 別のドキュメントを作成予定.
## アクションコード
- 非同期処理の進行状況を確認する際に使用する各アクションのコード.
- コードは数字文字列ではなく, そのままアクションの内容がわかるような文字列を使用する.
- 例えば, ユーザーの作成を行う場合, `create_user`というアクションコードを使用する.
- [アクションコード](./action_code.json)参照.
## 規定ヘッダー
- `Accept-Language`により言語を指定. 現段階では`ja`のみ対応
- `X-Request-Type`によりリクエストの種類を指定.原産開ではGETリクエストのsync, POST, PUT, DELETEリクエストのasyncのみ対応
    - `sync`は同期リクエスト, `async`は非同期リクエスト
    - GETリクエストは`sync`のみ, POST, PUT, DELETEリクエストは`sync`と`async`の両方を受け付ける
## 認証
- OAuth認証のAuthorization Code Flow