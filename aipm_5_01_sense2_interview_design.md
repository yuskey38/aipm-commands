# 01 Sense2 Interview Design

## 目的
初回Deliveryで実装したアプリ/機能に対し、仮説（Discovery/Focus）と照らしてユーザーテストを設計します。評価観点（達成したいアウトカム、期待ログ、受け入れ基準）を含めます。

## 実行手順（Rules Steps）
```yaml
- trigger: "(sense2_インタビュー設計|ユーザーテスト設計)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/acceptance_user_guide_*.md','**/sense2_research_summary.md','**/strategy_product_metrics.md','**/delivery2_development_plan.md']))}}"]
      instructions: |
        直近成果物から、テスト目的/対象プロフィール/主要タスク/成功基準の候補を各1-2行で抽出してください。
      store_as: "auto_test_design"
    - name: "prefill_from_context"
      action: "display"
      content: |
        🔎 自動候補
        {{auto_test_design}}
    - name: "show_references"
      action: "display"
      content: |
        🔎 参照候補
        - 3_discovery: problem_map.yaml / solution_map.yaml / story_map.yaml
        - 4_delivery: 08_チケット開始/acceptance_user_guide_*.md / 09_チケット実行と検証/check_*.md
        - 5_strategy: strategy_product_metrics.md

    - name: "collect_test_design"
      action: "ask_questions_with_template"
      template: |
        === ユーザーテスト設計 ===
        1) テスト目的（検証したい仮説/アウトカム）
        →
        2) 対象プロフィール（現ユーザー/想定ユーザー）
        →
        3) 主要タスク（アプリ上での操作3-5）
        →
        4) 成功基準（受け入れ基準/期待ログ/メトリクス）
        →
        5) 制約（時間/環境/記録方法/倫理）
        →
        ==========================
      store_as: "td"

    - name: "confirm"
      action: "confirm"
      message: "Sense2用のインタビュー（ユーザーテスト）ガイドを生成します。よろしいですか？"

    - name: "write_interview_guide"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/sense2_interview_guide.md"
      content: |
        # ユーザーテストガイド（Sense2）
        - 目的: {{td.1}}
        - 対象: {{td.2}}
        - 制約: {{td.5}}

        ## タスクシナリオ
        {{td.3}}

        ## 成功基準（合否/観測）
        - 受け入れ基準: {{td.4}}
        - 期待ログ/メトリクス: strategy_product_metrics.md 参照

        ## 進行
        1) ウォームアップ → 2) タスク実施（発話思考） → 3) ふりかえり

    - name: "notify"
      action: "notify"
      message: |
        ✅ ユーザーテスト設計を作成しました：
        - {{patterns.flow_date}}/sense2_interview_guide.md
```

## 次に実行
- `02_インタビュー分析（個別）`
