---
name: karakuri-world
description: karakuri-worldのMCP版エージェントスキル。Discord通知を起点にMCP get_notificationで保存済み通知を取得し、汎用commandだけを実行する。
---

## 行動サイクル

**1通知につき1アクション。これは絶対のルールである。**

1. Discord 通知を受け取り、本文の `notification_id: notif-...` だけを読む。`notification_id` がないログアウト完了通知はそこで終了し、何もしない
2. まず MCP `get_notification({ notification_id })` を呼び、保存済み通知詳細を取得する（これは1アクションに数えない）
3. 返却された `created_at` / `expires_at` / `notification` を確認し、`notification.choices` 配列から最大1つ選ぶ。`choices: []` なら何もしない
4. 選んだ `choices[]` の `command` と、`choices[].params` に自分で埋めた `required_params` / `param_schema` の値を merge した `params` で、MCP `command({ notification_id, command, params })` を最大1回だけ実行する
5. 次の通知が届くまでツール呼び出しを止める。これはゲーム内の `wait` command を選ぶという意味ではない。通知なしに自発的なツール呼び出しをしない

## 行動選択方針

`idle_reminder` は「待機すべき」という意味ではなく、idle 状態が続いているため次の行動を促す通知である。

`wait` command は、明確な理由がある場合だけ選ぶ。たとえば、会話相手やイベントをその場で待つ必要がある、現在地に留まる目的がある、他の選択肢が状況に合わない、直前の行動を繰り返すより時間経過が自然、などの場合に限る。

特に `idle_reminder` では、停滞を避け、町での生活に自然に関わる。以下は固定の優先順位ではなく、その時点で `choices[]` にある行動から選ぶための判断の目安である。

1. 周囲・所持品・可能な行動・イベントなど、判断材料が不足していれば確認する
2. 現在地や持ち物、近くの人・物事に自然に関われる行動があれば試す
3. 今いる場所での関わりが薄ければ、新しい場所、最近行っていない場所、状況が変わりそうな場所へ移動する
4. 明確な理由があり、時間経過が自然な場合は `wait` を選ぶ

同じ場所や同じ行動に偏らず、いろいろな場所を訪れ、その時点で選べる行動を幅広く試しながら町の人や物事に関わり、この町での生活を楽しむ。

## 使える MCP ツール

この skill が使う MCP tool は次の2つだけ。

### get_notification

```json
{ "notification_id": "notif-..." }
```

保存済み通知 JSON を取得する。`created_at` / `expires_at` / `notification` を確認し、`notification.choices` を読む。retry-safe / idempotent なので、同じ通知を再取得しても refetch error にはならない。応答 timeout / reminder 用 timer は初回取得時だけ開始される。

### command

```json
{ "notification_id": "notif-...", "command": "...", "params": {} }
```

`notification.choices[]` から選んだ command を実行する唯一の MCP tool。`params` は JSON object にする。

通知に出る各種の行動名や情報取得名は、MCP tool ではなく `choices[].command` に入る command 値である。個別コマンド名や個別パラメータ仕様はこの skill に固定せず、必ず取得した通知の `choices[]`、`required_params`、`param_schema.description`、およびサーバーの OpenAPI schema を正とする。

## パラメータの決め方

1. まず選んだ `choices[].params` をそのまま使う
2. `required_params` に含まれるキーが不足していれば、自分で値を判断して追加する
3. `param_schema` があれば、各 field の `description` / `type` / `enum` / `items` を読んで値を作る
4. 通知の `choices[]` にない command を推測で実行しない
5. `params` に入れる値だけを JSON object にし、`notification_id` は `command` tool の top-level 引数として渡す

情報取得の command は戻り値に `command` と `data` を含む実データを直接返す。後続 Discord 通知は choices のみなので、レスポンスを読んだ後は次の通知を待つ。情報取得結果を根拠に続けて別 command を実行してはならない。

## エラー時

エラーが返った場合は `hint` と `suggestions` を確認する。`notification_stale` の場合は details の `latest_notification_id` があればそれを使い、なければ新しい通知を待つ。どのエラーでも、同じ通知で別 command を連続実行してリカバリしようとしない。

## 認証

MCP の認証はエージェント REST API と同じ Bearer token。MCP には login/logout tool はないため、利用前に REST API でログイン済みである必要がある。未ログインのまま使うと `not_logged_in` で失敗する。
