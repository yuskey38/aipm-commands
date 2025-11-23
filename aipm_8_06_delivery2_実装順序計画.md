# : 実装順序計画

## 目的
Backlog2/依存/価値・難易度を踏まえ、二段階目の最適な実装順序を決定します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(delivery2_実装順序計画|impl_order2)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/backlog/backlog2.yaml','**/delivery2_development_plan.md']))}}"]
      instructions: |
        依存/難易度/価値の初期候補を抽出し、質問の入力補助として箇条書き提示してください。
      store_as: "auto_order_seed"
    - name: "prefill_order_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_order_seed}}
    - name: "collect_inputs"
      action: "ask_questions"
      questions:
        - key: project_id
          question: "プロジェクトID"
          required: true
        - key: story_ids
          question: "対象ストーリーIDs（US-1,US-2）"
          required: true
        - key: dependencies
          question: "各ストーリーの依存関係（例: US-1:US-2,US-3;US-2:US-3）"
          required: false
        - key: complexity
          question: "各ストーリーの難易度（例: US-1:高,US-2:中）"
          required: false
        - key: business_value
          question: "各ストーリーのビジネス価値（例: US-1:高,US-2:中）"
          required: false
        - key: prioritization_approach
          question: "優先度付けの方針"
          options: ["ビジネス価値優先","技術的依存関係優先","リスク低減優先","最小実装(MVP)優先","バランス型"]
          default: "バランス型"
          required: true
      store_as: ord

    - name: "confirm"
      action: "confirm"
      message: "実装順序計画ドキュメントを作成します。よろしいですか？"

    - name: "create_draft"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/delivery2_implementation_order.md"
      content: |
        ---
        doc_type: implementation_order
        project_id: {{ord.project_id}}
        created_at: {{today}}
        version: v1.0
        ---

        # 実装順序計画：{{ord.project_id}}

        ## 1. 概要
        優先度付け方針: **{{ord.prioritization_approach}}**

        ## 2. ストーリー分析（入力）
        - IDs: {{ord.story_ids}}
        - 依存: {{ord.dependencies}}
        - 難易度: {{ord.complexity}}
        - 価値: {{ord.business_value}}

        ## 3. 推奨実装順序（雛形）
        1. 基盤ストーリー（他に依存されているもの）
        2. 高価値/低複雑性ストーリー
        3. 中価値ストーリー
        4. その他

        ## 4. 実装スケジュール案（雛形）
        | フェーズ | ストーリー | 予定期間 |
        |--------|----------|---------|
        | フェーズ1 |  |  |
        | フェーズ2 |  |  |
        | フェーズ3 |  |  |

        ## 5. 次のステップ
        1. ストーリー実装へ
        2. 実装中に発見された依存を追記

    - name: "notify"
      action: "notify"
      message: |
        ✅ 生成しました：{{patterns.flow_date}}/delivery2_implementation_order.md
        次は「07_ストーリー実装」へ。
```

## 次に実行
- `07_ストーリー実装`
