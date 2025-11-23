# : ラフロードマップ作成（12週/4Q粒度）

## 目的
Sense→Focusで決めた方向性を、四半期/週の粗い粒度でロードマップ化します。MVP→Pilot→GAの大枠を描き、OKRと整合させます。

## 実行手順（Rules Steps）
```yaml
- trigger: "(focus_ラフロードマップ|Rough Roadmap)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}", "{{read_files(find_files(patterns=['Flow/**/*.md','Flow/**/*.mdx','**/submit_*/README.md']))}}"]
      instructions: |
        スレッドの文脈から、以下の推定値を返してください（なければ空）。
        keys: period, phases, milestones, dependencies
        出力はJSONのオブジェクト1件のみ。
      store_as: "prefill"
    - name: "collect_inputs"
      action: "ask_questions_with_template"
      template: |
        === ロードマップ入力 ===
        1) 期間（例: 2025-Q4〜2026-Q2）
        → {{prefill.period}}
        2) フェーズ構成（例: MVP→Pilot→GA）
        → {{prefill.phases}}
        3) 主要マイルストーン（箇条書き）
        → {{prefill.milestones}}
        4) 依存/前提（任意）
        → {{prefill.dependencies}}
        =========================

    - name: "wait"
      action: "wait_for_all_answers"

    - name: "confirm"
      action: "confirm"
      message: "入力に基づきラフロードマップを作成します。よろしいですか？"

    - name: "create_draft"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/focus_rough_roadmap.md"
      content: |
        # ラフロードマップ（粗粒度）
        - 期間: {{period}}
        - フェーズ: {{phases}}

        ## マイルストーン
        {{milestones}}

        ## 依存/前提
        {{dependencies}}

        ```mermaid
        gantt
          dateFormat  YYYY-MM-DD
          title Rough Roadmap
          section MVP
          MVP準備           :done,    des1, 2025-10-01, 2025-10-15
          MVP開発           :active,  des2, 2025-10-16, 2025-11-15
          MVP検証           :         des3, 2025-11-16, 2025-12-01
          section Pilot
          Pilot準備         :         des4, 2025-12-02, 2026-01-15
          Pilot実施         :         des5, 2026-01-16, 2026-02-28
          section GA
          GA準備            :         des6, 2026-03-01, 2026-03-31
        ```
```

## 次に実行
- `04_OKR作成`


