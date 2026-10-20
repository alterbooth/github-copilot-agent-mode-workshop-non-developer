# GitHub Copilotのカスタム指示とカスタムエージェントを使った研究業務活用ワークショップ

## ワークショップの概要

このワークショップは、[**GitHub Copilot Agent Modeで実現する研究業務活用ワークショップ**](textbook-copilot-chat.md)で作成した研究プロジェクトを引き継ぐ続編です。リポジトリに共通指示と役割別のカスタムエージェントを追加します。成果物ファイルを使って、調査、分析、報告、レビューを順に進めます。

### 演習全体のシナリオ

前編では、GitHub Copilotを使ってABC-101の研究資料の調査や実験データの分析、報告書の作成を体験しました。

今後も同様の業務を繰り返し行うことを考えると、毎回細かく指示するのは手間がかかり、調査方法や成果物の品質にもばらつきが生じる可能性があります。

そこで、Copilotに共通のルールを設定し、調査・分析・報告・レビューを担当する専門のAIエージェントを作成します。さらに、各エージェントが成果物を引き継ぎながら作業することで、一連の研究業務を効率的に進められる仕組みを構築します。

### このワークショップで体験すること

1. リポジトリ共通のカスタム指示を定義する
2. VS Codeで認識されるカスタムエージェントを作成する
3. エージェントごとの役割と入出力を分け、成果物ファイルを介して連携する

**注意**  
演習では架空の化合物ABC-101と配布済みの架空データのみを扱います。Copilotが作成した調査・分析・提案は研究判断ではありません。内容の確認と判断は受講者が行います。

---

## 前提条件

- 前編で使用した研究プロジェクトをVS Codeで開いている
- GitHub Copilot Chatを利用できる
- VS CodeでAgent Modeとカスタムエージェントを利用できる
- 前編の手順が未完了の場合は、少なくとも研究資料、`results/assay-results.csv`、`protocols/`、`experiments/`がリポジトリにある
- Python 3とpandasを設定済みの `.venv`

前編から続けるため、新しいリポジトリを作成する必要はありません。作業を始める前に、VS Codeのエクスプローラーで研究プロジェクトのルートを開いていることを確認します。

ターミナルで現在の変更状態を確認します。

```bash
git status
```

前編で作成した未コミットの変更がある場合は、その内容を確認してから作業してください。自分が意図していない変更を削除したり、上書きしたりしないでください。この演習ではリモートへのpushやPull Request作成は行いません。

Pythonの追加演習を行う場合は、設定済みの `.venv` にあるPython 3とpandasを確認します。演習を省略しても、ほかの作業は続行できます。

```text
Windows PowerShell:
.\.venv\Scripts\python.exe --version
.\.venv\Scripts\python.exe -c "import pandas; print(pandas.__version__)"

macOS/Linux:
.venv/bin/python --version
.venv/bin/python -c "import pandas; print(pandas.__version__)"
```

---

## 4つの役割と成果物の流れを確認する

### 目的

エージェントに作業を分担させる理由と、成果物ファイルを使って作業結果を引き継ぐ方法を理解します。

### 役割

| エージェント | 主な作業 | 出力 |
| --- | --- | --- |
| `researcher` | 研究資料の理解、必要に応じた公開情報調査 | `references/research-summary.md` |
| `analyst` | CSV分析、公開情報との比較、追加試験案の作成 | `reports/assay-analysis.md`, `scripts/analyze_assay_variability.py`, `reports/assay-variability.md` |
| `reporter` | 調査・分析結果を統合した報告書の作成 | `reports/executive-summary.md` |
| `reviewer` | 元資料と作成済み成果物の照合 | `reports/review-comments.md` |

作業の流れは次のとおりです。

```text
研究資料 ──> researcher ──> research-summary.md
                               │
CSV・調査結果 ──> analyst <────┘
                         │
                         └──> assay-analysis.md
                                  │
                   reporter <────┘
                       │
                       └──> executive-summary.md
                                  │
                   reviewer <────┘ + 元資料 + 作成済み成果物
                       │
                       └──> review-comments.md
```

Pythonの演習では、analystが元CSVから反復測定のばらつきを集計し、`scripts/analyze_assay_variability.py`と`reports/assay-variability.md`を作成します。

