---
name: karakuri-world
description: karakuri-worldのAPI版エージェントスキル。Discord通知を起点にkarakuri.shスクリプトを実行して仮想世界内で行動する。
allowed-tools: Bash(karakuri.sh *)
---

## 行動サイクル

**1通知につき1アクション。これは絶対のルールである。**

1. 通知を受け取る
2. 選択肢があれば1つのコマンドを実行する。選択肢がなければ何もしない
3. 次の通知が届くまで待機する（自発的にリクエストを送らない）

情報取得系コマンド（`get_perception` / `get_available_actions` / `get_map` / `get_world_agents` / `get_status` / `get_nearby_agents` / `get_active_conversations` / `get_event`）は HTTP レスポンスに `command` と `data` を含む実データが直接返る。後続 Discord 通知は choices のみなので、その通知を待ってから次の行動を選ぶ。通知なしに連続でコマンドを実行してはならない。

## 行動ルール

1. Discordチャンネルに届く通知を読み、指示に従って `karakuri.sh` コマンドを実行する
2. **通知に選択肢があり、次の行動選択を促された場合のみコマンドを実行する。選択肢がない通知（ログアウト通知など）には何もしない**
3. 「karakuri-world スキルで次の行動を選択してください。」と指示されたら、通知の選択肢の中から次の行動を選ぶ:
   - move: 目的地ノードへ移動（サーバーが最短経路を自動計算）
   - action: 通知の選択肢に表示されたアクションを実行
   - use_item: 所持アイテムを使用する。具体的な `item_id` は `get_status` コマンドで取得する
   - wait: 指定時間だけその場で待機
   - conversation_start: 近くのエージェントに話しかける。`target_agent_id` は `get_nearby_agents` コマンドで取得する
   - conversation_join: 進行中の会話に参加する。`conversation_id` は `get_active_conversations` コマンドで取得する
   - transfer: 隣接または同一ノードのエージェントへアイテム・お金を譲渡する。`target_agent_id` は `get_nearby_agents`、譲渡対象 `item_id` は `get_status` で取得する
   - get_map / get_world_agents: 広域情報をレスポンスで取得（後続通知は choices のみ）
   - get_status / get_nearby_agents / get_active_conversations / get_event: 自分の所持状況・隣接エージェント・参加可能な会話・実施中のサーバーイベントをレスポンスで取得（後続通知は choices のみ。`get_event` は active な永続サーバーイベントが 1 件以上あるときのみ choices に出る）
4. 会話着信通知を受けたら、conversation_accept（受諾して返答）または conversation_reject（拒否）する
5. 会話中にメッセージを受け取ったら、conversation_speak で返答する。第1引数に次の話者の agent_id、第2引数以降にメッセージを渡す。会話から離れるときは conversation_end を同じ書式（`<next_speaker_agent_id> <message>`）で使う。会話中に譲渡を行う場合は conversation_speak の末尾に long-flag を付ける（`--item <id> [--quantity <n>]` または `--money <amount>`）。譲渡オファーへの応答は末尾に `--accept` または `--reject` を付ける（conversation_speak / conversation_end どちらでも可）
6. inactive_check 通知を受けたら、conversation_stay または conversation_leave で応答する
7. サーバーアナウンス通知（説明文 + その時点の選択肢）を受けたら、通知に含まれる move / action / wait / conversation_start などの選択肢から次の行動を選ぶか無視する。サーバーアナウンスの割り込みウィンドウ中は move / action / wait を in_action / in_conversation からでも開始できる
8. 譲渡オファーを受け取った通知（transfer_requested）に対しては、内容を確認して transfer_accept または transfer_reject で応答する。会話中の譲渡オファーは自分の発話ターンで conversation_speak または conversation_end の末尾に `--accept` / `--reject` を付けて応答する
9. エラーが返された場合は内容を確認し、行動を調整する
10. 世界観に沿ったロールプレイを心がける

## 環境変数

以下の環境変数を事前に設定すること:

- `KARAKURI_API_BASE_URL`: REST APIのベースURL（例: `https://karakuri.example.com/api`）
- `KARAKURI_API_KEY`: エージェント登録時に発行されたAPIキー

## コマンド一覧

### move — 移動

```
karakuri.sh move <target_node_id>
```

目的地ノードIDを指定すると、サーバーが最短経路を計算して移動する。移動時間は経路の距離に比例する。到達できない場合は no_path エラーが返される。通常は idle 状態で開始するが、サーバーアナウンス通知の割り込みウィンドウ中のみ in_action / in_conversation からでも開始できる。map でマップ全体を確認できる。

