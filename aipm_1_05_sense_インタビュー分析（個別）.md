# : インタビュー分析（個別）

## 目的
1件のインタビュー結果を短い構造で整理し、示唆を抽出します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(sense_インタビュー分析|インタビュー結果|インタビュー記録|個別分析)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}", "{{read_files(find_files(patterns=['Flow/**/*.md','Flow/**/*.mdx','**/submit_*/README.md']))}}"]
      instructions: |
        スレッドの直近文脈と成果物から、以下キーの推定値を出力（なければ空）。
        keys: participant_id, date, summary
        出力はJSONのオブジェクト1件のみ。
      store_as: "prefill"
    - name: "collect_meta"
      action: "ask_questions"
      questions:
        - key: "participant_id"
          question: "参加者ID/イニシャル"
          required: true
          default: "{{prefill.participant_id}}"
        - key: "date"
          question: "実施日（YYYY-MM-DD）"
          required: true
          default: "{{prefill.date}}"
        - key: "summary"
          question: "概要（3-5行）"
          required: true
          default: "{{prefill.summary}}"
      store_as: "meta"

    - name: "confirm"
      action: "confirm"
      message: "インタビュー分析（個別）レポートを作成します。よろしいですか？"

    - name: "ensure_output_dir"
      action: "execute_shell"
      command: "mkdir -p {{patterns.flow_date}}/1_sense/05_インタビュー分析（個別）"

    - name: "create_report"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/1_sense/05_インタビュー分析（個別）/draft_interview_analysis_{{meta.participant_id}}.md"
      content: |
        # インタビュー分析（個別） - {{meta.participant_id}}
        - 実施日: {{meta.date}}

        ## 概要
        {{meta.summary}}

        ## 行動/発言ハイライト
        - 
        - 
        - 

        ## 課題/痛点
        - 
        - 

        ## 期待する体験/価値
        - 
        - 

        ## 示唆（Implications）
        - 
        - 

    - name: "notify"
      action: "notify"
      message: |
        ✅ 個別分析を作成しました：
        {{patterns.flow_date}}/draft_interview_analysis_{{meta.participant_id}}.md
        次は「07_リサーチサマリー（全体）」へ。
```

## 次に実行
- `07_リサーチサマリー（全体）`