会話の履歴だけに頼らず、各エージェントの作業結果を成果物ファイルに保存し、次のエージェントが入力として使えるようにします。

---

## Step 1：共通カスタム指示を作成する

### 目的

研究資料の扱い方や根拠の示し方を、毎回のプロンプトではなくリポジトリ共通の指示として定義します。

### シナリオ：Copilotの回答品質を統一したい

前編ではCopilotを使って研究資料の調査や分析を行いました。今後も継続的に活用するため、根拠の明示や推測の扱いなど、毎回同じ指示を入力する手間をなくしたいと考えています。

そこで、研究プロジェクト共通のルールを定義し、Copilotが一貫した方針で作業できるようにします。

### 作業

1. VS Codeのエクスプローラーでリポジトリ直下に`.github`フォルダーを作ります。
2. `.github/copilot-instructions.md`を作成します。
3. 次の内容をファイルに記載し、保存します。
    ```markdown
    # 共通方針

    あなたは架空の製薬研究プロジェクトを支援するAIエージェントです。

    ## 対象データ

    - リポジトリ内の研究資料
    - 実験ノート
    - CSV形式の実験結果
    - Web上の公開情報

    ## 必ず守ること

    1. 確認できた事実と解釈・推測・提案を区別する。
    2. 主張には根拠となるファイル名、または公開情報の出典を示す。
    3. 確認できない内容を補完・推測しない。「未確認」または「情報なし」と明記する。
    4. 公開情報を使う場合は、出典、公開日、対象、研究条件を記録する。
    5. 公開研究の結果をABC-101の結果として扱わない。
    6. 生成結果を研究判断そのものとして扱わず、研究上の結論を断定しない。
    7. 作成を依頼された成果物はリポジトリ内の指定ファイルに保存する。
    8. 他のエージェントへ渡す情報は、会話だけでなく成果物ファイルに記録する。

    このワークショップでは架空データのみを扱う。実在する機密情報や未公開の研究データを使用しない。
    ```
4. ここまでの変更をコミットします。コミットメッセージは `共通Copilot指示を追加` とします。

### チェックポイント

- [ ] `.github`フォルダに`copilot-instructions.md`がある
- [ ] 事実と解釈の区別、根拠の記録、架空データの使用が含まれている

---

## Step 2：researcherエージェントを作成する

### 目的

研究資料の調査に専念するエージェントを定義し、入力・出力と作業範囲を明確にします。

### シナリオ：研究資料の調査を専門に行うエージェントを用意したい

研究資料の整理や公開情報の調査は、今後も繰り返し発生する業務です。そのたびに調査方法や出力形式を細かく指示するのは手間がかかります。

そこで、研究資料の調査に特化した researcher エージェントを作成し、必要なときに同じ方針で調査を依頼できるようにします。

### 作業

1. `.github/agents`フォルダーを作成します。
2. `.github/agents/researcher.agent.md`を作成し、以下を記載します。
    ```markdown
    ---
    name: researcher
    description: 研究資料を確認し、根拠付きの研究概要と公開情報調査結果を作成します。
    ---

    # Mission

    研究資料を理解し、必要に応じて関連する公開情報を調査する。

    # Inputs

    - README.md
    - protocols/*
    - experiments/*
    - results/*.csv
    - references/internal-reference.md

    # Output

    - references/research-summary.md

    # Instructions

    1. リポジトリ内の研究資料を確認し、研究目的、実験条件、既存結果を整理する。
    2. 数値や条件を記載する場合は、元資料と照合し、根拠ファイルを示す。
    3. 必要に応じてABC-101に関連する公開情報を調査する。ABC-101は架空の化合物であり、実在する化合物として扱わない。
    4. 公開情報は出典、公開日、対象、研究条件を記録し、ABC-101の情報と類似研究の情報を区別する。
    5. 信頼できる情報を確認できない場合は、その旨を記載し、推測で補わない。
    6. 結果を指定された出力ファイルに保存する。

    # Output Structure

    - 研究目的
    - 実験条件
    - 既存結果
    - 公開情報調査結果
    - 出典一覧
    - 未確認事項
    ```
3. ここまでの変更をコミットします。コミットメッセージは `researcherエージェントを追加` とします。

### チェックポイント

