---
name: codex_ref_finder
description: Search academic references (primarily via PubMed) using OpenAI Codex CLI and export to CSV. Triggers: "論文検索", "文献検索", "文脈検索", "コンテキスト検索", "原稿の引用を探して", "codex_ref_finder", "/codex_ref_finder"
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

## 並列実行時のルール（重要）

複数のcodex検索を並列実行する場合、同じCSVファイルに同時に書き込むとレースコンディション（競合状態）が発生し、データが消失する。これを防ぐため、以下のルールに従う。

### 並列実行の流れ

1. **各codex検索は個別の一時CSVに書き出す**
   - ファイル名: `_tmp_refs_<連番またはテーマ>.csv`（例: `_tmp_refs_01.csv`, `_tmp_refs_circadian.csv`）
   - 各codexプロセスは自分専用の一時CSVのみに書き込む
   - 既存のメインCSVには直接書き込まない

2. **全codex検索が完了した後に、メインプロセスでマージする**
   - メインCSV（例: `refs_YYYYMMDD.csv`）を読み込む
   - 各一時CSVを順番に読み込む
   - DOIで重複チェック → 新規論文は行追加、既存論文はwhats_interesting列に追記
   - マージ完了後、一時CSVファイルを削除する

3. **マージ処理はClaude Code本体（親プロセス）が行う**
   - codex（子プロセス）にはマージさせない
   - マージ処理はPythonスクリプトとして実装し、順次処理する

### 並列実行時の検索リクエスト形式

```
「<検索キーワード>」に関する論文をPubMedから検索して、CSVファイルに保存してください。

保存先: _tmp_refs_<連番>.csv（新規作成）
注意: 既存のメインCSVには書き込まないこと。一時ファイルに新規作成のみ。

CSV列: PubMed_ID, Author, Year, Title, Journal, Volume, Issue, Pages, doi, abstract, whats_interesting1, whats_interesting2, whats_interesting3, whats_interesting4, whats_interesting5
- whats_interesting1に検索文脈に関連した知見を記載

重要: Pythonスクリプトを一時ファイルとして書き出してから実行すること。
```

### マージ処理のPythonテンプレート

```python
import csv
import glob
import os

MAIN_CSV = 'refs_YYYYMMDD.csv'
TMP_PATTERN = '_tmp_refs_*.csv'
FIELDNAMES = ['PubMed_ID', 'Author', 'Year', 'Title', 'Journal', 'Volume', 'Issue', 'Pages', 'doi', 'abstract', 'whats_interesting1', 'whats_interesting2', 'whats_interesting3', 'whats_interesting4', 'whats_interesting5']
WI_COLS = ['whats_interesting1', 'whats_interesting2', 'whats_interesting3', 'whats_interesting4', 'whats_interesting5']

# 1. メインCSV読み込み
rows = []
doi_index = {}
if os.path.exists(MAIN_CSV):
    with open(MAIN_CSV, 'r', encoding='utf-8-sig') as f:
        reader = csv.DictReader(f)
        for row in reader:
            rows.append(row)
            if row.get('doi'):
                doi_index[row['doi']] = len(rows) - 1

# 2. 各一時CSVをマージ
tmp_files = sorted(glob.glob(TMP_PATTERN))
for tmp_file in tmp_files:
    with open(tmp_file, 'r', encoding='utf-8-sig') as f:
        reader = csv.DictReader(f)
        for new_row in reader:
            doi = new_row.get('doi', '')
            if doi and doi in doi_index:
                # 既存論文 → 次の空きwhats_interesting列に追記
                existing = rows[doi_index[doi]]
                for col in WI_COLS:
                    if not existing.get(col):
                        existing[col] = new_row.get('whats_interesting1', '')
                        break
            else:
                # 新規論文 → 行追加
                rows.append(new_row)
                if doi:
                    doi_index[doi] = len(rows) - 1

# 3. メインCSVに書き出し
with open(MAIN_CSV, 'w', encoding='utf-8', newline='') as f:
    writer = csv.DictWriter(f, fieldnames=FIELDNAMES)
    writer.writeheader()
    for row in rows:
        writer.writerow({k: row.get(k, '') for k in FIELDNAMES})

# 4. 一時ファイル削除
for tmp_file in tmp_files:
    os.remove(tmp_file)

print(f'Merged {len(tmp_files)} temp files. Total entries: {len(rows)}')
```

