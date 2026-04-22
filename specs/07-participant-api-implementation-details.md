# 07 参加者API実装詳細

このドキュメントは、参加者向けAPIをRoute/UseCase/Repositoryに分けて実装するための仕様を定義する。

依存: `06-participant-domain-state-transition-spec.md`

## 1. 対象エンドポイント

- `GET /api/v1/session/start/:groupId`
- `POST /api/v1/q1/:subQuestion/answer`
- `POST /api/v1/q1/:subQuestion/checkpoint`
- `POST /api/v1/q2/answer`
- `POST /api/v1/q2/checkpoint`
- `POST /api/v1/q3/keyword`
- `POST /api/v1/q3/code`
- `POST /api/v1/q4/answer`
- `GET /api/v1/progress`
- `POST /api/v1/complete/epilogue-viewed`

## 2. Route実装配置

`app/routes/api.v1.*` に配置し、ルーティングと入力検証だけを担当する。

入力検証は `zod` で行い、UseCaseへは正規化済DTOのみ渡す。

## 3. UseCase分割

`src/application/usecases/participant/`:

- `start-session.usecase.ts`
- `submit-answer.usecase.ts`（stage別内部分岐）
- `confirm-checkpoint.usecase.ts`
- `get-progress.usecase.ts`
- `mark-epilogue-viewed.usecase.ts`

## 4. 冪等制御

`groupId` を伴う更新系APIでは `X-Idempotency-Key` 必須。

欠落時:

- HTTP 400
- `BAD_REQUEST`

同一キー既処理:

- 保存済み `response_json`, `status_code` をそのまま返す

## 5. 競合制御

`users.state_version` 条件付き更新で0件更新なら競合とみなし:

- HTTP 409
- `CONFLICT_STATE`

クライアントは `GET /api/v1/progress` で再取得後に再送。

## 6. ログ書き込み

更新成功時は必ず以下を同一トランザクション内で書く。

- `user_progress_logs`
- 必要に応じ `attempt_logs`
- `idempotency_keys`

## 7. KV連携

更新成功後:

- `dash:version:v1` をインクリメント
- 失敗時は短時間リトライ後に継続（D1 commitを取り消さない）

## 8. 画面ルートとの接続

`GET /start/:groupId`, `GET /q*` 画面は `loader` で `GET /api/v1/progress` 相当の状態を参照し、未解放画面は `CONFLICT_STATE` を返す。

## 9. テスト観点（integration）

- 同一Idempotency-Key二重送信で同一応答
- 競合更新で片方が `CONFLICT_STATE`
- Q1の順序固定がAPI経由でも保持される
- `epilogue-viewed` が `FAKE_END` 前に拒否される
