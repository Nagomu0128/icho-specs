# 06 参加者ドメイン状態遷移仕様

このドキュメントは、参加者進行の状態遷移ロジックを純粋関数として実装するための仕様を定義する。

依存: `05-d1-data-model-and-drizzle-implementation-details.md`

## 1. 対象関数

`app/modules/progress/domain/state-machine.ts` に以下を実装する。

- `startOrResumeSession(user)`
- `submitQ1Answer(user, subQuestion, normalizedAnswer)`
- `confirmQ1Checkpoint(user, subQuestion, checkpointCode, method)`
- `submitQ2Answer(user, normalizedAnswer)`
- `confirmQ2Checkpoint(user, checkpointCode, method)`
- `submitQ3Keyword(user, normalizedAnswer)`
- `submitQ3Code(user, normalizedAnswer)`
- `submitQ4Answer(user, normalizedAnswer)`
- `markEpilogueViewed(user)`

## 2. 入力正規化

`app/modules/progress/domain/answer-normalizer.ts`:

- 前後空白除去
- 全角英数を半角
- 英字小文字化
- 整数ゼロ埋め同値化（`029` -> `29`）

## 3. Q1順序固定ルール

- `q1Order` 未設定なら初回開始時にランダム決定
- 一度確定した `q1Order` は変更不可
- `currentUnlockedSubQuestion` は常に未完了側を指す

## 4. ステージ遷移表（参加者）

- `START -> Q1`（開始時）
- `Q1`:
  - サブ設問1つ完了で `Q1` 継続（もう片方を解放）
  - 2つ完了で `Q2`
- `Q2`:
  - 回答正解後checkpoint完了で `Q3_KEYWORD`
- `Q3_KEYWORD`:
  - keyword正解で `Q3_CODE`
- `Q3_CODE`:
  - code正解で `Q4`
- `Q4`:
  - 正解で `FAKE_END`
- `FAKE_END`:
  - エピローグ閲覧記録で `COMPLETE`

無効遷移は必ず `CONFLICT_STATE`。

## 5. 判定関数分離

`app/modules/progress/domain/answer-judge.ts`:

- `isQ1AnswerCorrect(subQuestion, normalizedAnswer)`
- `isQ2AnswerCorrect(normalizedAnswer)`
- `isQ3KeywordCorrect(normalizedAnswer)`
- `isQ3CodeCorrect(normalizedAnswer)`
- `isQ4AnswerCorrect(normalizedAnswer)`

初期は固定解答テーブルで実装し、外部I/Oは禁止。

## 6. Domainイベント出力

UseCaseがログを書きやすいように、各関数は結果にイベント配列を含める。

例:

- `ANSWER_CORRECT`
- `CHECKPOINT_COMPLETED`
- `STAGE_TRANSITION`
- `EPILOGUE_VIEWED`

## 7. テスト観点（unit）

- Q1順序固定（再開時に再抽選されない）
- 未解放サブ設問への回答はエラー
- 正解でもcheckpoint前に次ステージへ進まない
- `FAKE_END` 到達後のみ `epilogue_viewed` 可
- 不正ステージからのAPI呼び出しで `CONFLICT_STATE`
