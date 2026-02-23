# codex_ref_finder

[OpenAI Codex CLI](https://github.com/openai/codex) を使って学術文献を検索し、結果をCSVに出力する [Claude Code](https://docs.anthropic.com/en/docs/claude-code) カスタムスラッシュコマンドです。PubMedを中心に文献を検索し、PMID・著者・タイトル・ジャーナル・DOI・要旨などを構造化してCSVに保存します。

## 特徴

- Codex CLI経由でPubMedなどの学術データベースを検索
- 標準化されたCSV列で結果を出力
- DOIベースの重複チェックにより、既存CSVへの安全な追記が可能
- 各文献に最大5つの文脈別知見（`whats_interesting1`〜`5`）を付与

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
cp codex_ref_finder/.claude/commands/codex_ref_finder.md /path/to/your/project/.claude/commands/
```

または `.claude/commands/codex_ref_finder.md` を直接ダウンロードして配置してください。

## 使い方

Claude Code上でスラッシュコマンドを実行します：

```
/codex_ref_finder
```

## CSV列定義

| 列名 | 説明 |
|---|---|
| PubMed_ID | PMID |
| Author | 著者名（セミコロン区切り） |
| Year | 発行年 |
| Title | 論文タイトル |
| Journal | ジャーナル名 |
| Volume | 巻番号 |
| Issue | 号番号 |
| Pages | ページ範囲 |
| doi | DOI |
| abstract | 要旨 |
| whats_interesting1〜5 | 検索文脈に関連した知見 |

## ライセンス

MIT