- [ ] ファイル名が`researcher.agent.md`である
- [ ] ファイル先頭に`name`と`description`を含むYAML frontmatterがある
- [ ] 出力先が`references/research-summary.md`と明記されている

---

## Step 3：残りのエージェントを作成する

### 目的

残り3つの役割を定義し、前のエージェントが保存した成果物を入力として受け取れるようにします。

3つのエージェントの定義を、指定されたファイル名で`.github/agents/`に保存します。

### シナリオ：調査から報告までの作業を役割分担したい

研究業務には、情報収集だけでなく、実験データの分析、報告書の作成、内容の確認など、さまざまな作業があります。

そこで、分析担当の analyst、報告担当の reporter、レビュー担当の reviewer を追加し、それぞれが専門の役割を担えるようにします。

### analystエージェント

ファイル名：`.github/agents/analyst.agent.md`

```markdown
---
name: analyst
description: CSVの数値を整理し、調査結果との比較と根拠付きの追加試験案を作成します。
---

# Mission

社内研究と公開情報を比較し、分析結果と追加試験案を作成する。

# Inputs

- references/research-summary.md
- results/*.csv
- reports/assay-statistics.md（存在する場合。派生資料として扱う）
- protocols/*
- experiments/*

# Output

- reports/assay-analysis.md
- scripts/analyze_assay_variability.py
- reports/assay-variability.md

# Instructions

1. 調査結果とCSVの対象範囲、単位、条件を確認する。
2. CSVに記録された数値と、数値から導く解釈を分けて記載する。
3. CSVの値を引用する場合は、元の値・単位・条件を変更せず、根拠ファイルを示す。
4. 内部研究と公開研究は、比較可能な評価軸と条件を明示して比較する。条件が異なる場合は直接比較できないことを記載する。
5. 不足する評価項目と追加試験案には、それぞれ根拠と想定リスクを示す。研究上の結論は断定しない。
6. 結果を指定された出力ファイルに保存する。
7. Pythonによる反復測定のばらつき分析を依頼された場合は、pandasとPython標準ライブラリのpathlibを使い、`qc_flag`が`PASS`の行だけを対象に、化合物・濃度ごとの有効件数、標本標準偏差、最小値、最大値、範囲を集計する。欠損値は測定項目ごとに除外し、有効件数を示す。入力CSVを変更せず、専用スクリプトとレポートを指定先に保存する。数値の事実と解釈を区別し、スクリプトを利用者の確認前に実行しない。

# Output Structure

- 分析対象
- 事実（CSV）
- 解釈
- 公開研究との比較
- 不足評価項目
- 追加試験案
- 根拠
- リスク
```

### reporterエージェント

ファイル名：`.github/agents/reporter.agent.md`

```markdown
---
name: reporter
description: 調査・分析の成果物を統合し、研究責任者向け報告書を作成します。
---

# Mission

調査結果と分析結果を統合し、研究責任者が確認できる報告書を作成する。

# Inputs

- references/research-summary.md
- reports/assay-analysis.md
- reports/assay-variability.md（存在する場合。元CSVから作成した派生資料として扱う）

# Output

- reports/executive-summary.md

# Instructions

1. 入力ファイルの記述と根拠を確認し、入力にない事実を追加しない。
2. 確認済みの事実、解釈、提案、未確認事項を明確に区別する。
3. 重要な主張ごとに根拠となるファイル名または公開情報の出典を示す。
4. 現時点の要約を冒頭に置く。研究継続可否などの研究判断を断定しない。
5. `reports/assay-variability.md`がある場合は、反復測定のばらつきを記述的な事実として扱い、研究上の結論に結び付けない。元CSVと照合し、根拠として示す。
6. 結果を指定された出力ファイルに保存する。

# Output Structure

- エグゼクティブサマリー
- 結論（現時点で言える範囲の要約）
- 事実
- 解釈
- 提案
- リスク
- 未確認事項
- 次回アクション
```

### reviewerエージェント

ファイル名：`.github/agents/reviewer.agent.md`

