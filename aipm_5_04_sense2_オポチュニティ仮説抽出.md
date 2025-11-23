# : オポチュニティ仮説抽出（検証結果から次の機会）

## 目的
ユーザーテスト結果とDelivery成果物を踏まえ、次サイクルで取り組むべき機会（Opportunity）仮説を抽出します。根拠はSense2サマリー/個別分析/計測を優先します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(sense2_オポチュニティ抽出|Opportunity Extraction 2)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/sense2_research_summary.md','**/sense2_interview_analysis_*.md','**/delivery2_development_review.md']))}}"]
      instructions: |
        重要インサイト/阻害要因/クイックウィン候補を抽出し、抽出設定の初期値（参照ソース/重視観点/出力数）を提案してください。
      store_as: "auto_ops_seed"
    - name: "prefill_ops_seed"
      action: "display"
      content: |
        🔎 自動抽出候補
        {{auto_ops_seed}}
    - name: "collect_inputs"
      action: "ask_questions_with_template"
      template: |
        === 入力テンプレ ===
        1) 参照ソース（カンマ区切り。未指定なら自動探索）
        →
        2) 重視観点（例: 離脱要因/成功阻害/期待未達/競合差別化）
        →
        3) 出力数（例: 10）
        →
        ======================

    - name: "wait"
      action: "wait_for_all_answers"

    - name: "confirm"
      action: "confirm"
      message: "Sense2の検証結果からオポチュニティ仮説を抽出します。よろしいですか？"

    - name: "extract_ops"
      action: "analyze"
      data: ["{{read_files(inputs.1 | default: find_files(patterns=['**/sense2_interview_analysis_*.md','**/sense2_research_summary.md','**/strategy_product_metrics.md','**/progress_report.md','**/dev_tasks.yaml']))}}"]
      instructions: |
        参照ドキュメントから、以下の形式で最大{{inputs.3 | default:10}}件の機会仮説を抽出してください。
        - id: OP2-001
          title: 短い表題
          evidence: [引用/観測/計測]
          root_cause: 成功阻害となった根因
          expected_value: 期待価値
          quick_change: 直近の改善（1-2日で反映可能）
          experiment: 次サイクルの検証方法（指標）
      store_as: "ops"

    - name: "write_ops_yaml"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/sense2_opportunities.yaml"
      content: |
        opportunities:
          {{ops}}

    - name: "write_ops_md"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/sense2_opportunities.md"
      content: |
        # オポチュニティ仮説（Sense2）
        {{#each ops}}
        - **{{id}}**: {{title}}
          - 根拠: {{evidence}}
          - 根因: {{root_cause}}
          - 期待価値: {{expected_value}}
          - 即時改善: {{quick_change}}
          - 実験: {{experiment}}
        {{/each}}

    - name: "notify"
      action: "display"
      content: |
        ✅ Sense2の機会仮説を出力しました：
        - sense2_opportunities.yaml / sense2_opportunities.md
        次は Focus2/Delivery に反映してサイクル2を回しましょう。
```

## 次に実行
- `6_focus2/01_ロードマップ設計` → `6_focus2/02_WBS作成` → `6_focus2/03_バックログ初期化` → `4_delivery/07_開発タスク分解`
