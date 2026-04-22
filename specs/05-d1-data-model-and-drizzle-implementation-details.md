# 05 D1データモデルとDrizzle実装詳細

このドキュメントは、`tech-specs.md` のDDLを実装用に具体化する。

依存: `04-application-skeleton-and-dependency-injection.md`

## 1. 方針

- D1が正データ
- すべての更新系APIはD1トランザクションで処理
- `users.state_version` を楽観ロックとして使用
- `idempotency_keys` を30日保持

## 2. Drizzle schema ファイル分割

```text
db/schema/
  users.ts
  logs.ts
  operator-auth.ts
  idempotency.ts
  index.ts
```

## 3. 型定義（重要カラム）

### users

- `groupId: text primary key`
- 形式: `g_` + `UUIDv4`（小文字ハイフン区切り）
- `currentStage: text`（enum相当）
- `stateVersion: integer not null default 0`
- `q1Order: text nullable`
- `currentUnlockedSubquestion: text nullable`
- 各フラグ: integer(0/1)
- `startedAt`, `completedAt`, `reportedAt`, `epilogueViewedAt`
- `createdAt`, `updatedAt`

### idempotencyKeys

- PK: `(groupId, apiName, idempotencyKey)`
- `responseJson`（初回レスポンスを保存）
- `statusCode`
- `expiresAt`

### operatorSessions

- `sessionId: text pk`
- `operatorId: text`
- `expiresAt: text`
- `revokedAt: text nullable`
- `createdAt: text`
- 単一運営アカウント制約として `operatorId = operator` を常に維持

## 4. enum値固定

実装内の定数として固定する。

- `CurrentStage`
  - `START`, `Q1`, `Q2`, `Q3_KEYWORD`, `Q3_CODE`, `Q4`, `FAKE_END`, `COMPLETE`
- `Q1Order`
  - `Q1_1_FIRST`, `Q1_2_FIRST`
- `SubQuestion`
  - `Q1_1`, `Q1_2`

## 5. Repository実装要件

`app/modules/*/infrastructure/` に以下を作成（責務ごとに配置）。

- `app/modules/progress/infrastructure/progress-repository.d1.server.ts`
- `app/modules/progress/infrastructure/idempotency-repository.d1.server.ts`
- `app/modules/operator-session/infrastructure/operator-session-repository.d1.server.ts`
- `app/modules/operator-session/infrastructure/operator-credential-repository.d1.server.ts`
- `app/modules/progress/infrastructure/progress-transaction.d1.server.ts`

## 6. トランザクションの実装規約

更新系UseCaseで共通処理を使う。

1. `idempotency_keys` 参照（同一キーがあれば即返却）
2. `users` を取得
3. Domainで遷移検証
4. `WHERE state_version = :current` で更新
5. 成功時にログテーブルと `idempotency_keys` へINSERT
6. commit後にKV世代インクリメント

## 7. SQLインデックス方針

`tech-specs.md` で定義済みのindexをそのまま適用し、省略しない。

特に重要:

- `idx_users_updated_at`
- `idx_operator_sessions_expires`
- `idx_idempotency_expires`

## 8. マイグレーション運用

- 初期migration: `0001_initial.sql`
- 追加変更: 連番で追加、既存ファイルを書き換えない
- schema変更時は必ず `drizzle-kit` で差分生成

## 9. seed方針

初期seed:

- `operator_credentials` に `operator` の1行を投入
- ローカル開発向けテスト `users` を任意投入

seed実行は自動化せず、`prod` で手動承認のうえ実行する。

運営認証のドメイン制約:

- `operator_sessions.operator_id` と `operator_session_events.operator_id` は常に `operator` を記録する