## コンテキスト認識検索モード

原稿テキストに含まれる引用マーカーの前後の文脈を分析し、各引用に最適な論文を検索するモード。キーワードだけでなく「なぜその引用が必要か」を理解してから検索するため、的中率が高い。

### 実行手順

1. ユーザーから原稿テキスト（引用マーカー付き）とCSVファイル名を受け取る
2. **Claude Codeがコンテキスト分析を行う**（codex呼び出し前）:
   a. テキスト中の引用マーカーを検出する
      - 対応パターン: `<sup>N)</sup>`, `[要引用]`, `[要引用N]`, `[ref]`, `[citation needed]`, `N)` など
   b. 各引用マーカーの**直前テキスト**から以下を抽出・推論する:
      - **原文の主張**: どのような事実・知見を述べているか
      - **必要なエビデンスの種類**: RCT、メタアナリシス、コホート研究、レビュー、基礎研究等
      - **検索キーワード（英語）**: PubMed検索に適した英語キーワード
      - **whats_interesting記載方針**: CSVのwhats_interesting列に何を書くべきか
   c. 検索インテントを構造化して出力する（下記テンプレート参照）
3. 検索インテントに基づいてcodexを実行する
   - 引用数が3件以下: 単発実行
   - 引用数が4件以上: 並列実行（既存の並列実行ルールを適用）
4. 生成されたCSVを確認し、結果を報告する

### コンテキスト分析の出力テンプレート

Claude Codeがcodex呼び出し前に生成する中間出力:

```
## 検索インテント

### 引用 [1]
- 原文の主張: 「VLPOを中心とした睡眠促進系とオレキシン神経系を中心とした覚醒促進系の相互抑制機構」
- 必要なエビデンス: レビュー論文または基礎神経科学研究
- 検索キーワード（英語）: "VLPO sleep-wake regulation flip-flop switch hypothalamus"
- whats_interesting記載方針: 睡眠促進系と覚醒促進系の相互抑制機構（フリップフロップモデル）を提唱または実証した研究であることを記載

### 引用 [2]
- 原文の主張: 「扁桃体・前頭前皮質・前帯状皮質のネットワークが情動制御に重要」
- 必要なエビデンス: レビュー論文またはfMRI研究
- 検索キーワード（英語）: "amygdala prefrontal cortex anterior cingulate emotional regulation"
- whats_interesting記載方針: 扁桃体-前頭前皮質ネットワークによる情動制御機構を示した研究であることを記載
```

### 検索リクエストの形式（コンテキスト認識版）

```
以下の学術論文の文脈に基づいて、適切な引用文献をPubMedから検索してCSVに保存してください。

## 検索コンテキスト
原稿の分野: <例: 精神医学、睡眠医学>
対象読者: <例: 精神科医向け日本語総説>

## 検索対象
### 引用 [1]
原文の主張: <主張の要約>
必要なエビデンス: <研究デザインの種類>
検索キーワード: "<英語キーワード>"
whats_interesting記載方針: <記載すべき観点>

### 引用 [2]
（同様の構造）

保存先: <ファイルパス>
保存方法: 既存のCSVファイルがある場合は行を追加して追記、ない場合は新規作成

CSV列: PubMed_ID, Author, Year, Title, Journal, Volume, Issue, Pages, doi, abstract, whats_interesting1, whats_interesting2, whats_interesting3, whats_interesting4, whats_interesting5
- whats_interesting1には上記「whats_interesting記載方針」に基づく具体的な知見を記載
- 各引用に対して最も適切な論文を1〜3件ずつ検索すること
- 各引用の検索キーワードで検索し、原文の主張を最もよく裏付ける論文を選ぶこと

重要な実装指示:
- Pythonスクリプトを一時ファイルとして書き出してからpythonで実行すること
- PowerShellのhere-stringやパイプ経由のPython実行は禁止
```

## 使用例

### 基本的な文献検索（単発実行）
```bash
codex exec --full-auto --sandbox danger-full-access --skip-git-repo-check --cd "C:\path\to\project" "「腸内細菌叢とうつ病」に関する論文をPubMedから検索して、CSVファイルに保存してください。保存先: manuscripts/refs.csv。保存方法: 既存CSVがあればdoiで重複チェックし、既存論文は次の空きwhats_interesting列に追記、新規論文は行追加。CSV列: PubMed_ID, Author, Year, Title, Journal, Volume, Issue, Pages, doi, abstract, whats_interesting1, whats_interesting2, whats_interesting3, whats_interesting4, whats_interesting5。whats_interesting1に検索文脈「腸内細菌叢とうつ病」に関連した知見を記載。重要: Pythonスクリプトを一時ファイルとして書き出してから実行すること。PowerShellのhere-stringやパイプ経由のPython実行は禁止。"
```

