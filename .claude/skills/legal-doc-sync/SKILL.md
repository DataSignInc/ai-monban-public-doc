---
name: legal-doc-sync
description: Googleドキュメントで作成・レビューされたプライバシーポリシー／利用規約の更新案を、ai-monban-public-docのMarkdown（privacy-policy.md／terms-of-service.md）に反映してPRを作成する標準フロー
argument-hint: "<Googleドキュメントの共有URL> [反映先: privacy-policy.md | terms-of-service.md]"
---

# 法務文書（Googleドキュメント → GitHub Markdown）反映フロー

> **応答言語**: このスキルの実行中、依頼者への返答・報告・確認事項はすべて日本語で書くこと（コード・コマンド・ログ・エラーメッセージの引用は原文のままでよい）。サブエージェントに作業を任せる場合も、日本語で報告させる。

`ai-monban-public-doc` は `privacy-policy.md`（プライバシーポリシー）と
`terms-of-service.md`（利用規約）の2ファイルのみを持つリポジトリ。法務文書の
更新案は社内でGoogleドキュメントとして作成・レビューされることが多く、確定した
内容をこのリポジトリのMarkdownに反映してPRを作成するのが本スキルの役割。

## 前提知識（リポジトリ概要）

- ビルド・デプロイの仕組みは無く、2つのMarkdownファイルのみ
- `develop`/`main` の両ブランチにPR必須のルールセットが設定済み（2026-08-21〜）。
  直接pushはできないため、必ず作業ブランチ→PRの手順を踏む
- CIワークフロー（2026-08-21追加）:
  - `markdown-lint.yml`: `markdownlint-cli2` で `*.md` を検証。既存の法的文書が
    HTMLテーブル（`<ul><li>`等）や独自のリスト記法を使っているため、
    `.markdownlint-cli2.yaml` でスタイル系ルール（MD001/MD013/MD022/MD029/
    MD030/MD032/MD033/MD055/MD060）を無効化している。**内容を変えずにCIを通す
    ための設定なので、新たにスタイル崩れを起こした場合以外はこの設定を緩めたり
    厳しくしたりしない**
  - `link-check.yml`: `lychee` で文書内の外部リンクの有効性を検証（push/PR/週次）。
    個人情報保護委員会の一部PDF（`Washington_report.pdf`／`massachusetts_report.pdf`）
    は既知の404で、これはリンク先サイト側の問題であり本リポジトリの変更では直せない。
    **required status checkには設定していない**ため、この既知の失敗がPRのマージを
    ブロックすることはない
  - `semgrep.yml`: 秘密情報の誤コミット検出（`p/secrets`のみ。コードが無いため
    Python/OWASP系configは使用していない）

## いつ使うか

- 「プラポリ／利用規約のこの内容をGitHubに反映して」のような直接依頼
- Googleドキュメントの共有URL（`docs.google.com/document/d/...`）が渡された場合

## 作業フロー

### 1. Googleドキュメントを取得する

- URLから `documentId`（`/d/` と `/edit` の間の文字列）を取り出す
- `mcp__claude_ai_AI_MONBAN__Google_Docs_readGoogleDoc` は deferred tool なので、
  先に `ToolSearch` で `select:mcp__claude_ai_AI_MONBAN__Google_Docs_readGoogleDoc`
  を呼んでスキーマをロードしてから使う。`format: "markdown"` で取得する
- **ネイティブGoogleドキュメントのみ対応**。アップロードされた`.docx`等の
  Officeファイルだと `"This operation is not supported for this document.
  The document must not be an Office file."` エラーになる。この場合は依頼者に
  Google Drive上で「ファイル → Googleドキュメントで保存」してコピーを作って
  もらい、新しいドキュメントURLを教えてもらう（既存ファイルを直接変換できる
  操作はない）
- `listGoogleDocs`等によるDrive全体の検索・一覧は権限（scope）が無く
  `Permission denied` になることがある。渡された直接のドキュメントURLに対する
  `readGoogleDoc`／`getDocumentInfo` だけを使う前提で進める

### 2. DLPマスキングを踏まえて内容を読む（最重要）

このAI MONBANのコネクタ経由で取得したテキストは、システムのDLPパイプラインを
通っており、**組織名・住所・氏名・メールアドレス等が `[REDACTED:organization]`
`[REDACTED:address]` `[REDACTED:person]` `[REDACTED:email]` に自動マスキング
されている場合がある**。これは実際にその文書に記載されている値であることを
示すものであり、値が削除・変更されたことを意味しない。

