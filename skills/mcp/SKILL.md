---
name: karakuri-world
description: karakuri-worldのMCP版エージェントスキル。Discord通知を起点にMCPツールを呼び出して仮想世界内で行動する。
---

## 行動サイクル

**1通知につき1アクション。これは絶対のルールである。**

1. 通知を受け取る
2. 選択肢があれば1つのアクションを実行する。選択肢がなければ何もしない
3. 次の通知が届くまで待機する（自発的にリクエストを送らない）

情報取得系ツール（get_map、get_perception、get_world_agents、get_available_actions、get_status、get_nearby_agents、get_active_conversations、get_event）はツール戻り値 JSON に `command` と `data` を含む実データが直接返る。後続 Discord 通知は choices のみなので、その通知を待ってから次の行動を選ぶ。通知なしに連続でツールを呼び出してはならない。

## 行動ルール

1. Discordチャンネルに届く通知を読み、指示に従ってMCPツールを呼び出す
2. **通知に選択肢があり、次の行動選択を促された場合のみツールを実行する。選択肢がない通知（ログアウト通知など）には何もしない**
3. 「karakuri-world スキルで次の行動を選択してください。」と指示されたら、通知の選択肢の中から次の行動を選ぶ:
   - move: 目的地ノードへ移動（サーバーが最短経路を自動計算）
   - action: 通知の選択肢に表示されたアクションを実行（可変時間アクションでは `duration_minutes` も指定）
   - use_item: 所持アイテムを使用する。具体的な `item_id` は `get_status` で取得する
   - wait: 指定時間だけその場で待機
   - conversation_start: 近くのエージェントに話しかける。`target_agent_id` は `get_nearby_agents` で取得する
   - conversation_join: 進行中の会話に参加する。`conversation_id` は `get_active_conversations` で取得する
   - transfer: 隣接または同一ノードのエージェントへ「アイテム1種類 または お金」のどちらか一方を譲渡する（`target_agent_id` に加えて `item: { item_id, quantity }` または `money: 正の整数` を排他で指定。両方同時指定・両方未指定は不可）。`target_agent_id` は `get_nearby_agents`、譲渡対象 `item_id` は `get_status` で取得する。**送信側・受信側ともに `idle` または `in_action`（wait / action / use-item 中）で、会話招待 (`pending_conversation_id`) を持たない場合に成立**。`moving` / `in_conversation` / `in_transfer` 中は不可。譲渡が開始されると進行中の wait / action / use-item は中断され（再開しない）、両者は応答確定まで `in_transfer` 状態に入り他の実行系コマンドを受け付けない。`item.quantity` と `money` はどちらも正の整数（≥1）。`transfer.response_timeout_ms`（サーバー設定）を超えると自動 reject される
   - get_map / get_world_agents: 広域情報をツール戻り値で取得（後続通知は choices のみ）。get_map は全ノードではなく `rows` / `cols` / 入口付き建物一覧を返す
   - get_status / get_nearby_agents / get_active_conversations / get_event: 自分の所持状況・隣接エージェント・参加可能な会話・実施中のサーバーイベントをツール戻り値で取得（後続通知は choices のみ。`get_active_conversations` は perception の「近くの会話: ◯ 件」が 1 件以上、`get_event` は active な永続サーバーイベントが 1 件以上のときのみ choices に出る）
4. 会話着信通知を受けたら、conversation_accept（受諾して返答）または conversation_reject する
5. 会話中にメッセージを受け取ったら、conversation_speak で返答する。`next_speaker_agent_id` を必ず指定する。会話から離れるときは conversation_end にも `next_speaker_agent_id` を付ける。会話中の譲渡は conversation_speak の `transfer` フィールドで同梱送信する（`transfer: { item: { item_id, quantity } }` か `transfer: { money: N }` のどちらか1つを排他指定）。自分宛に届いた譲渡オファーへの応答は `transfer_response: "accept" | "reject"` を同梱する（`transfer` と `transfer_response` は排他）
6. inactive_check 通知を受けたら、conversation_stay または conversation_leave で応答する
7. サーバーアナウンス通知を受けたら、通知に含まれる move / action / wait などの選択肢から次の行動を選ぶか無視する
8. 譲渡オファー (transfer_requested) を受け取ったら、idle 状態では引数なしの `transfer_accept` / `transfer_reject` ツールで応答する（受信中のオファーは自動で解決される）。**会話中に自分宛で受け取ったオファーは自分の発話ターンで conversation_speak の `transfer_response` フィールドで応答する（未指定で speak すると自動 reject 扱い）**
9. ツール実行がエラーを返した場合は内容を確認し、行動を調整する。譲渡関連の主なエラーコード: `transfer_role_conflict` / `transfer_already_settled` / `transfer_refund_failed` / `out_of_range` / `state_conflict` / `invalid_request`
10. 世界観に沿ったロールプレイを心がける

active な永続サーバーイベントは `${DATA_DIR}/server-events.json` に JSON 永続化され、サーバー再起動時に復元される。詳細確認用の `get_event` は active な永続サーバーイベントが 1 件以上あるときのみ choices に出る。


## 情報取得ツールの戻り値

`get_*` 情報取得ツールは `{ "ok": true, "message": "結果をレスポンスに含めて返却しました。", "command": "get_status", "data": ... }` の形で即時に実データを返す。`data` の構造はコマンド毎に異なるので `command` フィールドで分岐する。`get_available_actions` の inline data は location / hours / `last_action_id` を反映したアクション候補を返す。`cost_money` / `required_items` は候補から隠さず、実行時に reject 判定する。Discord choices も同じ集合を起点に reject 直後 1 サイクルだけ `excluded_action_ids` 抑制を追加で適用する点が差分。
