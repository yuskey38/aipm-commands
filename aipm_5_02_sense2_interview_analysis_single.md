# 02 Sense2 Interview Analysis Single

## 目的
ユーザーテスト1件の結果を、仮説検証の観点（成功/失敗/観測ログ/次の仮説）で短く構造化します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(sense2_インタビュー分析|sense2_個別分析)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/sense2_interview_guide.md','**/delivery2_story_*.md','**/acceptance_user_guide_*.md']))}}"]
      instructions: |
        直近のガイド/実装計画から、検証対象ストーリーや観測ログ観点の候補を抽出し、1-2行でまとめてください。
      store_as: "auto_interview_meta"
    - name: "prefill_meta"
      action: "display"
      content: |
        🔎 自動候補
        {{auto_interview_meta}}
    - name: "collect_meta"
      action: "ask_questions"
      questions:
        - key: "participant_id"
          question: "参加者ID/イニシャル"
          required: true
        - key: "date"
          question: "実施日（YYYY-MM-DD）"
          required: true
        - key: "story_or_task"
          question: "検証対象のストーリー/タスク（STや機能名）"
          required: true
        - key: "summary"
          question: "概要（3-6行）"
          required: true
      store_as: "m"

    - name: "confirm"
      action: "confirm"
      message: "Sense2個別分析レポートを作成します。よろしいですか？"

    - name: "create_report"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/sense2_interview_analysis_{{m.participant_id}}.md"
      content: |
        # ユーザーテスト分析（個別） - {{m.participant_id}}
        - 実施日: {{m.date}}
        - 対象: {{m.story_or_task}}

        ## 概要
        {{m.summary}}

        ## 観測ログ/指標（抜粋）
        - クリック/入力ログ: 
        - エラー/例外: 
        - 所要時間/成功率: 

        ## 成功/失敗判定（受け入れ基準/メトリクス準拠）
        - 判定: 
        - 根拠: 

        ## 気づき/痛点
        - 
        - 

        ## 次の仮説/改善案
        - 
        - 

    - name: "notify"
      action: "notify"
      message: |
        ✅ 個別分析を作成しました：
        {{patterns.flow_date}}/sense2_interview_analysis_{{m.participant_id}}.md
        次は「03_リサーチサマリー（全体）」へ。
```

## 次に実行
- `03_リサーチサマリー（全体）`
