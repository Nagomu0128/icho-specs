# 02 Terraform単一環境運用と状態管理

このドキュメントは、CloudflareリソースをTerraformで管理するための単一環境（`prod`）運用ルールを定義する。

依存: `01-repository-structure-and-implementation-guidelines.md`

## 1. 環境戦略

- 運用環境は `prod` のみとする
- 変更は必ずPR経由でレビューし、`apply` は手動承認を必須化する
- 将来 `dev` を追加する場合でも、module構成は再利用し変数とStateを分離して拡張する

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
    prod/
      main.tf
      providers.tf
      variables.tf
      terraform.tfvars
      outputs.tf
```

## 3. State管理

- リモートStateを使用（Cloudflare R2 backend もしくは Terraform Cloud）
- lockを有効化できるバックエンドを使う
- ローカルStateのコミットは禁止
- `prod` はState閲覧権限を最小化

## 4. Providersとバージョン固定

- `hashicorp/terraform` の required_version を固定
- `cloudflare/cloudflare` provider versionを固定
- 変更時はPR時の `plan` 結果をレビューしてから `prod` に反映する

## 5. 環境変数の設計

最低限必要な入力変数:

- `cloudflare_account_id`
- `cloudflare_zone_id`（Route/Domainを使う場合）
- `project_name`（例: `icho-game`）
- `environment`（`prod` 固定）
- `worker_name`
- `d1_database_name`
- `kv_namespace_name`

運用上の可変値:

- ドメイン/ルート
- 監視閾値

## 6. 命名規則

`{project}-{env}-{resource}` を基本にする。

例:

- Worker: `icho-game-prod-app`
- D1: `icho-game-prod-d1`
- KV: `icho-game-prod-kv`

## 7. 実装手順

1. `modules/*` のインターフェースを確定
2. `envs/prod` で `terraform plan` を実行し差分をレビュー
3. `envs/prod` の `terraform apply` を手動承認付きで実行

## 8. CI連携方針

- PR時: `terraform fmt -check`, `validate`, `plan`（`apply` は実行しない）
- mainマージ後:
  - 自動 `apply` は実行しない
  - `workflow_dispatch` + Environment承認で `prod` のみ `apply` を実行する

## 9. 失敗時ルール

- `apply` 失敗時に手動でCloudflare側を変更しない
- 先にStateとの差分原因を特定して再plan
- 緊急対応で手動変更した場合、同日中にTFへ取り込む
