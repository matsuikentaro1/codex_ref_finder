# codex_ref_finder

[OpenAI Codex CLI](https://github.com/openai/codex) を使って学術文献を検索し、結果をCSVに出力する [Claude Code](https://docs.anthropic.com/en/docs/claude-code) カスタムスラッシュコマンドです。PubMedを主な検索ソースとし、書籍・ガイドライン・プレプリントなどPubMed未掲載の文献にも対応します。

## 特徴

- Codex CLI経由でPubMedなどの学術データベースを検索
- 標準化されたCSV列（PMID・著者・タイトル・ジャーナル・DOI・要旨など）で結果を出力
- DOIベースの重複チェックにより、既存CSVへの安全な追記が可能
- 各文献に最大5つの文脈別知見（`whats_interesting1`〜`5`）を付与
- 複数テーマの並列検索に対応（レースコンディション防止機構付き）

## 必要環境

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [OpenAI Codex CLI](https://github.com/openai/codex)（`npm install -g @openai/codex`）
- Python 3（Codex内部でPubMed API呼び出しに使用）

## インストール

`.claude/commands/codex_ref_finder.md` をプロジェクトの `.claude/commands/` ディレクトリにコピーしてください。

```bash
# リポジトリをクローン
git clone https://github.com/matsuikentaro1/codex_ref_finder.git

# コマンドファイルをプロジェクトにコピー
mkdir -p /path/to/your/project/.claude/commands
cp codex_ref_finder/.claude/commands/codex_ref_finder.md /path/to/your/project/.claude/commands/
```

## 使い方

Claude Code上でスラッシュコマンドを実行します：

```
/codex_ref_finder
```

### 基本的な文献検索（単発実行）

キーワードを指定してPubMedを検索し、結果をCSVに保存します。

```bash
codex exec --full-auto --sandbox danger-full-access --skip-git-repo-check \
  --cd "C:\path\to\project" \
  "「腸内細菌叢とうつ病」に関する論文をPubMedから検索して、CSVファイルに保存してください。
   保存先: manuscripts/refs.csv。
   保存方法: 既存CSVがあればdoiで重複チェックし、既存論文は次の空きwhats_interesting列に追記、新規論文は行追加。
   CSV列: PubMed_ID, Author, Year, Title, Journal, Volume, Issue, Pages, doi, abstract, whats_interesting1~5。
   重要: Pythonスクリプトを一時ファイルとして書き出してから実行すること。"
```

### 特定論文を指定して検索

著者名・年・ジャーナルなどから特定の論文を検索してCSVに追記します。

```bash
codex exec --full-auto --sandbox danger-full-access --skip-git-repo-check \
  --cd "C:\path\to\project" \
  "以下の論文をPubMedから検索してCSVに追記してください:
   (1) Herring et al. 2016 Ann Intern Med suvorexant
   (2) Mignot et al. 2022 Lancet Neurol daridorexant
   保存先: manuscripts/refs.csv。
   重要: Pythonスクリプトを一時ファイルとして書き出してから実行すること。"
```

### 並列実行（複数テーマを同時検索）

複数テーマを同時に検索する場合、レースコンディション防止のため各codexは個別の一時CSVに出力し、全完了後にマージします。

```bash
# 検索1
codex exec ... "「VLPO sleep regulation」に関する論文を検索。保存先: _tmp_refs_01.csv（新規作成）。..."
# 検索2
codex exec ... "「amygdala prefrontal sleep deprivation」に関する論文を検索。保存先: _tmp_refs_02.csv（新規作成）。..."
# 検索3
codex exec ... "「circadian rhythm bipolar disorder」に関する論文を検索。保存先: _tmp_refs_03.csv（新規作成）。..."
```

全完了後にマージスクリプトを実行してメインCSVに統合します（マージ処理のPythonテンプレートはスキルファイル内に記載）。

## 並列実行のルール

1. **各codex検索は個別の一時CSVに書き出す** — ファイル名: `_tmp_refs_<連番>.csv`
2. **全完了後にメインプロセスでマージ** — DOIで重複チェック、新規は行追加、既存はwhats_interesting列に追記
3. **マージ処理はClaude Code本体（親プロセス）が実行** — codex子プロセスにはマージさせない
4. **絶対に複数codexが同じCSVに同時書き込みしない**

## CSV列定義

| 列名 | 説明 |
|------|------|
| PubMed_ID | PubMed ID（PMID）。PubMed未掲載の場合はエムダッシュ（—） |
| Author | 著者名（セミコロン区切り） |
| Year | 発行年 |
| Title | 論文タイトル |
| Journal | ジャーナル名 |
| Volume | 巻番号 |
| Issue | 号番号 |
| Pages | ページ範囲 |
| doi | DOI |
| abstract | 要旨 |
| whats_interesting1〜5 | 検索文脈に関連した知見（最大5つの異なる検索文脈に対応） |

## コマンドオプション

| オプション | 説明 |
|------|------|
| `--full-auto` | 確認なしで自動実行 |
| `--sandbox danger-full-access` | ファイル書き込みとネットワークアクセスの両方を許可 |
| `--skip-git-repo-check` | Gitリポジトリ外のディレクトリでも実行可能にする |

## 注意事項

### PowerShellエスケープ問題の回避

Codexは内部でPowerShellを使用するため、here-string（`@'...'@`）やパイプ経由のPython実行でエスケープエラーが発生しやすいです。**必ずPythonスクリプトを一時ファイルとして書き出してから実行してください。**

### 重複論文の扱い

- DOIで重複チェックを行い、同じ論文は同一行に保持
- 異なる検索文脈での知見は `whats_interesting1`〜`5` の空き列に順次追記
- DOIが空欄の論文は常に新規行として追加

## ライセンス

MIT
