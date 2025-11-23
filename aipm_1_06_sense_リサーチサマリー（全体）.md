# : リサーチサマリー（全体）

## 目的
複数の個別分析を統合し、ペルソナ/行動/課題/期待価値の観点で全体の洞察をまとめます。

## 実行手順（Rules Steps）
```yaml
- trigger: "(sense_リサーチサマリー|調査サマリー|全体分析|インタビューまとめ)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}", "{{find_files(patterns=['**/draft_interview_analysis_*.md','**/sense_*research*.md','**/draft_research_summary.md'])}}"]
      instructions: |
        スレッド内の直近の生成物パスを収集し、sources候補としてカンマ区切りで返してください。
        出力は `{"sources": "path1,path2,..."}` のJSON1件のみ。
      store_as: "prefill"
    - name: "collect_sources"
      action: "ask_question"
      question: "個別分析ファイルのパス（カンマ区切り。例：draft_interview_analysis_*.md）"
      required: true
      default: "{{prefill.sources}}"
      store_as: "sources"

    - name: "wait"
      action: "wait_for_all_answers"

    - name: "confirm"
      action: "confirm"
      message: "指定の個別分析を統合してリサーチサマリーを作成します。よろしいですか？"

    - name: "ensure_output_dir"
      action: "execute_shell"
      command: "mkdir -p {{patterns.flow_date}}/1_sense/06_リサーチサマリー（全体）"

    - name: "create_summary"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/1_sense/06_リサーチサマリー（全体）/draft_research_summary.md"
      content: |
        # リサーチサマリー（全体）
        参照: {{sources}}

        ## 主要インサイト
        - 
        - 
        - 

        ## ペルソナ仮説（更新）
        - 

        ## 行動パターン（カスタマージャーニー示唆）
        - 

        ## 課題クラスタ
        - 

        ## 期待する価値/体験
        - 

        ## 機会（Opportunity）候補
        - 

    - name: "notify"
      action: "notify"
      message: |
        ✅ リサーチサマリーを作成しました：
        {{patterns.flow_date}}/draft_research_summary.md
        次は「03_オポチュニティ仮説抽出」で機会仮説を整理しましょう。
```

## 次に実行
- `03_オポチュニティ仮説抽出`