```markdown
---
name: reviewer
description: 元資料と成果物を照合し、根拠不足や不整合などの問題点を記録します。
---

# Mission

成果物の整合性と根拠を確認し、問題点のみをレビューコメントとして報告する。

# Inputs

- README.md
- protocols/*
- experiments/*
- results/*.csv
- references/research-summary.md
- reports/assay-analysis.md
- reports/assay-statistics.md（存在する場合）
- reports/assay-variability.md（存在する場合）
- reports/executive-summary.md
- 作成済みの成果物ファイル

# Output

- reports/review-comments.md

# Instructions

1. 元資料との不整合、数値の誤り、出典漏れ、過度な推測を確認する。
2. 事実と解釈の混在、報告書内の矛盾、根拠不足の主張を確認する。
3. `reports/assay-variability.md`がある場合は、元CSVの`PASS`行を使って集計値を照合し、統計量と有効件数を確認する。
4. 問題が確認できた箇所のみを指摘し、対象ファイルと理由を示す。
5. 修正案を示す場合は、元資料または指示に基づく理由を添える。
6. 問題が見つからない項目を、問題があるかのように記載しない。
7. 自身で研究上の判断や結論を追加しない。
8. 結果を指定された出力ファイルに保存する。

# Output Structure

- Review Summary
- Critical Issues
- Moderate Issues
- Minor Issues
- 修正推奨事項
```

ここまでの変更をコミットします。コミットメッセージは `分析・報告・レビューエージェントを追加` とします。

### チェックポイント

- [ ] 3ファイルとも`.agent.md`で終わっている
- [ ] それぞれ異なる役割、入力、出力が定義されている
- [ ] analystは数値と解釈を区別している
- [ ] reporterは研究判断を断定しない
- [ ] reviewerは問題点のみを指摘し、研究判断を追加しない

---

## Step 4：カスタムエージェントが認識されたことを確認する

### 目的

作成したエージェント定義をVS Codeで選択できることを確認します。

### シナリオ：作成したAIエージェントが正しく動くか確認したい

研究業務で使い始める前に、作成した4つのエージェントがCopilotから利用できる状態になっているか確認する必要があります。

まずは researcher に簡単な調査を依頼し、設定した役割や指示が反映されているか確認します。

### 作業

1. Copilot Chatを開き、Agent Modeにします。
2. エージェント選択メニューを開き、`researcher`、`analyst`、`reporter`、`reviewer`が一覧にあるか確認します。  
    ![vscode-copilot-custom-agents](img/agents/vscode-copilot-custom-agents.png)
3. 一覧に表示されない場合は、ファイル名、配置場所、frontmatterの区切り（`---`）、`name`と`description`を確認します。保存後も反映されない場合は、VS Codeの表示を更新するか、ウィンドウを再読み込みして、もう一度確認します。
4. `researcher`エージェントを選び、資料を確認するだけの短い依頼を実行します。
    ```text
    研究資料を確認し、研究目的と候補化合物を根拠ファイル名付きで簡潔にまとめてください。
    この確認ではファイルを変更しないでください。
    ```

### チェックポイント

- [ ] 4つのエージェントが選択肢に表示された
- [ ] 選択したエージェントの役割に沿った応答が得られた
- [ ] 根拠として示されたファイルを開き、記述を照合した

---

## Step 5：researcherで調査結果を保存する

### 目的

最初のエージェントに資料調査を依頼し、後続作業の入力となる成果物を作成します。

### シナリオ：研究資料の調査を専門エージェントに任せたい

4つのエージェントが正しく認識され、researcher の動作確認もできました。

そこで、今度は実際の研究業務を想定し、ABC-101の研究状況や関連する公開情報の調査を依頼します。調査結果は、後続の分析業務でも活用できるようにファイルに保存します。

### 作業

1. `researcher`エージェントを選びます。
2. 次の依頼を入力します。
    ```text
    ABC-101の研究資料を確認し、研究目的、実験条件、既存結果を根拠となるファイル名付きで整理してください。
    必要に応じて関連する公開情報も調査してください。ABC-101は架空の化合物として扱い、実在する化合物の情報と混同しないでください。
    公開情報は出典、公開日、対象、研究条件を記録してください。確認できない事項は推測で補わず、未確認と明記してください。
    references/research-summary.mdに保存してください。
    ```
3. Copilotが提示する変更内容を確認してから適用します。
4. `references/research-summary.md`を開き、元資料と公開情報の出典を照合します。
5. ここまでの変更をコミットします。コミットメッセージは `研究資料の調査結果を保存` とします。

