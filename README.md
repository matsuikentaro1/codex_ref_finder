# codex_ref_finder

[OpenAI Codex CLI](https://github.com/openai/codex) を使って学術文献を検索し、結果をCSVに出力する [Claude Code](https://docs.anthropic.com/en/docs/claude-code) カスタムスラッシュコマンドです。

## 特徴

- PubMedを中心に学術文献を検索（書籍・ガイドライン・プレプリントにも対応）
- PMID・著者・タイトル・DOI・要旨などを構造化してCSVに保存
- DOIベースの重複チェックで既存CSVへの安全な追記が可能
- 各文献に最大5つの文脈別知見（`whats_interesting1`〜`5`）を付与
- 複数テーマの並列検索に対応
- **コンテキスト認識検索**: 原稿テキストの文脈を分析し、各引用に最適な論文を自動検索

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
- **コンテキスト認識検索** — 原稿テキストを貼り付けると、引用マーカーの前後の文脈を分析し、「なぜその引用が必要か」を理解したうえで最適な論文を検索

### コンテキスト認識検索の例

```
以下の文章の引用を検索してください:

睡眠不足は扁桃体の過活動と前頭前皮質の抑制機能低下を引き起こし[要引用1]、
これにより情動調節障害が生じる[要引用2]。

CSVファイル: refs.csv
```

Claude Codeが各引用の文脈（主張の内容・必要なエビデンスの種類・適切な検索キーワード）を分析してからCodex CLIに検索を指示するため、的中率の高い文献が得られます。

## ライセンス

MIT
