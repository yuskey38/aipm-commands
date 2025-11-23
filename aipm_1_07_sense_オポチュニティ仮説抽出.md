# : オポチュニティ仮説抽出（発散→圧縮）

## 最初に質問（実行前に回答してください）
- 参照ソース（任意・複数可）：Senseの調査結果（例：`sense_customer_research.md`、`sense_competitor_research.md`、`draft_interview_analysis_*.md`、`draft_research_summary.md` など）
- 重視する観点（任意・例：顧客痛点/未充足/競合弱点/技術トレンド/規制変化）
- 出力数（任意・既定: 10件）

## 目的
Senseで収集したインサイトから、機会（Opportunity）仮説を構造化して抽出します。各仮説は根拠・期待価値・検証案を含みます。

## 実行手順（Rules Steps）
```yaml
- trigger: "(sense_オポチュニティ抽出|Opportunity Extraction)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}", "{{find_files(patterns=['**/sense_customer_research.md','**/sense_competitor_research.md','**/draft_interview_analysis_*.md','**/draft_research_summary.md'])}}"]
      instructions: |
        スレッドで直近に作られたSense系ファイルパスをsources初期値として提案してください。
        また、countは10をデフォルトに。
        出力は `{"sources": "path1,path2,...", "count": 10}` のJSON1件のみ。
      store_as: "prefill"
    - name: "collect_inputs"
      action: "ask_questions_with_template"
      template: |
        === 入力テンプレ ===
        1) 参照ソース（カンマ区切り。未指定なら既定パターンを自動探索）
        → {{prefill.sources}}
        2) 重視する観点（例: 顧客痛点/未充足/競合弱点/技術トレンド/規制変化）
        →
        3) 出力数（例: 10）
        → {{prefill.count}}
        ======================

    - name: "wait_inputs"
      action: "wait_for_all_answers"

    - name: "confirm"
      action: "confirm"
      message: "Sense調査のインサイトからオポチュニティ仮説を抽出します。よろしいですか？"

    - name: "ensure_output_dir"
      action: "execute_shell"
      command: "mkdir -p {{patterns.flow_date}}/1_sense/07_オポチュニティ仮説抽出"

    - name: "extract_opportunities"
      action: "analyze"
      data: ["{{read_files(inputs.sources | default: find_files(patterns=['**/sense_customer_research.md','**/sense_competitor_research.md','**/draft_interview_analysis_*.md','**/draft_research_summary.md']) )}}"]
      instructions: |
        参照ドキュメントから、以下の形式で最大{{inputs.count | default:10}}件のオポチュニティ仮説を抽出してください。
        - id: OP001  # 連番
          title: 短い表題
          evidence: [引用/出典/根拠の要約]
          customer_pain: 顧客の痛み/未充足
          competitor_gap: 競合の弱点/非対応
          trend_trigger: 技術/市場/規制トレンド
          expected_value: 期待する価値/効果
          risk_note: リスク/前提
          quick_test: 最小検証案（MVPで測る指標も）
      store_as: "op_list"

    - name: "write_opportunities_yaml"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/1_sense/07_オポチュニティ仮説抽出/sense_opportunities.yaml"
      content: |
        opportunities:
          {{op_list}}

    - name: "write_opportunities_md"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/1_sense/07_オポチュニティ仮説抽出/sense_opportunities.md"
      content: |
        # オポチュニティ仮説（Sense）
        {{#each op_list}}
        - **{{id}}**: {{title}}
          - 根拠: {{evidence}}
          - 顧客痛点: {{customer_pain}}
          - 競合ギャップ: {{competitor_gap}}
          - トレンド: {{trend_trigger}}
          - 期待価値: {{expected_value}}
          - リスク: {{risk_note}}
          - 最小検証案: {{quick_test}}
        {{/each}}

    - name: "notify"
      action: "display"
      content: |
        ✅ `sense_opportunities.yaml` / `sense_opportunities.md` を作成しました。
        次は Focus フェーズへ：優先順位付けとプロダクト定義に進みましょう。
```

## 次に実行
- `2_focus/01_プロダクト定義` → `2_focus/02_市場規模推定` → `2_focus/03_ラフロードマップ作成` → `2_focus/04_OKR作成`


