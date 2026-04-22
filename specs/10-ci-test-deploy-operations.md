# 10 CIテストデプロイ運用

このドキュメントは、0->1実装を安全に進めるためのCI・テスト・デプロイ手順を定義する。

依存: `02-terraform-environment-separation-and-state-management.md`, `03-terraform-resource-definition-details.md`

## 1. ブランチ戦略

- `main`: デプロイ可能状態を維持
- featureブランチ: PRで検証
- squash mergeを基本に履歴を簡潔化

## 2. GitHub Actionsワークフロー

### `ci.yml`（PR時）

1. `npm ci`
2. `npm run typecheck`
3. `npm run test:unit`
4. `npm run test:integration`
5. `terraform fmt -check`（`terraform/`）
6. `terraform validate`
7. `terraform plan`（`dev`）

### `deploy-dev.yml`（main push）

1. app build
2. D1 migration apply（dev）
3. Worker deploy（dev）
4. smoke test

### `deploy-stg-prod.yml`（手動）

- `workflow_dispatch` + environment approval
- `stg` 完了後に `prod`

## 3. テスト構成

- unit:
  - domain正規化
  - state machine遷移
  - PBKDF2比較ロジック
- integration:
  - APIエンドポイント
  - idempotency
  - state_version競合
  - KVフォールバック
- e2e:
  - 開始->完了導線
  - 運営ログイン->補正->報告済み

## 4. 受け入れ確認（自動化対象）

`tech-specs.md` の受け入れ項目をCIに写像する。

- Q1順序固定
- 未解放URLで `CONFLICT_STATE`
- Q1完了後のみQ2解放
- PBKDF2運営認証

## 5. デプロイ前チェック

- migration差分の確認
- secrets注入漏れ確認
- `SESSION_SIGNING_KEY` のローテーション計画確認

## 6. ロールバック

- Worker: 直前バージョンへ戻す
- D1: 破壊的変更は禁止。forward migrationで修正
- secrets: 旧キー保持期間を短く設け段階切替

## 7. 運用runbook最小要件

- ログイン障害時の切り分け手順
- `CONFLICT_STATE` 急増時の調査手順
- KV障害時のD1フォールバック確認手順
- 日次クリーンアップ失敗時の再実行手順