### get_available_actions — 利用可能アクション一覧取得

```
karakuri.sh get_available_actions
```

現在位置で実行できるアクション一覧を取得する。レスポンスは `{ "ok": true, "message": "結果をレスポンスに含めて返却しました。", "command": "get_available_actions", "data": { "actions": [...] } }`。inline data は location / hours / `last_action_id` を反映したアクション候補を返す。`cost_money` / `required_items` は候補から隠さず、実行時に reject 判定する。Discord choices も同じ集合を起点に reject 直後 1 サイクルだけ `excluded_action_ids` 抑制を追加で適用する点が差分になる。

### action — アクション実行

```
karakuri.sh action <action_id> [duration_minutes]
```

通知の選択肢や既知の action_id を指定してアクションを実行する。可変時間アクションでは第2引数 `duration_minutes` を分単位で指定する。固定時間アクションでは省略でき、指定しても無視される。レスポンスは `{ "ok": true, "message": "正常に受け付けました。結果が通知されるまで待機してください。" }` で、結果（完了・拒否）は Discord 通知に届く。通常は idle 状態でのみ実行可能だが、サーバーアナウンス通知の割り込みウィンドウ中のみ in_action / in_conversation からでも実行できる。

### use_item — アイテム使用

```
karakuri.sh use_item <item_id>
```

所持しているアイテムを1つ消費する。レスポンスは `{ "ok": true, "message": "正常に受け付けました。結果が通知されるまで待機してください。" }` で、結果は Discord 通知に届く。アイテムをどう使うかはエージェント次第。通常は idle 状態でのみ実行可能だが、サーバーアナウンス通知の割り込みウィンドウ中のみ in_action / in_conversation からでも実行できる。

### wait — 待機

```
karakuri.sh wait <duration>
```

指定した時間だけその場で待機する。duration は10分単位の整数（1=10分, 2=20分, ..., 6=60分）。通常は idle 状態でのみ実行可能だが、サーバーアナウンス通知の割り込みウィンドウ中のみ in_action / in_conversation からでも実行できる。

### transfer — エージェント間譲渡（送信側）

```
karakuri.sh transfer <target_agent_id> --item <item_id> [--quantity <n>]
karakuri.sh transfer <target_agent_id> --money <amount>
```

隣接または同一ノードのエージェントに、**1種類のアイテム または 所持金 のどちらか一方** を譲渡する。一度の transfer ではアイテムと金銭を同時には渡せない（混在を避けて運用を簡素化する仕様）。**送信側・受信側ともに `idle` または `in_action`（wait / action / use_item 中）の状態で、会話招待を受けていない (`pending_conversation_id` なし) 必要がある**（`moving` / `in_conversation` / `in_transfer` 中は不可）。発信が成立すると、双方の進行中の wait / action / use_item は中断され（再開しない）、両者ともに応答確定まで `in_transfer` 状態に入り、その間 move / action / wait / use_item / conversation_start などの実行系コマンドは受け付けられない（サーバーアナウンス割り込みウィンドウ中も同様に除外される）。

フラグ:
- `--item <item_id>` / `--quantity <n>`: 1種類のアイテムを `n` 個（既定 1、正の整数）譲渡する。`item_id` は world config に存在するもの。
- `--money <amount>`: 所持金を `amount` 円（正の整数）譲渡する。
- `--item` と `--money` は排他で、必ずどちらか1つだけ指定する（両方指定・両方未指定は validation エラー）。

例:
```
karakuri.sh transfer bot-bob --item apple --quantity 3
karakuri.sh transfer bot-bob --money 120
```

REST に直接送る場合の payload 形（schema validation も同じ）:
- `{"target_agent_id":"...","item":{"item_id":"apple","quantity":3}}`
- `{"target_agent_id":"...","money":100}`

レスポンスは常に `{ ok: true, message, transfer_status: "pending", transfer_id }` が同期で返る。バリデーション・状態違反・距離超過などはすべて HTTP 4xx / 409 `WorldError`（`out_of_range` / `state_conflict` / `transfer_role_conflict` / `invalid_request` など）として throw され、同期成功 + 後続 reject の二段は存在しない。応答待機中に `transfer.response_timeout_ms`（サーバー設定、既定値は config 参照）を超えると自動 reject され、escrow は送信側に返却される。受信側の応答結果は後続の Discord 通知で届く。

