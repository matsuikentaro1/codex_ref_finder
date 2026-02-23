---
name: codex_ref_finder
description: Search academic references (primarily via PubMed) using OpenAI Codex CLI and export to CSV. Triggers: "論文検索", "文献検索", "codex_ref_finder", "/codex_ref_finder"
---
# Codex Ref Finder
Codex CLIを使用して学術文献を検索し、結果をCSVファイルに保存するスキル。PubMedを主な検索ソースとするが、PubMed未掲載の文献（書籍、ガイドライン、プレプリント等）も対象とする。

## 実行コマンド
```bash
codex exec --full-auto --sandbox danger-full-access --skip-git-repo-check --cd <project_directory> "<request>"
```

### コマンドオプションについて
- `--sandbox danger-full-access`: ファイル書き込みとネットワークアクセスの両方が必要なため使用（他の選択肢: `read-only`, `workspace-write`）
- `--skip-git-repo-check`: Gitリポジトリ外のディレクトリでも実行可能にするため使用

## 検索リクエストの形式

```
「<検索キーワード>」に関する論文をPubMedから検索して、CSVファイルに保存してください。

保存先: <ファイルパス>
保存方法: 既存のCSVファイルがある場合は行を追加して追記、ない場合は新規作成

CSV列: PubMed_ID, Author, Year, Title, Journal, Volume, Issue, Pages, doi, abstract, whats_interesting1, whats_interesting2, whats_interesting3, whats_interesting4, whats_interesting5
- PubMed_IDはPMID（PubMed識別子）。PubMed未掲載の文献はエムダッシュ（—）を入れる
- whats_interesting1〜whats_interesting5は検索文脈に関連した主要な知見を記載するための列
- 新規論文の場合はwhats_interesting1に記載
- 既存の論文（doiで照合）に追記する場合は、次の空きwhats_interesting列に記載

重要な実装指示:
- Pythonスクリプトを一時ファイル（例: _tmp_search.py）として書き出してからpythonで実行すること
- PowerShellのhere-string（@'...'@）やパイプ経由のPython実行は禁止（エスケープエラーの原因）
- abstractやwhats_interesting内のカンマ・引用符はcsv.DictWriterが自動処理するので手動エスケープ不要
- 既存CSVに追記する場合、まず既存データを読み込み、doiで重複チェックを行うこと
- 重複論文が見つかった場合は行を追加せず、既存行の次の空きwhats_interesting列に新しい知見を追記すること
```

## 使用例

### 基本的な文献検索
```bash
codex exec --full-auto --sandbox danger-full-access --skip-git-repo-check --cd "C:\path\to\project" "「腸内細菌叢とうつ病」に関する論文をPubMedから検索して、CSVファイルに保存してください。保存先: manuscripts/refs.csv。保存方法: 既存CSVがあればdoiで重複チェックし、既存論文は次の空きwhats_interesting列に追記、新規論文は行追加。CSV列: PubMed_ID, Author, Year, Title, Journal, Volume, Issue, Pages, doi, abstract, whats_interesting1, whats_interesting2, whats_interesting3, whats_interesting4, whats_interesting5。whats_interesting1に検索文脈「腸内細菌叢とうつ病」に関連した知見を記載。重要: Pythonスクリプトを一時ファイルとして書き出してから実行すること。PowerShellのhere-stringやパイプ経由のPython実行は禁止。"
```

### 特定論文を指定して検索
```bash
codex exec --full-auto --sandbox danger-full-access --skip-git-repo-check --cd "C:\path\to\project" "以下の論文をPubMedから検索してCSVに追記してください: (1) Herring et al. 2016 Ann Intern Med suvorexant, (2) Mignot et al. 2022 Lancet Neurol daridorexant。保存先: manuscripts/refs.csv。保存方法: doiで重複チェックし、既存論文は次の空きwhats_interesting列に追記、新規論文は行追加。CSV列: PubMed_ID, Author, Year, Title, Journal, Volume, Issue, Pages, doi, abstract, whats_interesting1, whats_interesting2, whats_interesting3, whats_interesting4, whats_interesting5。whats_interesting列に検索文脈「DORA RCT」に関連した知見を記載。重要: Pythonスクリプトを一時ファイルとして書き出してから実行すること。"
```

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
| whats_interesting1 | 検索文脈に関連した知見1 |
| whats_interesting2 | 別の検索文脈での知見2 |
| whats_interesting3 | 知見3 |
| whats_interesting4 | 知見4 |
| whats_interesting5 | 知見5 |

**注意**: 列名に特殊文字を含めない。アポストロフィはPowerShellのエスケープエラーの原因となる。

## 実行手順

1. ユーザーから検索キーワード（またはリクエスト内容）とCSVファイル名を受け取る
2. プロジェクトディレクトリのパスを確認する
3. 上記コマンド形式でCodexを実行
4. 生成されたCSVファイルの行数を確認して結果を報告

## 注意事項

### PowerShellエスケープ問題の回避
Codexは内部でPowerShellを使用するため、以下のパターンでエラーが頻発する：
- `@'...'@`（here-string）内の引用符
- パイプ（`|`）経由でのPython実行
- 長い文字列リテラル内の特殊文字

**解決策**: プロンプトで「Pythonスクリプトを一時ファイルとして書き出してから実行」を明示的に指示する。

### 重複論文の扱い
- doiで重複チェックを行い、同じ論文は同一行に保持する
- 異なる検索文脈での知見はwhats_interesting1〜whats_interesting5の空き列に順次追記
- 重複チェックの流れ:
  1. 既存CSVを読み込み、doiの一覧を取得
  2. 新規検索結果のdoiが既存に含まれていれば、その行の次の空きwhats_interesting列に知見を追記
  3. 新規論文（doiが一致しない）は新しい行として追加
- doiが空欄の論文は常に新規行として追加する