### チェックポイント

- [ ] 研究目的、実験条件、既存結果に根拠ファイル名がある
- [ ] 公開情報に出典、公開日、対象、条件がある
- [ ] ABC-101の架空設定と公開研究が混同されていない
- [ ] 未確認事項が推測で埋められていない
- [ ] `references/research-summary.md`が保存されている

---

## Step 6：analystでCSV分析と追加試験案を作成する

### 目的

調査結果とCSVを照合し、確認済み数値、解釈、提案を分けた分析を作成します。

### シナリオ：調査結果と実験データから、次の研究方針を検討したい

researcher による調査が完了したので、次は実験データと照らし合わせ、候補化合物の特徴や追加で確認すべき事項を整理します。

そこで、analyst に分析を依頼し、公開情報の調査結果と実験結果を比較して、不足する評価項目や追加試験案を整理します。また、測定結果のばらつきも把握するため、Pythonによる集計スクリプトを作成してもらい、内容を確認してから実行し、集計結果を元データと照合します。

### 作業

1. `analyst`エージェントを選びます。
2. 次の依頼を入力します。
    ```text
    references/research-summary.mdとresults/assay-results.csv、関連するprotocols/およびexperiments/を確認してください。
    CSVの数値・単位・実験条件を元ファイルと照合し、確認済みの事実と数値からの解釈を分けてください。
    内部研究と公開情報を比較するときは、比較軸と条件を明記し、条件が異なる場合は直接比較できないことを示してください。
    reports/assay-statistics.mdがある場合は補助資料として参照し、元CSVと照合してください。統計レポートを元データとして扱わないでください。
    不足する評価項目と追加試験案には、それぞれ根拠と想定リスクを記載してください。
    研究上の結論は断定せず、reports/assay-analysis.mdに保存してください。
    ```
3. 提案された変更を確認して適用します。
4. 分析ファイルに記載された数値を`results/assay-results.csv`と照合します。

#### pandasで反復測定のばらつきを集計する

反復測定のばらつきを集計する専用スクリプト `scripts/analyze_assay_variability.py` とレポート `reports/assay-variability.md` を作成します。  
実行環境には設定済みの `.venv` を指定してください。

`analyst`エージェントに次の指示を入力します。

```text
results/assay-results.csvを読み取り、化合物・濃度ごとに反復測定のばらつきを集計するPythonスクリプトを作成してください。

スクリプトの保存先:
scripts/analyze_assay_variability.py

レポートの保存先:
reports/assay-variability.md

条件:
- pandasとPython標準ライブラリのpathlibを使う。pandas以外の追加パッケージやWebアクセスは使わない
- 入力CSVは読み取り専用とし、リポジトリのルートから実行できるようにする
- experiment_id、compound_id、concentration_uM、replicate、qc_flag、およびcell_viability_pct、il6_inhibition_pct、tnf_alpha_inhibition_pct、cytotoxicity_pctを必要列として確認する
- concentration_uMと測定値の列は数値として扱う。空欄以外で数値に変換できない値があれば、列名とCSV上の行番号が分かるエラーを表示して失敗させる
- qc_flagがPASS以外の行は統計量の計算から除外し、除外件数とexperiment_id、compound_id、concentration_uM、replicate、qc_flagをレポートする
- PASSの行について、compound_idとconcentration_uMごとに測定項目別の有効件数、標本標準偏差（ddof=1）、最小値、最大値、範囲（最大値と最小値の差）を計算する
- 欠損測定値は項目ごとに計算から除外し、有効件数に反映する
- 測定値が2件未満の項目は標本標準偏差と範囲を「算出不可」とし、1件だけの値をばらつきがないと解釈しない
- 集計結果、対象ファイル、QC行の扱い、集計方法をreports/assay-variability.mdに保存する。reportsフォルダーがない場合は作成する
- Markdown表はPython標準ライブラリで生成し、追加パッケージに依存しない
- 出力は数値の集計事実に限り、有効性、安全性、研究上の結論を判定しない
- 既存のCSV、前編のスクリプトやレポート、指定出力先以外のファイルを変更しない
```

スクリプトを実行する前に、入力がCSV、出力先が `reports/assay-variability.md` に限定されていることを確認します。リポジトリのルートから、設定済みの `.venv` のPythonで実行します。

