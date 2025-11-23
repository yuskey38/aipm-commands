# : リスク分析（Delivery後の本格Planning）

## 目的
次サイクルに向け、Delivery結果とバックログ/ロードマップを踏まえたリスクを整理します。影響度×確率、対応策、責任者までを簡潔に記録します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(focus2_リスク分析|postDelivery_risk)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/strategy_roadmap.yaml','**/backlog/*.yaml','**/delivery2_development_plan.md','**/progress_report.md']))}}"]
      instructions: |
        現状の計画と進捗から、上位リスク（3-5件）の初期候補を抽出し、分類/影響度/確率/暫定対応策を短く提示してください。
      store_as: "auto_risk_seed"
    - name: "prefill_risk_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_risk_seed}}
    - name: "show_inputs"
      action: "display"
      content: |
        🔎 参照候補
        - {{patterns.flow_date}}/strategy_roadmap.yaml
        - {{patterns.flow_date}}/backlog/backlog.yaml
        - {{patterns.flow_date}}/focus2_wbs.md
        - Flow/.../07_開発タスク分解/dev_tasks.yaml（status）

    - name: "collect_risks"
      action: "ask_questions_with_template"
      template: |
        === リスク入力（複数OK。行ごと）===
        記法: 分類=技術/スケジュール/リソース/外部/組織 | 事象 | 影響度(H/M/L) | 確率(H/M/L) | 対応策 | 責任者
        →
        =====================================

    - name: "confirm"
      action: "confirm"
      message: "リスク計画ドラフトを生成します。よろしいですか？"

    - name: "write_risk_plan"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/focus2_risk_plan.md"
      content: |
        # リスク計画（Delivery後）

        | 分類 | 事象 | 影響度 | 確率 | 対応策 | 責任者 |
        | --- | --- | --- | --- | --- | --- |
        {{risks_table}}

        ## 評価マトリクス
        影響度×確率で優先度を決め、H×Hは直ちに対策、H×M/M×Hは早期対策、L×*は監視。

    - name: "notify"
      action: "notify"
      message: |
        ✅ 出力しました：
        - {{patterns.flow_date}}/focus2_risk_plan.md
```

## 次に実行
- `4_delivery/07_開発タスク分解` の順序/非機能に反映
- 定期的に `10_タスクリファイン` で見直し
