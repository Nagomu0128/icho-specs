# 09 KVキャッシュと冪等運用詳細

このドキュメントは、KVの用途制限と無効化戦略、冪等データ運用を実装可能な粒度で定義する。

依存: `07-participant-api-implementation-details.md`, `08-operator-auth-and-api-implementation-details.md`

## 1. KV用途（再確認）

許可:

- ダッシュボード一覧キャッシュ
- グループ詳細キャッシュ
- 固定設定値（checkpoint設定）
- 運営セッションキャッシュ

禁止:

- 進捗の正データ
- 正答判定の根拠
- idempotency管理

## 2. キー体系

- `dash:version:v1`
- `dash:list:v2:{version}:{cursor}:{limit}`
- `dash:group:v2:{groupId}:{version}`
- `cfg:checkpoint:v1:{checkpointCode}`
- `op:session:cache:v1:{sessionId}`

## 3. TTL

- `dash:list`: 15秒
- `dash:group`: 10秒
- `cfg:checkpoint`: 10分
- `op:session:cache`: 5分

## 4. 世代管理

以下イベントで `dash:version:v1` をインクリメント:

- 参加者進捗更新
- 運営のstatus-correction
- mark-reported

更新失敗時（D1 commit済み）:

- 3回まで短時間リトライ
- 失敗してもAPIは成功返却
- エラーログを残す

## 5. idempotency_keys運用

保存値:

- `idempotency_key`
- `group_id`
- `api_name`
- `response_json`
- `status_code`
- `expires_at`（30日）

## 6. 同時到着時の扱い

同一key同時到着時に一部 `409 CONFLICT_STATE` になるケースを許容する（`tech-specs.md` 準拠）。

クライアント再送ルール:

1. `GET /api/v1/progress` で再同期
2. 同一操作を再送

## 7. 日次クリーンアップ

D1削除対象:

- `idempotency_keys`
- `attempt_logs`
- `hint_logs`
- `user_progress_logs`
- `operator_actions`
- `operator_session_events`

実行:

- Cloudflare Cron Trigger で日次実行
- `created_at` または `expires_at` が30日を超えた行を削除
- 失敗時は次回再試行
- 削除ジョブ失敗時は失敗内容を運用ログへ記録（通知は行わない）

## 8. 監視すべきメトリクス

- KV hit ratio（dashboard）
- KV再構築回数
- idempotency hit rate
- `CONFLICT_STATE` 発生率