### 特定論文を指定して検索
```bash
codex exec --full-auto --sandbox danger-full-access --skip-git-repo-check --cd "C:\path\to\project" "以下の論文をPubMedから検索してCSVに追記してください: (1) Herring et al. 2016 Ann Intern Med suvorexant, (2) Mignot et al. 2022 Lancet Neurol daridorexant。保存先: manuscripts/refs.csv。保存方法: doiで重複チェックし、既存論文は次の空きwhats_interesting列に追記、新規論文は行追加。CSV列: PubMed_ID, Author, Year, Title, Journal, Volume, Issue, Pages, doi, abstract, whats_interesting1, whats_interesting2, whats_interesting3, whats_interesting4, whats_interesting5。whats_interesting列に検索文脈「DORA RCT」に関連した知見を記載。重要: Pythonスクリプトを一時ファイルとして書き出してから実行すること。"
```

### コンテキスト認識検索（原稿テキストから引用を探す）
ユーザー入力:
```
以下の文章の引用を検索してください:

睡眠不足は扁桃体の過活動と前頭前皮質の抑制機能低下を引き起こし[要引用1]、
これにより情動調節障害が生じる[要引用2]。

CSVファイル: refs_20260302.csv
```

Claude Codeの分析（codex呼び出し前の中間出力）:
```
### 引用 [1]
- 原文の主張: 「睡眠不足→扁桃体過活動+前頭前皮質の抑制機能低下」
- 必要なエビデンス: fMRI研究
- 検索キーワード（英語）: "sleep deprivation amygdala prefrontal cortex fMRI"
- whats_interesting記載方針: 睡眠不足による扁桃体-前頭前皮質の機能的解離を示したfMRI研究であることを記載

### 引用 [2]
- 原文の主張: 「睡眠不足による情動調節障害」
- 必要なエビデンス: レビュー論文またはメタアナリシス
- 検索キーワード（英語）: "sleep deprivation emotional dysregulation review"
- whats_interesting記載方針: 睡眠不足と情動調節障害の関連を体系的にまとめた研究であることを記載
```

この分析結果を含めたリクエストでcodexを実行する。

### 並列実行（複数テーマを同時検索）
各codexには個別の一時CSVを指定する:
```bash
# 検索1
codex exec ... "「VLPO sleep regulation」に関する論文を検索。保存先: _tmp_refs_01.csv（新規作成）。..."
# 検索2
codex exec ... "「amygdala prefrontal sleep deprivation」に関する論文を検索。保存先: _tmp_refs_02.csv（新規作成）。..."
# 検索3
codex exec ... "「circadian rhythm bipolar disorder」に関する論文を検索。保存先: _tmp_refs_03.csv（新規作成）。..."
```
全完了後にマージスクリプトを実行してメインCSVに統合する。

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

1. ユーザーから検索リクエストとCSVファイル名を受け取る
2. プロジェクトディレクトリのパスを確認する
3. **検索モードを判断する**
   - **コンテキスト認識検索**: 原稿テキスト（引用マーカー付き）が提供された場合 → コンテキスト分析を先に実行
   - **通常検索**: キーワードや論文情報が明示されている場合 → 直接codexを実行
4. コンテキスト認識検索の場合、Claude Codeが検索インテントを生成する（codex呼び出し前）
5. 並列実行が必要か判断する
   - **単発実行**: メインCSVに直接書き込み
   - **並列実行**: 各codexは `_tmp_refs_<連番>.csv` に個別出力 → 全完了後にマージ
6. 上記コマンド形式でCodexを実行
7. 生成されたCSVファイルの行数を確認して結果を報告

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

### 並列実行時のレースコンディション防止
- **絶対に複数のcodexプロセスが同じCSVファイルに同時書き込みしない**
- 並列実行時は必ず個別の一時CSVに出力し、全完了後にマージする
- マージ処理は親プロセス（Claude Code本体）が順次実行する
