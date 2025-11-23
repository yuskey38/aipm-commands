# : 開発計画作成

## 目的
Backlog2を対象に、実装対象・実装アプローチ・役割分担を明確化した開発計画を作成します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(delivery2_開発計画作成|development_plan2)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/backlog/backlog2.yaml','**/strategy_roadmap.yaml','**/focus2_wbs.md']))}}"]
      instructions: |
        ストーリー候補、技術的制約、役割分担の初期案を抽出して提示してください。
      store_as: "auto_plan_seed"
    - name: "prefill_plan_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_plan_seed}}
    - name: "collect_plan"
      action: "ask_questions"
      questions:
        - key: project_id
          question: "プロジェクトID"
          required: true
        - key: start_date
          question: "開発開始日"
          default: "{{today}}"
          required: true
        - key: story_ids
          question: "対象ストーリーIDs（US-1,US-2 カンマ区切り。空欄可）"
          required: false
        - key: technical_constraints
          question: "技術的な注意点・制約事項（任意）"
          required: false
        - key: roles
          question: "開発者の役割分担（任意）"
          required: false
        - key: implementation_approach
          question: "実装順序の基本方針"
          options: ["機能横断的","レイヤー横断的","ユーザーインターフェース優先","バックエンド優先","その他"]
          required: true
      store_as: plan

    - name: "confirm"
      action: "confirm"
      message: "開発計画ドキュメントを作成します。よろしいですか？"

    - name: "create_draft"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/delivery2_development_plan.md"
      content: |
        ---
        doc_type: development_plan
        project_id: {{plan.project_id}}
        created_at: {{today}}
        version: v1.0
        ---

        # 開発計画：{{plan.project_id}}

        ## 1. 概要
        - 開発開始日: {{plan.start_date}}
        - 実装アプローチ: {{plan.implementation_approach}}

        ## 2. 対象ストーリー
        {{#if plan.story_ids}}
        {{plan.story_ids}}
        {{else}}
        バックログから自動取得したストーリーを実装します。
        {{/if}}

        ## 3. 技術スタック構成
        - 要検討
        {{#if plan.technical_constraints}}
        ### 技術的制約事項
        {{plan.technical_constraints}}
        {{/if}}

        ## 4. 実装順序
        {{plan.implementation_approach}} に基づき、基盤→主要→拡張の順で実装します。

        {{#if plan.roles}}
        ## 5. 役割分担
        {{plan.roles}}
        {{/if}}

        ## 6. 次のステップ
        1. 実装順序計画の詳細化
        2. 依存関係の分析
        3. ストーリー実装の開始

    - name: "notify"
      action: "notify"
      message: |
        ✅ 生成しました：{{patterns.flow_date}}/delivery2_development_plan.md
        次は「06_実装順序計画」へ。
```

## 次に実行
- `06_実装順序計画`
