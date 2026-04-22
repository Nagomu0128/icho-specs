# 04 アプリケーション骨格と依存注入

このドキュメントは、React Router + Workers で3層クリーンアーキテクチャを実装するための骨格を定義する。

依存: `01-repository-structure-and-implementation-guidelines.md`

## 1. src配下の責務

```text
src/
  domain/
    progress/
      state-machine.ts
      answer-normalizer.ts
      answer-judge.ts
  application/
    ports/
      progress-repository.port.ts
      idempotency-repository.port.ts
      operator-session-repository.port.ts
      clock.port.ts
      id-generator.port.ts
    usecases/
      participant/
      operator/
  infrastructure/
    d1/
    kv/
    services/
```

## 2. Env定義

`workers/bindings/env.ts` に型を集約する。

必須:

- `DB: D1Database`
- `CACHE: KVNamespace`
- `ENV: string`
- `SESSION_SIGNING_KEY: string`
- `RATE_LIMIT_SECRET: string`

任意（初期シード時のみ）:

- `OPERATOR_PASSWORD_HASH_B64?: string`
- `OPERATOR_PASSWORD_SALT_B64?: string`
- `OPERATOR_PASSWORD_ITERATIONS?: string`

## 3. Request Context

全`loader/action`で共通コンテキストを使う。

- `requestId`
- `now`
- `env`
- `logger`
- `bindings`
- `actor`（participant or operator）

生成は1箇所（middleware）に固定する。

## 4. 依存注入の方針

- UseCaseは interface(Port) のみ受け取る
- Route層で実装クラスを組み立てる
- グローバルシングルトンを避け、テスト可能性を優先

## 5. 共通middleware

最初に実装するmiddleware:

- `withRequestId`（`X-Request-Id` 生成/透過）
- `withErrorHandling`（`AppError` -> APIエラー形式）
- `withRateLimit`（回答系・ログイン系）
- `withOperatorAuth`（operator API向け）

## 6. APIレスポンス整形

成功/失敗で形式を固定:

- 成功: endpoint固有のJSON
- 失敗:

```json
{
  "error": {
    "code": "CONFLICT_STATE",
    "message": "Current progress does not allow this operation.",
    "requestId": "req_xxx"
  }
}
```

## 7. 実装時の禁止事項

- Domain層でD1/KVへ直接アクセスしない
- Route層に状態遷移ロジックを書かない
- `any` で境界型をごまかさない
- KVに正データを書かない
