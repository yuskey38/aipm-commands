# : バックログ2初期化（PRD/DesignDoc反映）

## 目的
Discovery2のPRDとDesignDoc、Sense2の学びを取り込み、二段階目のバックログ（Backlog2）を初期化します。エピック/ストーリー/優先順位/依存を更新します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(discovery2_バックログ2初期化|backlog2_init)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/discovery2_prd.md','**/discovery2_designdoc.md','**/sense2_research_summary.md','**/strategy_roadmap.yaml','**/focus2_wbs.md','**/backlog/backlog.yaml']))}}"]
      instructions: |
        エピック/ストーリーの追加・更新候補と優先基準の草案を抽出し、display向けの箇条書きで提示してください。
      store_as: "auto_backlog2_seed"
    - name: "prefill_backlog2_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_backlog2_seed}}
    - name: "show_refs"
      action: "display"
      content: |
        🔎 参照候補
        - discovery2_prd.md / discovery2_designdoc.md
        - sense2_research_summary.md / sense2_opportunities.yaml
        - strategy_roadmap.yaml / focus2_wbs.md
        - 既存: backlog/backlog.yaml（差分更新の参考）

    - name: "collect_inputs"
      action: "ask_questions_with_template"
      template: |
        === Backlog2 入力 ===
        1) プロダクト名
        →
        2) 追加/更新するエピック（行単位。PRD/DesignDoc根拠もあれば併記）
        →
        3) 追加/更新するユーザーストーリー（行単位。As/I want/So that）
        →
        4) 優先順位基準（例：価値×学習×依存クリティカル）
        →
        =====================

    - name: "confirm"
      action: "confirm"
      message: "Backlog2（backlog2.yaml/epics2.yaml）を生成します。よろしいですか？"

    - name: "write_backlog2_yaml"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/backlog/backlog2.yaml"
      content: |
        product: {{inputs.1}}
        priority_policy: {{inputs.4}}
        epics:
          {{epics2_from_inputs_and_prd}}
        stories:
          {{stories2_from_inputs_and_prd}}

    - name: "write_epics2_yaml"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/backlog/epics2.yaml"
      content: |
        epics:
          {{epics2_from_inputs_and_prd}}

    - name: "notify"
      action: "notify"
      message: |
        ✅ 出力しました：
        - {{patterns.flow_date}}/backlog/backlog2.yaml
        - {{patterns.flow_date}}/backlog/epics2.yaml
        次は `4_delivery/07_開発タスク分解` でBacklog2を反映してください。
```

## 次に実行
- `4_delivery/07_開発タスク分解`（Backlog2に基づく実装計画）
