# 11 実装タスク一覧（AI投入単位）

このドキュメントは、AI codingへ順番に投入する最小タスク単位を定義する。

## 1. 実装順（依存あり）

1. Terraform module骨格作成（`02`, `03`）
2. `envs/prod` の `plan` 可能化と手動 `apply` フロー整備（`02`, `03`）
3. D1 schema + migration + Drizzle定義（`05`）
4. 共通エラー型とmiddleware（`04`）
5. RequestId + 共通エラーレスポンス整備（`04`）
6. 参加者Domain（正規化/判定/遷移）（`06`）
7. 参加者API start/progress 実装（`07`）
8. 参加者API answer/checkpoint 実装（`07`）
9. 運営login/logout + session検証 実装（`08`）
10. 運営dashboard/group取得 実装（`08`, `09`）
11. 運営status-correction/mark-reported 実装（`08`, `09`）
12. KV世代管理 + フォールバック実装（`09`）
13. 日次クリーンアップJob実装（`09`）
14. CI（lint/type/unit/integration/terraform plan）実装（`10`）
15. deploy workflow（prod手動承認）実装（`10`）
16. E2E主要シナリオ追加（`10`）
17. 非機能SLO計測（p95/同時接続）と監視指標ダッシュボード整備（`10`）
18. モバイル実機受け入れ（通常導線/低速回線/NFCフォールバック）実施（`10`）

## 2. 1タスクの投入サイズ目安

- 変更ファイル: 3〜8ファイル
- 追加コード: 150〜350行
- テスト: 最低1ファイル追加
- 1PRに複数ドメインを混在させない

## 3. AIに渡す指示テンプレート（短文）

以下を毎回貼る:

- 対象仕様: `specs/xx-...md`
- 守る前提: `tech-specs.md` を変更しない
- 変更範囲: 対象レイヤのみ
- 必須: テスト追加、型エラー0
- 禁止: D1/KV責務違反、ルートに業務ロジック直書き

## 4. 並列実装してよい組み合わせ

- A: Terraform module整備
- B: Domain純粋関数
- C: CIテンプレート

ただし、API実装はD1 schema確定後に開始する。

## 5. レビュー観点（最小）

- `CONFLICT_STATE` を正しく返しているか
- idempotencyが更新系全APIで強制されているか
- `state_version` 条件付き更新が漏れていないか
- KV障害でもD1正データで継続できるか
