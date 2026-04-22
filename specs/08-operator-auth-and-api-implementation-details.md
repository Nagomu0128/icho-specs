# 08 運営認証と運営API実装詳細

このドキュメントは、運営ログイン・セッション管理・ダッシュボードAPIを実装するための仕様を定義する。

依存: `05-d1-data-model-and-drizzle-implementation-details.md`

## 1. 対象エンドポイント

- `POST /api/v1/operator/login`
- `POST /api/v1/operator/logout`
- `GET /api/v1/operator/dashboard`
- `GET /api/v1/operator/group/:groupId`
- `POST /api/v1/operator/group/:groupId/status-correction`
- `POST /api/v1/operator/group/:groupId/mark-reported`

UseCase配置方針:
- 進捗更新系（`status-correction`, `mark-reported`）は `app/modules/progress/application/usecases/` に配置する
- セッション系（`login`, `logout`, session検証）は `app/modules/operator-session/application/usecases/` に配置する
- actor名（operator）をディレクトリ分割の軸にしない。認可は各usecase内でRBACポリシー評価する

認可モデル:
- roleは `participant` / `operator` の2値を採用する
- 運営APIは `operator` role を必須とし、未充足時は `FORBIDDEN`（403）を返す

## 2. PBKDF2検証

- アルゴリズム: `PBKDF2-SHA256`
- salt: 16byte以上
- iteration: 210000（初期）
- key length: 32byte
- 比較は一定時間比較を使う

資格情報は `operator_credentials`（`operator_id = operator`）の1行を使用。

## 3. セッション発行

ログイン成功時:

1. 署名付きランダム `session_id` を生成
2. D1 `operator_sessions` へ保存
3. KV `op:session:cache:v1:{sessionId}` へキャッシュ（TTL 5分）
4. Cookie `operator_session` を発行
5. D1 `operator_session_events` に `LOGIN_SUCCESS`

Cookie属性:

- `HttpOnly`
- `Secure`
- `SameSite=Strict`
- `Path=/`
- 有効期限: 12時間

## 4. セッション検証

順序固定:

1. KV参照
2. KVミスまたは不整合時にD1参照してキャッシュ再構築
3. `revoked_at` または `expires_at` なら拒否（401）

`CACHE_MISS` 発生時は `operator_session_events` に記録。

## 5. ログアウト

1. D1 `operator_sessions.revoked_at` 更新
2. KVセッションキー削除
3. `operator_session_events` に `LOGOUT`
4. Cookie削除

## 6. 運営更新APIの整合性

以下は更新系なので `X-Idempotency-Key` 必須:

- `status-correction`
- `mark-reported`

さらに `users.state_version` 条件付き更新を必須化し、競合時は `409 CONFLICT_STATE`。
同一キー同時到着の競合で `409 CONFLICT_STATE` が発生する場合を許容し、クライアントは最新進捗再取得後に同一操作を再送する。
`POST /api/v1/operator/login` と `POST /api/v1/operator/logout` は本ルールの対象外。

## 7. ダッシュボード一覧

`GET /api/v1/operator/dashboard?cursor=...&limit=50`

- KVキー: `dash:list:v2:{version}:{cursor}:{limit}`
- ミス時D1で取得しKVへ再構築
- `nextCursor` を返す
- レスポンス要素は `groupId`, `currentStage`, `attemptCountTotal`, `hintCountTotal`, `updatedAt` を含む

## 8. グループ詳細/補正APIのI/O

`GET /api/v1/operator/group/:groupId`:

- `groupId`, `currentStage`, `stateVersion`, 回答/チェックポイント進捗、監査表示用の最新ログ要約を返す

`POST /api/v1/operator/group/:groupId/status-correction`:

- リクエスト: `fromStage`, `toStage`, `reasonCode`, `note`
- レスポンス: `updated`, `currentStage`
- 成功時に `operator_actions` へ監査ログを書き込む

`POST /api/v1/operator/group/:groupId/mark-reported`:

- リクエスト: `reasonCode`, `note`
- レスポンス: `updated`, `reported`
- 成功時に `operator_actions` へ監査ログを書き込む

## 9. 運営画面ルートガード

- `/operator/*` は認証必須
- 未認証・期限切れ・失効は401
- UI側はログイン画面へ遷移し、再認証後に元画面復帰

## 10. テスト観点

- ログイン成功時にD1/KV/Cookie/Eventが揃う
- 失効済セッションで401
- ログアウト後の再アクセスで401
- 更新系APIの冪等動作
