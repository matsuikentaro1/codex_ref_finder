# codex_ref_finder

[OpenAI Codex CLI](https://github.com/openai/codex) を使って学術文献を検索し、結果をCSVに出力する [Claude Code](https://docs.anthropic.com/en/docs/claude-code) カスタムスラッシュコマンドです。

## 特徴

- PubMedを中心に学術文献を検索（書籍・ガイドライン・プレプリントにも対応）
- PMID・著者・タイトル・DOI・要旨などを構造化してCSVに保存
- DOIベースの重複チェックで既存CSVへの安全な追記が可能
- 各文献に最大5つの文脈別知見（`whats_interesting1`〜`5`）を付与
- 複数テーマの並列検索に対応

## 必要環境

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [OpenAI Codex CLI](https://github.com/openai/codex)（`npm install -g @openai/codex`）
- Python 3

## インストール

`.claude/commands/codex_ref_finder.md` をプロジェクトの `.claude/commands/` ディレクトリにコピーしてください。

```bash
git clone https://github.com/matsuikentaro1/codex_ref_finder.git

mkdir -p /path/to/your/project/.claude/commands
cp codex_ref_finder/.claude/commands/codex_ref_finder.md /path/to/your/project/.claude/commands/
```

## 使い方

Claude Code上でスラッシュコマンドを実行します：

```
/codex_ref_finder
```

検索キーワードとCSVファイル名を伝えるだけで、文献検索からCSV保存まで自動で行います。

### できること

- **キーワード検索** — 「腸内細菌叢とうつ病」のようなテーマで論文を検索
- **特定論文の検索** — 「Herring et al. 2016 Ann Intern Med suvorexant」のように著者・年・ジャーナルを指定
- **並列検索** — 複数テーマを同時に検索し、結果を1つのCSVに統合

## ライセンス

MIT
