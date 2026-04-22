# 01 リポジトリ構成と実装ガイドライン

このドキュメントは、`tech-specs.md` の前提を変えずに、初期実装で迷わないためのリポジトリ構成と責務分離を定義する。

## 1. 採用技術の固定条件

- ランタイム: Cloudflare Workers
- Web: React Router Framework Mode
- 言語: TypeScript
- 正データ: D1
- キャッシュ: KV
- インフラ管理: Terraform
- CI: GitHub Actions

上記は固定であり、代替技術への置換はしない。

## 2. ルートディレクトリ構成

```text
icho26/
  app/
    routes/                    # React Router routes / loaders / actions
    modules/                   # Feature単位（progress, operator-session, ...）
    shared/                    # 横断的ports / infra / schemas / result / errors
    composition.server.ts      # DIコンポジションルート
  workers/
    bindings/                  # Env型・Binding取得ヘルパ
    middleware/                # 認証・requestId
  db/
    schema/                    # Drizzle schema(ts)
    migrations/                # SQL migration
    seeds/                     # 初期データ投入
  terraform/
    modules/
      workers_service/
      d1/
      kv/
      secrets/
      ci_oidc/
    envs/
      prod/
  tests/
    unit/
    integration/
    e2e/
  specs/                       # この仕様群
```

## 3. 実装レイヤ責務

- `app/routes/`
  - HTTP入出力、`loader`/`action`
  - zod検証、usecase呼び出し、レスポンス整形のみ
- `app/modules/<feature>/domain/`
  - ステージ遷移、回答正規化、正答判定、ドメインエラー
  - I/Oを持たない純粋ロジック
- `app/modules/<feature>/application/`
  - usecase、認可呼び出し、トランザクション境界
  - portsを `driving` / `driven` に分離
- `app/modules/<feature>/infrastructure/`
  - D1/KVなど技術依存実装（`*.d1.server.ts`, `*.kv.server.ts`）
- `app/shared/`
  - 横断ports、Result、共通エラー、共通スキーマ

## 4. 命名規約

- ファイル名: kebab-case
- TypeScript型: PascalCase
- 関数: camelCase
- API DTO: `XxxRequest` / `XxxResponse`
- UseCase: `verb-target.usecase.ts`
- Repository Port: `xxx-repository.port.ts`
- Infrastructure実装: `*.d1.server.ts` / `*.kv.server.ts` / `*.server.ts`

## 5. エラー標準化

`tech-specs.md` のエラーコードを固定で採用する。

- `BAD_REQUEST`
- `UNAUTHORIZED`
- `FORBIDDEN`
- `NOT_FOUND`
- `CONFLICT_STATE`
- `INTERNAL_ERROR`

実装上は `AppError` を1つ定義し、`code`, `message`, `httpStatus`, `requestId` を必須にする。

## 6. 追加で管理対象に含めるべきクラウド要素

ユーザー指定の管理対象（Workers / D1 / KV / Secrets / CI連携）に加え、初期から次をTerraform管理対象に含める。

- Worker Route / Custom Domain（本番公開時に必要）
- GitHub Actions用 OIDC 連携（長期鍵レス運用）
- 環境ごとの `wrangler.toml` 相当設定値の一元管理（TF outputsで生成元を固定）

次は将来拡張として扱い、初期スコープ外にする。

- Durable Objects（`tech-specs.md` の将来拡張）
- Queues / R2 / Logpush

## 7. 実装順序（高レベル）

1. Terraform基盤（単一環境運用・最低限リソース）
2. D1スキーマとDrizzle定義
3. 共通ミドルウェア（認証・エラー・リクエストID）
4. 参加者API
5. 運営認証と運営API
6. KVキャッシュ最適化
7. CI/E2E/運用監視