```text
Windows PowerShell: .\.venv\Scripts\python.exe scripts/analyze_assay_variability.py
macOS/Linux:        .venv/bin/python scripts/analyze_assay_variability.py
```

`reports/assay-variability.md`の値を元CSVのうち`qc_flag`が`PASS`の行と照合します。測定項目ごとの有効件数、標本標準偏差、最小値、最大値、範囲を確認し、QC除外行が統計量に含まれていないことを確かめます。数値の違いは解釈で埋めず、原因を確認してください。

```text
reports/assay-variability.mdの各集計値をresults/assay-results.csvと照合してください。
qc_flagがPASSの行だけを使い、化合物・濃度・測定項目ごとの有効件数、標本標準偏差、最小値、最大値、範囲を確認してください。
不一致があれば、該当する行と列を示してください。元CSVを変更せず、数値の事実と解釈・提案を区別してください。
```

ここまでの変更をコミットします。コミットメッセージは `実験結果の分析を保存` とします。  
なお、Copilotでコードを生成する際に `%SystemDrive%` フォルダがプロジェクト内に作成されることがあります。  
本演習には不要なフォルダであるため、コミットせず削除して構いません。

### チェックポイント

- [ ] 事実、解釈、追加試験案が区別されている
- [ ] 比較できない条件差が明示されている
- [ ] 不足項目と試験案に根拠とリスクがある
- [ ] `reports/assay-analysis.md`が保存されている
- [ ] 専用スクリプトとばらつきレポートが保存され、PASSの測定値と照合できている

---

## Step 7：reporterで報告書を作成する

### 目的

保存済みの調査・分析結果を統合し、研究責任者向けの報告書を作成します。

### シナリオ：調査・分析結果を研究責任者に報告したい

researcher による調査と analyst による実験データの分析が完了し、研究責任者への報告に必要な材料がそろいました。

そこで、reporter に報告書の作成を依頼し、これまでの調査・分析結果を統合して、確認済みの事実や解釈、今後の提案、リスクなどを根拠とともに整理します。

### 作業

1. `reporter`エージェントを選びます。
2. 次の依頼を入力します。
    ```text
    references/research-summary.mdとreports/assay-analysis.mdを基に、ABC-101の研究責任者向け報告書を作成してください。
    reports/assay-variability.mdがある場合は補助資料として参照し、元CSVのPASS行と照合してください。ばらつきの記述は数値の事実にとどめ、研究上の結論を推測しないでください。
    現時点で言える範囲の要約を最初に置き、確認済みの事実、解釈、提案、リスク、未確認事項を区別してください。
    重要な主張には根拠となるファイル名または公開情報の出典を示してください。入力にない情報は追加せず、研究継続可否などを断定しないでください。
    reports/executive-summary.mdに保存してください。
    ```
3. 提示された変更を確認して適用します。
4. 報告書の記述を調査・分析ファイルまでさかのぼり、根拠を確認します。
5. ここまでの変更をコミットします。コミットメッセージは `研究責任者向け報告書を作成` とします。

### チェックポイント

- [ ] 冒頭の要約が入力成果物の範囲を超えていない
- [ ] 事実、解釈、提案、未確認事項が区別されている
- [ ] 数値と出典を元資料で確認できる
- [ ] 研究判断を断定していない
- [ ] `reports/executive-summary.md`が保存されている

---

## Step 8：reviewerで独立レビューを行う

### 目的

作成担当とは別の役割に、元資料と報告書の不整合や根拠不足を確認させます。

### シナリオ：報告書を提出する前に、内容の正確性を確認したい

reporter によって研究責任者向けの報告書が完成しました。しかし、AIが作成した内容には、数値の誤りや根拠不足、過度な推測が含まれている可能性があります。

そこで、reviewer にレビューを依頼し、元の研究資料と各成果物を照合して問題点を洗い出します。指摘内容を確認し、必要な修正を判断します。

### 作業

