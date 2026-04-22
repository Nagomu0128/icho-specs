# 03 Terraformリソース定義詳細

このドキュメントは、CloudflareリソースをTerraform module単位で実装するための具体仕様を定義する。

依存: `02-terraform-environment-separation-and-state-management.md`

## 1. module: workers_service

### 入力

- `name`
- `account_id`
- `script_path`（CIでビルド済みartifactを指す）
- `compatibility_date`
- `routes`（任意）
- `vars`（非機密）
- `secret_bindings`（機密名のみ）
- `d1_binding_name`, `d1_database_id`
- `kv_binding_name`, `kv_namespace_id`

### 出力

- `worker_name`
- `worker_script_id`
- `worker_routes`

### 注意

- secrets値はTerraform stateに平文保存させない運用を優先
- シークレット投入はCIステップで `wrangler secret put` か、安全に扱える仕組みで実施

## 2. module: d1

### 入力

- `account_id`
- `name`

### 出力

- `database_id`
- `database_name`

### 運用

- schema変更はDrizzle migrationで管理
- TerraformはDBインスタンスのライフサイクルのみ担当

## 3. module: kv

### 入力

- `account_id`
- `title`

### 出力

- `namespace_id`
- `namespace_title`

### 運用

- `dash:*`, `cfg:*`, `op:session:*` キー体系はアプリ側で固定
- TTLはアプリ側ロジックで設定

## 4. module: secrets

Terraformで値を持たない設計を推奨する。管理対象は「必要なシークレット名の定義」と「注入漏れ検知」まで。

### 必須シークレット名

- `SESSION_SIGNING_KEY`

初回シード専用（運用時は常用しない）:

- `OPERATOR_PASSWORD_HASH_B64`
- `OPERATOR_PASSWORD_SALT_B64`
- `OPERATOR_PASSWORD_ITERATIONS`

## 5. module: ci_oidc

GitHub Actions から Cloudflare へOIDCでアクセスするための設定を管理する。

### 要件

- 長期APIトークンをリポジトリに置かない
- リポジトリ単位・ブランチ単位で権限境界を設定
- `prod` deploy jobはEnvironment保護ルールを使う

## 6. envs/* の main.tf 例（構成）

1. `module.d1`
2. `module.kv`
3. `module.workers_service`（D1/KV出力をbinding）
4. `module.ci_oidc`

依存は明示的に `depends_on` ではなく、module output参照で解決する。

## 7. 初期適用後チェック

- Workerが起動し `GET /healthz` で200を返す
- D1/KV bindingが有効
- CIの `plan` と `apply` が通る
- `prod` 命名が既存Cloudflareリソースと衝突しない
