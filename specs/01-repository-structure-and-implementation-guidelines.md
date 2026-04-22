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
icho-specs/
  app/                         # React Router routes / loaders / actions
  src/
    domain/                    # 純粋関数: 状態遷移・正規化・判定
    application/               # UseCase / Port(interface)
    infrastructure/            # D1/KV/外部サービス実装
    shared/                    # 共通型・Result・エラー
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
      dev/
      stg/
      prod/
  tests/
    unit/
    integration/
    e2e/
  specs/                       # この仕様群
```

## 3. 実装レイヤ責務

- `app/`
  - HTTPの入出力、画面遷移、`loader`/`action`
  - バリデーション呼び出しとUseCase呼び出しのみ
- `src/domain/`
  - ステージ遷移、回答正規化、正答判定
  - I/Oを持たない純粋関数
- `src/application/`
  - トランザクション境界、冪等制御、権限判定
  - Repository interface(Port)を定義
- `src/infrastructure/`
  - D1/KVアクセス、SQL、Cloudflareバインディング依存コード

## 4. 命名規約

- ファイル名: kebab-case
- TypeScript型: PascalCase
- 関数: camelCase
- API DTO: `XxxRequest` / `XxxResponse`
- UseCase: `verb-target.usecase.ts`
- Repository Port: `xxx-repository.port.ts`
- Infrastructure実装: `d1-xxx.repository.ts` / `kv-xxx.cache.ts`

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

ユーザー指定の管理対象（Workers / D1 / KV / Secrets / 環境分離 / CI連携）に加え、初期から次をTerraform管理対象に含める。

- Worker Route / Custom Domain（本番公開時に必要）
- GitHub Actions用 OIDC 連携（長期鍵レス運用）
- 環境ごとの `wrangler.toml` 相当設定値の一元管理（TF outputsで生成元を固定）

次は将来拡張として扱い、初期スコープ外にする。

- Durable Objects（`tech-specs.md` の将来拡張）
- Queues / R2 / Logpush

## 7. 実装順序（高レベル）

1. Terraform基盤（環境分離・最低限リソース）
2. D1スキーマとDrizzle定義
3. 共通ミドルウェア（認証・エラー・リクエストID）
4. 参加者API
5. 運営認証と運営API
6. KVキャッシュ最適化
7. CI/E2E/運用監視

## 8. この後の仕様ファイルの読み方

- `02`〜`10` はそれぞれ単一テーマで実装可能な粒度に分割
- 1ファイルずつAIに入力しても成立するように、依存関係と実装範囲を冒頭で明示する
- 先行ファイルの決定事項を後続ファイルで上書きしない