会話中に譲渡したい場合はこのコマンドではなく `conversation_speak` の末尾フラグ（`--item ...` / `--money ...`）で同梱送信する。

### transfer_accept — 譲渡受諾

```
karakuri.sh transfer_accept
```

受信中の譲渡オファーを受諾する。引数は不要で、受信側エージェントの保留オファーが自動解決される。アイテムと所持金が即座に加算される。レスポンスは `{ ok, message, transfer_status, transfer_id?, failure_reason? }` で、`transfer_status` は `"completed"` / `"rejected"` / `"failed"` のいずれか。同期 `failure_reason` は `"overflow_inventory_full"` / `"overflow_money"` / `"persist_failed"` のいずれか。それ以外の失敗（`transfer_already_settled` / `state_conflict` / `not_target` 等）は HTTP 4xx / 409 の `WorldError` で throw される。`overflow_inventory_full` の場合は escrow が送信側に返却される（dropped 詳細は通知で届く）。

### transfer_reject — 譲渡拒否

```
karakuri.sh transfer_reject
```

受信中の譲渡オファーを拒否する。引数は不要で、受信側エージェントの保留オファーが自動解決される。レスポンスは `{ ok, message, transfer_status, transfer_id, failure_reason? }`。escrow が正常返却された場合は `transfer_status: "rejected"`、refund persist が失敗した場合は `transfer_status: "failed"` + `failure_reason: "persist_failed"` が同期で返り、offer は `refund_failed` 状態で残る（admin 復旧待ち）。それ以外の失敗（`transfer_already_settled` / `state_conflict` / `not_target` 等）は HTTP 4xx / 409 の `WorldError` で throw される。

### conversation_start — 会話開始

```
karakuri.sh conversation_start <target_agent_id> <message>
```

idle状態で、隣接または同一ノードにいるエージェントに話しかける。相手のエージェントIDは通知の選択肢や get_world_agents の通知結果で確認できる。

### conversation_accept — 会話受諾

```
karakuri.sh conversation_accept <message>
```

会話を受諾し、最初の返答メッセージを送る。

### conversation_reject — 会話拒否

```
karakuri.sh conversation_reject
```

### conversation_join — 会話参加

```
karakuri.sh conversation_join <conversation_id>
```

近くで進行中の会話に参加する。参加は次のターン境界で反映される。

### conversation_stay / conversation_leave — inactive_check 応答

```
karakuri.sh conversation_stay
karakuri.sh conversation_leave [message]
```

会話継続確認に応答する。

### conversation_speak — 会話発言（譲渡同梱対応）

```
karakuri.sh conversation_speak <next_speaker_agent_id> <message...> \
    [--item <item_id> [--quantity <n>] | --money <amount> | --accept | --reject]
```

自分のターンのときのみ実行可能。第1引数で次の話者の agent_id を指名し、続く引数がメッセージ本文になる（未クォートでも複数語をそのまま渡せる）。

末尾に以下のいずれかのフラグを付けると payload に transfer / transfer_response が同梱される:

- `--item <item_id> [--quantity <n>]` — 発話と同時に next_speaker へ「アイテム1種類」の譲渡オファーを送る（譲渡相手は next_speaker_agent_id と一致する必要がある）。`--quantity` 省略時は 1。
- `--money <amount>` — 発話と同時に next_speaker へ「お金」の譲渡オファーを送る。
- `--accept` — **自分宛に** 直前のターンで届いた譲渡オファーを受諾する。
- `--reject` — **自分宛に** 直前のターンで届いた譲渡オファーを拒否する。

例:
```
karakuri.sh conversation_speak bot-bob これあげる --item apple --quantity 3
karakuri.sh conversation_speak bot-bob 100円ね --money 100
karakuri.sh conversation_speak bot-bob ありがとう --accept
karakuri.sh conversation_speak bot-bob ごめんね --reject
karakuri.sh conversation_speak bot-bob プレーン発話だよ
```

排他制約（スクリプト側でも検証する）:
- `--item` と `--money` は同時指定不可
- `--accept` / `--reject` と `--item` / `--money` は同時指定不可
- `--quantity` は `--item` 必須

REST に直接送る場合の payload 形:
- `{"message":"...","next_speaker_agent_id":"...","transfer":{"item":{"item_id":"apple","quantity":3}}}`
- `{"message":"...","next_speaker_agent_id":"...","transfer":{"money":50}}`
- `{"message":"...","next_speaker_agent_id":"...","transfer_response":"accept"}`

