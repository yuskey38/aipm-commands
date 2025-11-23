# : リサーチサマリー（全体）

## 目的
ユーザーテストの個別分析を統合し、Discovery/Focusの仮説がどの程度達成されたかを評価します。成功/未達/学び/改善方針を明確にします。

## 実行手順（Rules Steps）
```yaml
- trigger: "(sense2_リサーチサマリー|sense2_全体分析)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/sense2_interview_analysis_*.md','**/strategy_product_metrics.md']))}}"]
      instructions: |
        個別分析から達成/部分達成/未達の候補と主要インサイト3件を抽出し、display用に要点列挙してください。
      store_as: "auto_summary_seed"
    - name: "prefill_summary"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_summary_seed}}
    - name: "collect_sources"
      action: "ask_question"
      question: "個別分析ファイルのパス（カンマ区切り。例：sense2_interview_analysis_*.md）"
      required: true
      store_as: "sources"

    - name: "confirm"
      action: "confirm"
      message: "指定の個別分析を統合してSense2リサーチサマリーを作成します。よろしいですか？"

    - name: "create_summary"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/sense2_research_summary.md"
      content: |
        # ユーザーテスト サマリー（Sense2）
        参照: {{sources}}

        ## 達成状況（仮説検証）
        - 達成: 
        - 部分達成: 
        - 未達: 

        ## 主要インサイト
        - 
        - 

        ## 障害/摩擦点
        - 

        ## 推奨改善（短期/中期）
        - 短期: 
        - 中期: 

        ## 計測の示唆（NSM/KPI）
        - 

    - name: "notify"
      action: "notify"
      message: |
        ✅ Sense2リサーチサマリーを作成しました：
        - {{patterns.flow_date}}/sense2_research_summary.md
        次は「04_オポチュニティ仮説抽出」で次サイクルの機会を抽出します。
```

## 次に実行
- `04_オポチュニティ仮説抽出`
