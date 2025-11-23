# 01 Focus2 Roadmap Design

## 目的
Discovery/Focus/Deliveryの成果物（問題・解決・ストーリー・UI・タスク・OKR 等）を再利用し、アウトカムベースのテーマ→フィーチャ→タイムラインで構成する実行可能なロードマップを短時間で作成します。画像のように、入力（リサーチ/OKR/能力/競合/メトリクス）から、出力（テーマ・フィーチャ・タイムライン）を得ます。

## 実行手順（Rules Steps）
```yaml
- trigger: "(strategy_ロードマップ|Roadmap Design|ロードマップ設計)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/sense2_opportunities.yaml','**/strategy_product_metrics.md','**/dev_tasks.yaml','**/backlog/*.yaml']))}}"]
      instructions: |
        アウトカム・テーマ候補、主要フィーチャ候補、今四半期の候補バケットを簡潔に抽出し提示してください。
      store_as: "auto_roadmap_seed"
    - name: "prefill_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_roadmap_seed}}
    - name: "show_inputs"
      action: "display"
      content: |
        🔎 読み込み候補（存在すれば参照）
        - 3_discovery: problem_map.yaml / solution_map.yaml / story_map.yaml
        - 5_discovery(UI): screen_map.yaml / screen_flow.yaml
        - 2_focus: focus_product_definition.md / focus_market_size_estimation.md / focus_okr_*.md / focus_rough_roadmap.md
        - 4_delivery: 07_開発タスク分解/dev_tasks.yaml
        - 5_strategy: strategy_product_metrics.md（計測）

    - name: "collect_context"
      action: "ask_questions_with_template"
      template: |
        === 入力（インプット）===
        1) 期間（例: 2025-Q4〜2026-Q2）
        →
        2) 会社/組織としての優先事項やOKR（要点で）
        →
        3) チームの対応力/スキル・技術的制約（要点で）
        →
        4) 競合インサイト（あれば）/ 重要メトリクス（NSMやKPI）
        →
        ========================

    - name: "collect_outcome_themes"
      action: "ask_questions_with_template"
      template: |
        === アウトカム・テーマ（3件まで）===
        T1: テーマ名 / 期待アウトカム / 測定指標
        →
        T2: テーマ名 / 期待アウトカム / 測定指標
        →
        T3: テーマ名 / 期待アウトカム / 測定指標
        →
        ====================================

    - name: "collect_features"
      action: "ask_questions_with_template"
      template: |
        === フィーチャ候補（テーマに紐づけ、ST/依存/工数/自信度）===
        記法例: T1: 支払い方法の選択肢を増やす / feature=PayPay対応 / links=ST2 / effort=3 / confidence=0.75 / lane=Integration
        →
        追加で複数行OK（改行区切り）
        →
        =============================================

    - name: "collect_timeline_buckets"
      action: "ask_questions_with_template"
      template: |
        === タイムライン（バケット割当）===
        今四半期: （カンマ区切りで feature 名）
        →
        1四半期後: 
        →
        2四半期後: 
        →
        3四半期後以降: 
        →
        ===================================

    - name: "collect_dependencies"
      action: "ask_questions_with_template"
      template: |
        === 依存関係（任意。記法: featureA -> featureB）===
        →
        ============================

    - name: "wait"
      action: "wait_for_all_answers"

    - name: "confirm"
      action: "confirm"
      message: "ロードマップ（YAML/ボード/ガント/可視化）を生成します。よろしいですか？"

    - name: "write_roadmap_yaml"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/strategy_roadmap.yaml"
      content: |
        roadmap:
          period: {{context.1}}
          org_priorities: {{context.2}}
          team_capability: {{context.3}}
          competitive_and_metrics: {{context.4}}
          themes:
            - id: T1
              name: {{outcome_themes.T1_テーマ名 | default:outcome_themes.1}}
              outcome: {{outcome_themes.T1_期待アウトカム | default:""}}
              metric: {{outcome_themes.T1_測定指標 | default:""}}
            - id: T2
              name: {{outcome_themes.T2_テーマ名 | default:outcome_themes.2}}
              outcome: {{outcome_themes.T2_期待アウトカム | default:""}}
              metric: {{outcome_themes.T2_測定指標 | default:""}}
            - id: T3
              name: {{outcome_themes.T3_テーマ名 | default:outcome_themes.3}}
              outcome: {{outcome_themes.T3_期待アウトカム | default:""}}
              metric: {{outcome_themes.T3_測定指標 | default:""}}
          features:
            {{features_structured_yaml}}
          lanes: [Frontend, Backend, Integration, Marketing]
          buckets:
            now_q: {{timeline_buckets.今四半期 | to_list}}
            next_q: {{timeline_buckets.1四半期後 | to_list}}
            next_2q: {{timeline_buckets.2四半期後 | to_list}}
            later_3q_plus: {{timeline_buckets.3四半期後以降 | to_list}}
          dependencies:
            {{dependencies_structured_yaml}}
          numbering_policy:
            - prefix: T=Theme, F=Feature

    - name: "write_roadmap_board"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/strategy_roadmap_board.md"
      content: |
        # ロードマップ（ボード表示）
        
        ## 今四半期
        {{timeline_buckets.今四半期}}
        
        ## 1四半期後
        {{timeline_buckets.1四半期後}}
        
        ## 2四半期後
        {{timeline_buckets.2四半期後}}
        
        ## 3四半期後以降
        {{timeline_buckets.3四半期後以降}}
        
        ---
        補足: 単なる機能順序ではなく、テーマ（アウトカム）と指標に紐づけて定期的に優先順位を見直してください。

    - name: "write_roadmap_gantt"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/strategy_roadmap_gantt.md"
      content: |
        ```mermaid
        gantt
          dateFormat  YYYY-MM-DD
          title Roadmap (Rough)
          section 今四半期
          {{gantt_now_q}}
          section 1四半期後
          {{gantt_1q}}
          section 2四半期後
          {{gantt_2q}}
          section 3四半期後以降
          {{gantt_3q_plus}}
        ```

    - name: "write_roadmap_relations"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/strategy_roadmap_relations_mermaid.md"
      content: |
        ```mermaid
        flowchart LR
          %% Theme -> Feature（rank/自信度/工数）
          T1[Theme T1] --> F1[F1]
          T2[Theme T2] --> F2[F2]
          T3[Theme T3] --> F3[F3]
        ```

    - name: "notify"
      action: "notify"
      message: |
        ✅ ロードマップを作成しました：
        - {{patterns.flow_date}}/strategy_roadmap.yaml
        - {{patterns.flow_date}}/strategy_roadmap_board.md
        - {{patterns.flow_date}}/strategy_roadmap_gantt.md（Mermaid）
        - {{patterns.flow_date}}/strategy_roadmap_relations_mermaid.md
        次は `4_delivery/07_開発タスク分解` にテーマ/フィーチャを反映して実装計画へ進みましょう。
```

## 次に実行
- `4_delivery/07_開発タスク分解`（テーマ→フィーチャ→STへの展開）
- 必要に応じて `01_プロダクトメトリクス` で測定設計を更新
