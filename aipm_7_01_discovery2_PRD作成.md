# : PRD作成（二段階目）

## 目的
初回DeliveryとSense2（ユーザーテスト）の学びを反映し、二段階目の詳細なプロダクト要求仕様書（PRD）を作成します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(discovery2_PRD|PRD二段階目)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/sense2_research_summary.md','**/sense2_opportunities.yaml','**/strategy_roadmap.yaml','**/delivery2_development_review.md']))}}"]
      instructions: |
        PRDの基本情報/概要/ソリューション/機能要件の候補を各1-2行で抽出して提示してください。
      store_as: "auto_prd_seed"
    - name: "prefill_prd_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_prd_seed}}
    - name: "show_references"
      action: "display"
      content: |
        🔎 参照候補（任意）
        - 5_sense2: sense2_research_summary.md / sense2_opportunities.yaml
        - 6_focus2: strategy_roadmap.yaml / focus2_wbs.md / backlog/backlog.yaml
        - 3_discovery: story_map.yaml / solution_map.yaml / problem_map.yaml

    - name: "collect_prd"
      action: "ask_questions"
      questions:
        # 基本情報
        - key: "product_name"
          question: "製品名"
          required: true
        - key: "version"
          question: "バージョン（例: 1.0）"
          default: "1.0"
          required: true
        - key: "author"
          question: "作成者"
          required: true
        - key: "creation_date"
          question: "作成日（YYYY-MM-DD）"
          default: "{{today}}"
          required: true
        # 製品概要
        - key: "product_vision"
          question: "製品ビジョン（解決する問題と将来の方向性）"
          required: true
        - key: "product_purpose"
          question: "製品の目的（この製品が達成しようとしていること）"
          required: true
        - key: "target_users"
          question: "主要ユーザー"
          required: true
        - key: "user_needs"
          question: "主な課題・ニーズ（背景 → 課題/ニーズ）"
          required: true
        # ソリューション定義
        - key: "proposed_solution"
          question: "提案するソリューション"
          required: true
        - key: "expected_behavior_change"
          question: "期待する行動変化（ソリューション → 行動変化）"
          required: true
        - key: "competitive_advantage"
          question: "競合差別化ポイント（任意）"
          required: false
        # 機能要件
        - key: "core_features"
          question: "コア機能（箇条書き）"
          required: true
        - key: "user_stories"
          question: "主要ユーザーストーリー（箇条書き）"
          required: true
        - key: "feature_priorities"
          question: "機能の優先順位（Must/Should/Could/Won't 等）"
          required: true
        # 非機能要件
        - key: "performance_requirements"
          question: "パフォーマンス要件（応答時間/スループット等）"
          required: true
        - key: "security_requirements"
          question: "セキュリティ要件（認証/データ保護等）"
          required: true
        - key: "scalability"
          question: "スケーラビリティ（任意）"
          required: false
        - key: "compatibility"
          question: "互換性・統合性（任意）"
          required: false
        - key: "usability_requirements"
          question: "ユーザビリティ要件（任意）"
          required: false
        # 技術仕様
        - key: "technology_stack"
          question: "技術スタック（任意）"
          required: false
        - key: "architecture_overview"
          question: "アーキテクチャ概要（任意）"
          required: false
        - key: "data_requirements"
          question: "データ要件（任意）"
          required: false
        # リリース計画
        - key: "mvp_definition"
          question: "MVPの定義"
          required: true
        - key: "release_roadmap"
          question: "リリースロードマップ（任意）"
          required: false
        - key: "feedback_plan"
          question: "フィードバックプラン（任意）"
          required: false
        # その他
        - key: "constraints"
          question: "制約条件（時間/予算/技術など）"
          required: true
        - key: "assumptions"
          question: "前提条件"
          required: true
        - key: "risks_and_mitigations"
          question: "リスクと軽減策（任意）"
          required: false
      store_as: "prd"

    - name: "wait"
      action: "wait_for_all_answers"

    - name: "confirm"
      action: "confirm"
      message: "PRDドラフトを生成します。よろしいですか？"

    - name: "create_prd"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/discovery2_prd.md"
      content: |
        # {{prd.product_name}} - プロダクト要求仕様書 (PRD)
        
        **バージョン:** {{prd.version}}  
        **作成日:** {{prd.creation_date}}  
        **作成者:** {{prd.author}}  
        **最終更新日:** {{prd.creation_date}}
        
        ## 1. 製品概要
        
        ### 1.1 製品ビジョン
        {{prd.product_vision}}
        
        ### 1.2 製品の目的
        {{prd.product_purpose}}
        
        ### 1.3 対象ユーザー
        {{prd.target_users}}
        
        ### 1.4 課題・ニーズ定義
        {{prd.user_needs}}
        
        ## 2. ソリューション
        
        ### 2.1 提案するソリューション
        {{prd.proposed_solution}}
        
        ### 2.2 期待する行動変化
        {{prd.expected_behavior_change}}
        
        ### 2.3 競合差別化ポイント
        {{prd.competitive_advantage}}
        
        ## 3. 機能要件
        
        ### 3.1 コア機能
        {{prd.core_features}}
        
        ### 3.2 ユーザーストーリー
        {{prd.user_stories}}
        
        ### 3.3 機能の優先順位
        {{prd.feature_priorities}}
        
        ## 4. 非機能要件
        
        ### 4.1 パフォーマンス要件
        {{prd.performance_requirements}}
        
        ### 4.2 セキュリティ要件
        {{prd.security_requirements}}
        
        ### 4.3 スケーラビリティ
        {{prd.scalability}}
        
        ### 4.4 互換性・統合性
        {{prd.compatibility}}
        
        ### 4.5 ユーザビリティ要件
        {{prd.usability_requirements}}
        
        ## 5. 技術仕様
        
        ### 5.1 技術スタック
        {{prd.technology_stack}}
        
        ### 5.2 アーキテクチャ概要
        {{prd.architecture_overview}}
        
        ### 5.3 データ要件
        {{prd.data_requirements}}
        
        ## 6. リリース計画
        
        ### 6.1 MVPの定義
        {{prd.mvp_definition}}
        
        ### 6.2 リリースロードマップ
        {{prd.release_roadmap}}
        
        ### 6.3 フィードバックプラン
        {{prd.feedback_plan}}
        
        ## 7. 制約・前提条件
        
        ### 7.1 制約条件
        {{prd.constraints}}
        
        ### 7.2 前提条件
        {{prd.assumptions}}
        
        ### 7.3 リスクと軽減策
        {{prd.risks_and_mitigations}}
        
        ## 8. 承認
        | 役割 | 氏名 | 署名 | 日付 |
        |------|------|------|------|
        | プロダクトオーナー | | | |
        | プロジェクトマネージャー | | | |
        | 開発リード | | | |
        
        ## 9. 変更履歴
        | バージョン | 日付 | 変更者 | 変更内容 |
        |-----------|------|-------|---------|
        | {{prd.version}} | {{prd.creation_date}} | {{prd.author}} | 初版作成 |

    - name: "notify"
      action: "notify"
      message: |
        ✅ PRDドラフトを作成しました：
        - {{patterns.flow_date}}/discovery2_prd.md
        次は DesignDoc を作成し、Backlog2 初期化へ。
```

## 次に実行
- `02_DesignDoc作成`