1. `reviewer`エージェントを選びます。
2. 次の依頼を入力します。
    ```text
    README.md、protocols/、experiments/、results/、references/research-summary.md、reports/assay-analysis.md、reports/executive-summary.mdを照合してください。
    reports/assay-statistics.mdがある場合は、元CSVおよびreports/assay-analysis.mdの数値と照合してください。統計レポートは派生資料として扱い、元CSVを基準にしてください。
    reports/assay-variability.mdがある場合は、元CSVのqc_flagがPASSの行だけを使って有効件数、標本標準偏差、最小値、最大値、範囲を照合してください。
    数値の誤り、出典漏れ、元資料との不整合、過度な推測、事実と解釈の混在、報告書内の矛盾、根拠不足の主張を確認してください。
    問題が確認できた箇所のみ、対象ファイル、問題、理由、必要に応じた修正案を記載してください。研究上の判断や結論は追加しないでください。
    reports/review-comments.mdに保存してください。
    ```
3. `reports/review-comments.md`を読み、各指摘を元資料で検証します。
4. 指摘を自動的にすべて採用せず、修正するかどうかを受講者が判断します。必要な場合は該当ファイルを修正し、内容を再確認します。
5. ここまでの変更をコミットします。コミットメッセージは `レビュー結果を保存` とします。

### チェックポイント

- [ ] レビュー指摘に対象ファイルと理由がある
- [ ] 元資料で指摘の正しさを確認した
- [ ] 根拠のない指摘や研究判断が追加されていない
- [ ] （Python集計がある場合）元CSVを基準に集計値と報告内容を照合した
- [ ] 修正要否を受講者自身が判断した
- [ ] `reports/review-comments.md`が保存されている

---

## Step 9：ローカルGitの履歴を確認する

### 目的

各Stepでコミットした変更と、作業ツリーの状態を確認します。

### シナリオ：一連の作業と変更履歴を確認したい

4つのエージェントを活用した調査・分析・報告・レビューが完了しました。

最後に、Gitの履歴を確認し、各ステップで行った変更が記録されていることと、未記録の変更が残っていないことを確認します。

### 作業

次のコマンドで作業ツリーとコミット履歴を確認します。

```bash
git status --short --branch
git log --oneline
```

各Stepのコミットメッセージが履歴に表示され、未コミットの変更が残っていないことを確認します。  
`git log` コマンドは `q` キーで終了します。

### チェックポイント

- [ ] コミット履歴に各Stepのメッセージが表示される
- [ ] 作業ツリーに未コミットの変更が残っていない

---

## 最終確認

ワークショップの終了時に、次の成果物が作成されていることを確認します。

```text
research-project/
├── .github/
│   ├── copilot-instructions.md
│   └── agents/
│       ├── researcher.agent.md
│       ├── analyst.agent.md
│       ├── reporter.agent.md
│       └── reviewer.agent.md
├── references/
│   └── research-summary.md
├── reports/
│   ├── assay-analysis.md
│   ├── assay-statistics.md
│   ├── assay-variability.md
│   ├── executive-summary.md
│   └── review-comments.md
└── scripts/
    ├── analyze_assay_variability.py
    └── summarize_assay.py
```

### 成果物チェック

- [ ] 共通指示と4つのカスタムエージェント定義がリポジトリ内に保存されている
- [ ] 4つのカスタムエージェントがVS Codeで選択できる
- [ ] `references/research-summary.md`に調査結果と参照元が記録されている
- [ ] `reports/assay-analysis.md`に実験結果の分析が記録されている
- [ ] `scripts/summarize_assay.py`と`reports/assay-statistics.md`の集計内容を確認した
- [ ] `scripts/analyze_assay_variability.py`と`reports/assay-variability.md`を作成し、PASS行の集計内容を確認した
- [ ] `reports/executive-summary.md`に研究責任者向けの報告が記録されている
- [ ] `reports/review-comments.md`にレビュー結果が記録されている

---

## 利用上の注意

- 実在する機密情報や未公開の研究データを入力しないでください。
- Web上の情報は、参照元、公開日、研究対象、研究条件を確認してください。
- Copilotが生成した内容を、そのまま研究判断に使用しないでください。
- 有効性、安全性、作用機序に関する記述は、必ず根拠を確認してください。
- 確認済みの事実、AIによる解釈、AIによる提案を区別してください。
- ローカルPCへ保存する情報も、所属組織の情報管理ルールに従ってください。
- 指示ファイルやエージェント定義は、誤りを完全に防ぐ保証ではありません。生成結果と差分の確認は毎回人が行ってください。