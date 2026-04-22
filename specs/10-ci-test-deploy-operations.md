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
7. `terraform plan`（`prod`）

### `deploy-prod.yml`（手動）

`workflow_dispatch` + Environment承認を通過した場合のみ実行する。

1. app build
2. D1 migration apply（prod）
3. Worker deploy（prod）
4. KV初期設定投入
5. smoke test
6. Q1順序固定シナリオのE2E確認

### `post-merge-check.yml`（main push）

1. app build
2. smoke test（applyなし）
3. `terraform plan`（`prod`）で差分再確認

## 2.1 デプロイポリシー（単一環境）

- `prod` への自動applyは禁止
- `prod` 反映は `workflow_dispatch` + Environment承認後のみ実行
- 緊急対応時もPR経由で差分レビューと実行ログ記録を必須化

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
- D1停止時の障害レスポンス

## 5. 受け入れ確認（手動）

- モバイル実機で主要導線が操作可能
- 画面が縦持ち最適化され、主要タップ領域が44px以上を満たす
- 入力欄の自動フォーカスとキーボード種別最適化が効いている
- NFC非対応端末でフォールバックQR導線が即時表示される
- 低速回線条件でローディング/再送導線が成立する
- Workers障害時にフォールバック文言が表示される

## 6. 非機能SLO/SLI

- 主要画面初回表示（4G相当）: p95 2.5秒以内
- 回答送信API: p95 600ms以内
- ダッシュボード取得: p95 800ms以内
- 想定ピーク同時接続: 50（参加者 + 運営合計）
- 監視指標: APIエラー率 / stage滞留時間 / ログイン失敗率

## 7. デプロイ前チェック

- migration差分の確認
- secrets注入漏れ確認
- `SESSION_SIGNING_KEY` のローテーション計画確認

## 8. ロールバック

- Worker: 直前バージョンへ戻す
- D1: 破壊的変更は禁止。forward migrationで修正
- secrets: 旧キー保持期間を短く設け段階切替

## 9. 運用runbook最小要件

- ログイン障害時の切り分け手順
- `CONFLICT_STATE` 急増時の調査手順
- KV障害時のD1フォールバック確認手順
- 日次クリーンアップ失敗時の再実行手順
- 一時失敗時の指数バックオフ再試行（最大2回）の動作確認手順