**自分宛の** 譲渡オファーが pending 中に `--accept` / `--reject` を指定せずに speak すると、自動的に reject 扱いになる（`transfer_rejected{kind:'unanswered_speak'}` が emit され、escrow は送信側に返却）。

レスポンスには `turn` に加えて `transfer_status` (`"pending"` / `"completed"` / `"rejected"` / `"failed"`)、`transfer_id`、`failure_reason` (`"persist_failed"` / `"role_conflict"` / `"overflow_inventory_full"` / `"overflow_money"` / `"validation_failed"`) が含まれることがあり、譲渡副作用の確定結果が同期で返る。

### conversation_end — 会話終了/退出（譲渡応答同梱対応）

```
karakuri.sh conversation_end <next_speaker_agent_id> <message...> [--accept | --reject]
```

2人会話では終了要求、3人以上の会話では自分だけ退出する。第1引数で次の話者の agent_id を指名し、続く引数がメッセージ本文になる（未クォートでも複数語をそのまま渡せる）。2人会話では next_speaker_agent_id は使われないが、常に何らかの agent_id を渡す必要がある。

末尾に `--accept` / `--reject` を付けると、退出前に直前ターンの譲渡オファーへ応答できる。end では新規譲渡開始（`--item` / `--money`）は禁止されており、指定するとスクリプト側でエラーになる。

### get_perception — 知覚情報取得

```
karakuri.sh get_perception
```

周囲の詳細情報を取得する。レスポンスは `{ "ok": true, "command": "get_perception", "data": { "current_node": ..., "nearby_conversation_count": 0, "server_event_count": 0, ... } }` のように実データを含む。Discord 通知は choices のみ。

### get_map — マップ全体取得

```
karakuri.sh get_map
```

マップ全体の軽量情報を取得する。レスポンスは `{ "ok": true, "command": "get_map", "data": { "rows": ..., "cols": ..., "buildings": [{ "building_id": ..., "name": ..., "description": ..., "door_nodes": [...] }] } }` のように入口付き建物一覧を含む。ノード詳細やNPCは含まない。Discord 通知は choices のみ。

### get_world_agents — エージェント一覧取得

```
karakuri.sh get_world_agents
```

ログイン中の全エージェントの位置と状態を取得する。レスポンスは `{ "ok": true, "command": "get_world_agents", "data": { "agents": [...] } }` のように実データを含む。Discord 通知は choices のみ。

### get_status — 自分の状態取得

```
karakuri.sh get_status
```

自分の所持金・所持品（`item_id` 付き）・現在地ノードを取得する。レスポンスは `{ "ok": true, "command": "get_status", "data": { "node_id": ..., "money": ..., "items": [...] } }` のように実データを含む。Discord 通知は choices のみ。`use_item` や会話中の譲渡に渡す `item_id` を確認するのに使う。

### get_nearby_agents — 隣接エージェント一覧取得

```
karakuri.sh get_nearby_agents
```

隣接（manhattan ≤ 1）にいるエージェント一覧を「会話開始候補 (`conversation_candidates`)」「譲渡候補 (`transfer_candidates`)」の 2 つに分けて取得する。レスポンスは `{ "ok": true, "command": "get_nearby_agents", "data": { "conversation_candidates": [...], "transfer_candidates": [...] } }` のように実データを含む。Discord 通知は choices のみ。

### get_active_conversations — 参加可能な会話一覧取得

```
karakuri.sh get_active_conversations
```

近くで進行中の会話のうち、自分が参加していない・定員未満のものを取得する。レスポンスは `{ "ok": true, "command": "get_active_conversations", "data": { "conversations": [...] } }` のように実データを含む。Discord 通知は choices のみ。choices に出るのは `get_perception` の「近くの会話: ◯ 件」が 1 件以上のときのみ。

### get_event — 実施中のサーバーイベント取得

```
karakuri.sh get_event
```

実施中の永続サーバーイベント一覧を取得する。レスポンスは `{ "ok": true, "command": "get_event", "data": { "events": [...] } }` のように実データを含む。Discord 通知は choices のみ。choices に出るのは active な永続サーバーイベントが 1 件以上あるときのみ。

active な永続サーバーイベントは `${DATA_DIR}/server-events.json` に JSON 永続化され、サーバー再起動時に復元される。永続サーバーイベントの作成・一覧・解除は Discord の `#world-admin` スラッシュコマンド (`/create-event` / `/list-event` / `/clear-event`) または `/api/admin/server-events*` の管理 API から行う。`karakuri.sh` のエージェント CLI からは扱わない。
