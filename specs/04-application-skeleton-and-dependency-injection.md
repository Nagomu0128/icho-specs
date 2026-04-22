# 04 アプリケーション骨格と依存注入（Feature First + Ports and Adapters）

このドキュメントは、React Router v7 Framework Mode + Cloudflare Workers の実装を、
ヘキサゴナルアーキテクチャと Package by Feature で構成するための標準を定義する。

依存: `01-repository-structure-and-implementation-guidelines.md`

## 1. 目標構成

```text
app/
  routes/                                  # React Router v7規約。loader/actionは薄く保つ
  modules/
    progress/
      domain/
        state-machine.ts
        answer-normalizer.ts
        answer-judge.ts
        progress.ts
        errors.ts
      application/
        ports/
          driving/
            submit-answer.usecase.ts
          driven/
            progress-repository.port.ts
            idempotency-repository.port.ts
        usecases/
          submit-answer.usecase.ts
      infrastructure/
        progress-repository.d1.server.ts
        idempotency-repository.d1.server.ts
      authorization/
        policies.ts
    operator-session/
      domain/
        operator-session.ts
        errors.ts
      application/
        ports/
          driving/
          driven/
            operator-session-repository.port.ts
        usecases/
      infrastructure/
        operator-session-repository.d1.server.ts
  shared/
    ports/
      clock.port.ts
      id-generator.port.ts
    infrastructure/
      clock.system.server.ts
      id-generator.uuid.server.ts
    schemas/
    errors/
    result.ts
  composition.server.ts
```

補足:
- `src/` 起点のレイヤ分割は廃止し、`app/modules/<feature>/` へ移行する。
- 技術名でディレクトリ分割しない。技術差はファイルサフィックスで表現する。

## 2. 設計原則

1. Screaming Architecture: ルート構成から機能が分かること
2. Package by Feature: 変更が `modules/<feature>/` に閉じること
3. 依存方向: `routes -> application -> domain`
4. Ports and Adapters分離: `driving` と `driven` を分離すること
5. 技術はサフィックス表現: `*.d1.server.ts`, `*.kv.server.ts`
6. `.server.ts` 規約厳守: インフラ、composition、Workers依存コードは必ず `.server.ts`
7. loader/actionは薄く: zod検証 -> usecase呼び出し -> レスポンス整形のみ
8. 認可はusecase責務: participant/operatorでusecaseを分けずRBACポリシーで分岐
9. DIはコンポジションルート集中: `composition.server.ts` で ports を具象に束ねる
10. YAGNI: 抽象は必要な境界にだけ導入する

## 3. Env定義とRequest Context

`workers/bindings/env.ts` に型を集約する。

必須:
- `DB: D1Database`
- `CACHE: KVNamespace`
- `ENV: string`
- `SESSION_SIGNING_KEY: string`

任意（初期シード時のみ）:
- `OPERATOR_PASSWORD_HASH_B64?: string`
- `OPERATOR_PASSWORD_SALT_B64?: string`
- `OPERATOR_PASSWORD_ITERATIONS?: string`

共通Request Context（全 loader/action）:
- `requestId`
- `now`
- `env`
- `logger`
- `bindings`
- `actor`

Context生成は1箇所（middlewareまたは共通ヘルパ）に固定する。

## 4. Ports定義ルール

- `application/ports/driving/`: usecaseの公開インターフェース（in-bound）
- `application/ports/driven/`: repository等の依存先インターフェース（out-bound）
- `shared/ports/`: 真に横断的な副作用ポート（clock, id-generator）

禁止:
- `domain` から `driven` への直接依存
- `driving` と `driven` の混在

## 5. DIとコンポジションルート

- `app/composition.server.ts` で全 `driven port -> infrastructure` を接続する
- `getContainer(env)` 形式で loader/action から利用可能にする
- Route層で `new D1...Repository()` を直接呼ばない

利用イメージ（参照実装要件）:
1. routeでzod検証
2. containerからusecase取得
3. usecase実行
4. 共通エラー/Resultでレスポンス整形

## 6. 認可ポリシー

- `app/modules/<feature>/authorization/policies.ts` に集約する
- actor名でusecaseディレクトリを分けない
- 同一usecase内でポリシー評価して許可/拒否を判定する

認可モデル（採用）:
- 本プロジェクトは RBAC を採用する（role: `participant` / `operator`）
- 判定は `role + 操作対象 + 現在状態` を入力にしたポリシー関数で行う
- 実装はシンプルな関数ベースとし、外部ポリシーエンジンは導入しない
- 将来複雑化した場合は、同一インターフェースを維持してPBAC相当へ差し替え可能とする

## 7. zod / Result / エラー配置

- zodスキーマ:
  - モジュール境界固有: `modules/<feature>/application/schemas/`
  - 汎用: `app/shared/schemas/`
- Resultヘルパ: `app/shared/result.ts`
- 横断エラー基底: `app/shared/errors/`
- ドメイン固有エラー: `modules/<feature>/domain/errors.ts`

## 8. 移行フェーズ（仕様先行）

### Phase 1: 骨格作成
1. 現行 `src/` の責務棚卸し
2. `app/modules/`, `app/shared/` を先に作成
3. `app/composition.server.ts` をスケルトン作成

### Phase 2: progress移行
1. `domain/progress/*` -> `modules/progress/domain/`
2. progress関連port -> `modules/progress/application/ports/driven/`
3. usecaseを機能軸に統合し `driving` を追加
4. progress関連実装を `modules/progress/infrastructure/*.{d1,kv}.server.ts` に移行

### Phase 3: operator-session独立
1. `modules/operator-session/domain/operator-session.ts` を新設
2. usecase/port/実装を同モジュールに統合
3. shared/auth配置案が必要なら判断材料を提示して選択する

### Phase 4: 横断要素整備
1. `clock/id-generator` を `app/shared/ports` へ
2. 具象を `app/shared/infrastructure/*.server.ts` へ
3. `services/` の曖昧フォルダを廃止して責務別に再配置
4. zod / Result / errors を規約どおり配置

### Phase 5: コンポジションルート実装
1. `composition.server.ts` でdriven portを束ねる
2. `env.DB`, `env.CACHE` を具象へ注入
3. loader/actionの利用例を1ルート分作成

### Phase 6: routes整備
1. routeはcomposition経由でusecase実行
2. 先頭でzod検証を実施

## 9. 実装時の禁止事項

- `domain` から他モジュールやinfrastructureへimportしない
- アクター名（participant/operator）でusecaseディレクトリを分けない
- `services/` という曖昧フォルダを作らない
- `.server.ts` サフィックスを外さない
- domain層へ zod / D1 / Cloudflare SDK 依存を持ち込まない
- Route層に状態遷移ロジックを書かない
- `any` で境界型をごまかさない
- KVに正データを書かない

## 10. フェーズ完了時の報告テンプレート

各Phase完了時は次を報告する。
1. 移動・新規作成ファイル一覧（旧 -> 新）
2. import更新箇所数
3. 設計判断ポイントと採用理由
4. 次Phase前の確認事項

## 11. 不明点の扱い

以下は実装前に選択肢とトレードオフを提示して合意する。
- operator-session を feature配下に置くか shared/auth に置くか
- usecase の機能境界
- authorization の粒度