- **`[REDACTED:...]` の文字列をそのままファイルに書き込んではならない**
- 誤検知にも注意する。例えば「ユーザーが指定した」の「ユーザー」が
  `[REDACTED:organization]` に誤分類される、テーブルセル内のリンクや
  `<br>`/`<ul><li>` 等のHTML装飾がマスキングや変換の過程で丸ごと失われる、
  といったことが実際に起きる
- マスキングされた箇所は、現行ファイル（Step 3）の対応箇所と比較し、
  実質的に同じ内容なら**現行ファイルの表記（実名・リンク・HTML装飾）を
  そのまま維持する**。値が変わったかどうかがマスキングのせいで判断できない
  場合は、推測せず依頼者に確認する

### 3. 現行ファイルと突き合わせて実質的な差分を洗い出す

- `Read` で反映先ファイル（`privacy-policy.md` または `terms-of-service.md`）の
  現状を確認する
- Step 1で取得したドラフトと文単位・条文単位で比較し、以下を分類する
  - **確実な追加・変更**: 新しい条項、文言の実質的な変更（例: オプトイン条件の
    追加、法令条文への言及の追加）
  - **書式・変換由来の差分**: HTMLタグの欠落、テーブル内リンクの消失、
    箇条書きのネスト崩れなど。→ 現行ファイルの書式を維持し、無視する
  - **要確認**: 日付が未確定（`X月X日`等のプレースホルダ）、リンク先URLや
    ドメインが現行と異なる、連絡先・代表者名など実データの変更可能性がある
    箇所など、独断で決められない点
- 「要確認」に分類した項目は、`AskUserQuestion` で依頼者に選択肢を示して確認
  してから反映する。特に日付・URL・ドメイン・連絡先のような具体的な値は
  絶対に推測で埋めない

### 4. Markdownを編集する

- `Edit` で該当箇所のみを差し替える（全文書き換えはしない）
- 既存ファイルの書式規約（HTMLテーブル記法・`<br>`・`<ul><li>`・見出しレベル等）
  に合わせる。Googleドキュメントの変換結果の書式（プレーンな `|---|---|` 記法等）
  に引き寄せない

### 5. Lintを確認する

```bash
npx --yes markdownlint-cli2 "*.md"
```

`0 issues` になることを確認する。ならない場合は新たに追加した記載がテーブル
崩れ等を起こしている可能性が高いので、書式を見直す（`.markdownlint-cli2.yaml`
側のルール無効化を追加で緩めることはしない）。

### 6. ブランチ作成・コミット

- `develop`/`main` はPR必須のため、`develop` から作業ブランチを作成する

```bash
git checkout develop && git pull --ff-only
git checkout -b update/<内容が分かるブランチ名>
git add <変更したファイル>
git commit -m "<変更内容の要約>"
git push -u origin update/<ブランチ名>
```

### 7. PR作成

```bash
gh pr create --repo DataSignInc/ai-monban-public-doc --base develop \
  --head update/<ブランチ名> --title "<タイトル>" --body "<本文>"
```

PR本文には以下を明記する:

- Googleドキュメントのどの変更を反映したか（条項単位で列挙）
- Step 3で「要確認」として依頼者に確認し、方針が決まった判断ポイント
  （日付・リンク先・ドメイン等）
- レビュー・確認してほしい観点（法務・コンプライアンス確認が必要な旨など）

### 8. CI確認

```bash
gh pr checks <PR番号> --repo DataSignInc/ai-monban-public-doc
```

- `markdownlint` と `semgrep` は成功する前提。失敗した場合は内容を確認して修正する
- `link-check` は既知の外部リンク404（個人情報保護委員会の一部PDF）で失敗する
  ことがあるが、required status checkではないためPRのマージはブロックされない。
  新規に追加したリンクが原因で失敗していないかは必ずログで確認する

### 9. マージ依頼

- **マージは必ず依頼者自身に行ってもらう**。エージェントが `gh pr merge` 等で
  マージ操作を実行することはない
- 依頼者にPR URLを共有し、内容確認・法務レビューを依頼して終了する

## 注意点

- 法務文書のため、書式や誤字レベルの修正であっても内容を独断で判断・補完しない。
  Googleドキュメントのドラフトを「唯一の正」として無条件に採用せず、現行ファイル
  との矛盾や不明点は必ず依頼者に確認する
- DLPマスキングは完全ではない（誤検知・書式欠落がある）。マスキング後の文字列
  （`[REDACTED:...]`）を絶対にファイルへ書き込まない
- Google Docs APIはネイティブGoogleドキュメント形式のみ対応。`.docx`等の
  アップロードファイルは読めないため、依頼者に変換してもらう
