# life

思考・感情・行動計画を記録するための個人用リポジトリ。

運用ルールの詳細は [GUIDELINES.md](./GUIDELINES.md) を参照。

## ざっくりした使い方

1. 出来事やテーマについて思ったことがあれば `notes/events/` または `notes/themes/` にMarkdownで書く。
2. 書いていて「やるべきこと」に気づいたら、その場でこのリポジトリに GitHub Issue を立てる。
3. 週1回くらいの頻度で `reviews/` にレビューを書き、Issueを整理する。

テンプレートは `templates/` にある。

## スマホ用Issueビューア

`docs/` に、スマホからIssueの一覧・詳細を見てワンタップでクローズ/再オープンできるページを置いている。GitHub Pages(main ブランチの `/docs`)で公開し、https://chiki-chiki.github.io/life/ を開いてホーム画面に追加して使う。

クローズ/再オープンには fine-grained トークン(対象: このリポジトリのみ、権限: Issues の Read and write)が必要。トークンはページの鍵アイコンから登録し、その端末のブラウザ内にだけ保存される。
