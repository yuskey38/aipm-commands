# : DesignDoc作成（二段階目）

## 目的
二段階目PRDとSense2/Focus2の結果を踏まえ、詳細な設計文書（Design Doc）を作成します。実装・テスト計画まで含め、次サイクルDeliveryに渡せる状態にします。

## 実行手順（Rules Steps）
```yaml
- trigger: "(discovery2_DesignDoc|DesignDoc二段階目|設計文書二段階目)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/discovery2_prd.md','**/sense2_research_summary.md','**/strategy_roadmap.yaml']))}}"]
      instructions: |
        設計文書の問題/背景/目標/概要/技術アプローチの候補を各1-2行で抽出し提示してください。
      store_as: "auto_dd_seed"
    - name: "prefill_dd_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_dd_seed}}
    - name: "show_refs"
      action: "display"
      content: |
        🔎 参照候補
        - discovery2_prd.md
        - sense2_research_summary.md / sense2_opportunities.yaml
        - strategy_roadmap.yaml / focus2_wbs.md / backlog/backlog.yaml

    - name: "collect_design"
      action: "ask_questions"
      questions:
        - key: project_name
          question: "プロジェクト名"
          required: true
        - key: author
          question: "作成者"
          required: true
        - key: creation_date
          question: "作成日（YYYY-MM-DD）"
          default: "{{today}}"
          required: true
        - key: status
          question: "ステータス（ドラフト等）"
          default: "ドラフト"
          required: true
        - key: version
          question: "文書バージョン"
          default: "0.1"
          required: true
        - key: problem_statement
          question: "このシステム/機能が解決する問題は何ですか？"
          required: true
        - key: background
          question: "この設計の背景情報"
          required: true
        - key: goals
          question: "この設計の目標（箇条書き）"
          required: true
        - key: non_goals
          question: "この設計の非目標（箇条書き・任意）"
          required: false
        - key: overview
          question: "ハイレベルな設計概要（3〜4段落）"
          required: true
        - key: system_context
          question: "システムコンテキスト図（任意・説明可）"
          required: false
        - key: technical_approach
          question: "技術的アプローチ（採用技術/コンポーネント/役割）"
          required: true
        - key: data_model
          question: "データモデル/ストレージ（任意）"
          required: false
        - key: api_interface
          question: "API/インターフェース（任意）"
          required: false
        - key: scalability
          question: "スケーラビリティとパフォーマンス（任意）"
          required: false
        - key: alternatives
          question: "検討した代替案"
          required: true
        - key: decision_rationale
          question: "現在の設計を選んだ理由"
          required: true
        - key: security_considerations
          question: "セキュリティの考慮点（任意）"
          required: false
        - key: privacy_considerations
          question: "プライバシーの考慮点（任意）"
          required: false
        - key: other_concerns
          question: "その他の懸念事項（任意）"
          required: false
        - key: implementation_plan
          question: "実装計画（任意）"
          required: false
        - key: testing_approach
          question: "テスト方針（任意）"
          required: false
        - key: monitoring
          question: "モニタリングとメトリクス（任意）"
          required: false
      store_as: dd

    - name: "wait"
      action: "wait_for_all_answers"

    - name: "confirm"
      action: "confirm"
      message: "Design Docドラフトを生成します。よろしいですか？"

    - name: "create_designdoc"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/discovery2_designdoc.md"
      content: |
        # {{dd.project_name}} - Design Doc（設計文書）
        
        **作成者:** {{dd.author}}  
        **作成日:** {{dd.creation_date}}  
        **ステータス:** {{dd.status}}  
        **バージョン:** {{dd.version}}
        
        ## 目次
        - 1. コンテキストとスコープ
        - 2. 目標と非目標
        - 3. 設計概要
        - 4. 詳細設計
        - 5. 代替案とトレードオフ
        - 6. 懸念事項と考慮点
        - 7. 実装とテスト計画
        
        ## 1. コンテキストとスコープ
        ### 問題の説明
        {{dd.problem_statement}}
        
        ### 背景
        {{dd.background}}
        
        ## 2. 目標と非目標
        ### 目標
        {{dd.goals}}
        
        ### 非目標
        {{dd.non_goals}}
        
        ## 3. 設計概要
        {{dd.overview}}
        
        {{#if dd.system_context}}
        ### システムコンテキスト図
        {{dd.system_context}}
        {{/if}}
        
        ## 4. 詳細設計
        ### 技術的アプローチ
        {{dd.technical_approach}}
        
        {{#if dd.data_model}}
        ### データモデル/ストレージ
        {{dd.data_model}}
        {{/if}}
        
        {{#if dd.api_interface}}
        ### APIとインターフェース
        {{dd.api_interface}}
        {{/if}}
        
        {{#if dd.scalability}}
        ### スケーラビリティとパフォーマンス
        {{dd.scalability}}
        {{/if}}
        
        ## 5. 代替案と検討したトレードオフ
        ### 検討した代替案
        {{dd.alternatives}}
        
        ### 選定の理由
        {{dd.decision_rationale}}
        
        ## 6. 懸念事項と考慮点
        {{#if dd.security_considerations}}
        ### セキュリティの考慮点
        {{dd.security_considerations}}
        {{/if}}
        
        {{#if dd.privacy_considerations}}
        ### プライバシーの考慮点
        {{dd.privacy_considerations}}
        {{/if}}
        
        {{#if dd.other_concerns}}
        ### その他の懸念事項
        {{dd.other_concerns}}
        {{/if}}
        
        ## 7. 実装とテスト計画
        {{#if dd.implementation_plan}}
        ### 実装計画
        {{dd.implementation_plan}}
        {{/if}}
        
        {{#if dd.testing_approach}}
        ### テスト方針
        {{dd.testing_approach}}
        {{/if}}
        
        {{#if dd.monitoring}}
        ### モニタリングとメトリクス
        {{dd.monitoring}}
        {{/if}}

    - name: "notify"
      action: "notify"
      message: |
        ✅ Design Docドラフトを作成しました：
        - {{patterns.flow_date}}/discovery2_designdoc.md
        次は Backlog2 初期化へ。
```

## 次に実行
- `03_バックログ2初期化`
