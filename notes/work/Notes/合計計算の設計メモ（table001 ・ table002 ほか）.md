調査日: 2026-06-15 / 対象ブランチ: feature/NAR_WEBAPP-386

## 1. 背景・対応内容

table001（選定申請書）・table002（完了報告書）の表で、画面に出ている「計」行を **Excel にも印字** するよう対応した。

- フロントの送信データ（スキーマ）は変更不要だった。合計のもとになる行データ（`table001.rows` / `table002.rows`）はすでにバックエンドに届いているため、**バックエンドで画面と同じ計算をして「計」行を印字** する方式とした。
- 共通の合計計算ヘルパーを新設: `backend/src/lib/document/builder/excel/table/totals.ts`
  - `calcTable001Totals` … 金額7列の合計
  - `calcTable002Totals` … 金額8列の合計（**各行の下段 `values[1]` のみ** を合計。画面仕様に合わせる）
- 各様式の formatter（計14ファイル）に「計」行を追加。
  - 選定申請書（table001）: All_common, I_3_1, I_3_2, I_4_common, I_5_1, II_1_central, II_1_local
  - 完了報告書（table002）: Report_common, Report_I_4_common, Report_I_5, Report_II_1_central, Report_II_1_local
  - 既存の「合計」行を修正: Report_I_3_1 / Report_I_3_2
    - 旧実装は「上段＋下段」を合算しており、画面（下段のみ）と数字が食い違っていた。共通ヘルパー経由の下段のみ合計に修正し、ラベルも「合計」→「計」に統一。
- バックエンドの型チェック（`npx tsc --noEmit`）はエラーなしで通過。実際の Excel 出力での目視確認は未実施。

## 2. 調査でわかった設計の実態

### フロントの計算値は送っていない

現行テーブル（`frontend/src/components/tables/new/`）には **`setValue`（計算結果をフォームへ書き戻す処理）が1か所も存在しない**。
→ どのテーブルも、画面で計算した合計をフォーム値として送信していない。**全テーブルが一貫して「表示はフロントで計算／印字はバックで再計算」方式**。

- table004 の `kei_a` / `kei_b` は「合計値」ではなく、**ユーザーが金額を手入力する行グループ**（`kei_a.rows[].kingaku`）。合計の受け渡しではない。
- table007 / table008 の `sum`（rule.ts 内）は画面表示ではなく **入力チェック用**。別枠。

### フロントとバックの両方で合計を計算しているテーブル（ロジック重複）

| テーブル | 内容 | フロント | バックエンド |
|---|---|---|---|
| table001 | 経費配分の計 | table001.tsx:142 | totals.ts に集約 |
| table002 | 経費配分の計 | table002.tsx:107 | totals.ts に集約 |
| table003 | 頭数・金額の計 | table003.tsx:64 | I_3_1 / I_3_2 formatter |
| table005 | 申請額の積算 | table005.tsx:100 | I_4_1 formatter |
| table006 | 機械等の額 | table006.tsx:102 | I_4_2 / Report_I_4_2 formatter |
| table020 | 補助頭数の計 | table020.tsx:68 | I_3_1 / I_3_2 formatter |
| table023 / _female | 生産振興額 | table023.tsx:104 | I_4_3male / female formatter |

## 3. なぜフロントとバックで同じ計算をするのか（メリット）

「手抜きの二重実装」ではなく、**信頼できないクライアント側の表示**と**信頼すべきサーバー側の出力**を分けた結果、同じ計算が2か所に現れる、と解釈できる。

1. **信頼境界 — クライアントの値を信用しない**
   フロントから送られる値は改ざん可能。合計をそのまま印字すると、行データと矛盾した数字の公的書類を作れてしまう。最終成果物はサーバーで計算し直すことで整合性を保証する。本プロジェクトではバックの合計は印字だけでなく添付Excelとの照合（style016.ts 等）にも使われており、ここを信用できないと検証が成立しない。
2. **Single source of truth は「入力行」**
   合計は入力から導出できる値。これをデータとして送ると「入力」と「送られた合計」の2つの真実が生まれ、ズレる余地ができる。送るのを生の入力だけに絞ると不整合が原理的に起きない。
3. **各層が独立して動ける**
   フロントはサーバーを待たず入力中にリアルタイムで合計表示できる（UX）。バックはフロント実装に依存せず、バッチ再生成など別経路でも正しい数字を出せる。
4. **テスト・保守の独立性**
   バックの出力ロジックをフロントなしで単体検証できる。

| | フロントの計算 | バックの計算 |
|---|---|---|
| 目的 | 入力中の即時表示（UX） | 公式な成果物の数字を保証 |
| 信頼性 | 改ざん可能（参考値） | サーバー管理で確実 |

## 4. 根拠とした原則（OWASP）

「クライアントを信用するな／導出値はサーバーで再計算せよ」は、OWASP の **Business Logic Security Cheat Sheet** に明文化された原則として実在を確認済み。

- 出典: https://cheatsheetseries.owasp.org/cheatsheets/Business_Logic_Security_Cheat_Sheet.html
- 関連: [OWASP Web Security Testing Guide - Test Business Logic Data Validation](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/10-Business_Logic_Testing/01-Test_Business_Logic_Data_Validation)

原文（該当箇所）:

> Never accept a price, subtotal, tax, or total from the client. Accept product identifiers and quantities only, and compute the rest on the server from your own database.

> if a value in the request influences price, access, ownership, or state, assume the client made it up and recompute it from trusted data.

価格改ざんの典型例:

> the application recalculates the total from the client-submitted price instead of looking it up server-side, so a user can pay one cent for a television.

補足: この原則が最も強く語られるのは「価格・権限・所有権・状態変更」の文脈。本件の合計はお金が即動くわけではないため EC の価格改ざんほど直接的な攻撃対象ではないが、「クライアントの合計を信じると入力と矛盾した公的書類を作れる」という点で、再計算する設計には同じ理屈で合理性がある。

## 5. 残課題

- **計算ロジックの二重管理**: 同じ式がフロントとバックの2か所にあり（table001/002/003/005/006/020/023）、仕様変更時に両方直す必要がある。片方を忘れるとフロント表示と Excel がズレる。
  - 解消の筋: フロントから合計を「送る」のではなく、**計算式を共有モジュール化して両方から呼ぶ**。メリット（サーバー再計算）を保ったまま重複を消せる。まず手軽で効果が大きいのは、バックエンド内で印字（totals.ts）と検証（style016 等）の合計計算を統一すること。
- 各テーブルで **フロントとバックの計算結果が実際に一致しているか**（式の取り違え・ズレがないか）は未検証。
- 今回追加した「計」行の Excel 出力の目視確認が未実施。