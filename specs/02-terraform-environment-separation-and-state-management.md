# 02 Terraform環境分離と状態管理

このドキュメントは、CloudflareリソースをTerraformで管理するための環境分離ルールを定義する。

依存: `01-repository-structure-and-implementation-guidelines.md`

## 1. 環境戦略

- 環境: `dev`, `stg`, `prod`
- すべて同一コード、変わるのは変数のみ
- `prod` への手動承認を必須化

## 2. terraformディレクトリ構成

```text
terraform/
  modules/
    workers_service/
    d1/
    kv/
    secrets/
    ci_oidc/
  envs/
    dev/
      main.tf
      providers.tf
      variables.tf
      terraform.tfvars
      outputs.tf
    stg/
      ...
    prod/
      ...
```

## 3. State管理

- リモートStateを使用（Cloudflare R2 backend もしくは Terraform Cloud）
- lockを有効化できるバックエンドを使う
- ローカルStateのコミットは禁止
- `prod` はState閲覧権限を最小化

## 4. Providersとバージョン固定

- `hashicorp/terraform` の required_version を固定
- `cloudflare/cloudflare` provider versionを固定
- 変更時は `dev -> stg -> prod` の順で検証

## 5. 環境変数の設計

最低限必要な入力変数:

- `cloudflare_account_id`
- `cloudflare_zone_id`（Route/Domainを使う場合）
- `project_name`（例: `icho-game`）
- `environment`（`dev`/`stg`/`prod`）
- `worker_name`
- `d1_database_name`
- `kv_namespace_name`

`prod` だけ変える値:

- ドメイン/ルート
- レート制限の厳しさ
- 監視閾値

## 6. 命名規則

`{project}-{env}-{resource}` を基本にする。

例:

- Worker: `icho-game-dev-app`
- D1: `icho-game-dev-d1`
- KV: `icho-game-dev-kv`

## 7. 実装手順

1. `modules/*` のインターフェースを確定
2. `envs/dev` を先に作成して `terraform plan/apply`
3. `envs/stg` を同一構成で作成
4. `envs/prod` を作成し、applyには手動承認を挟む

## 8. CI連携方針

- PR時: `terraform fmt -check`, `validate`, `plan`
- mainマージ後:
  - `dev` は自動apply可
  - `stg` は手動承認後apply
  - `prod` は必ず手動承認

## 9. 失敗時ルール

- `apply` 失敗時に手動でCloudflare側を変更しない
- 先にStateとの差分原因を特定して再plan
- 緊急対応で手動変更した場合、同日中にTFへ取り込む
