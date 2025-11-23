# : WBS作成（Delivery後の本格Planning）

## 目的
初回Deliveryで得た学びと現状アセット（story_map / dev_tasks / UI など）を前提に、次サイクルの作業分解構造（WBS）を作ります。PMBOK準拠の粒度で、後続のバックログ・スプリント計画に接続します。

## 実行手順（Rules Steps）
```
- trigger: "(focus2_WBS作成|postDelivery_WBS)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/strategy_roadmap.yaml','**/dev_tasks.yaml','**/progress_report.md']))}}"]
      instructions: |
        現在の成果物から、主要成果物候補・分解レベルの推奨・担当/期間のヒントを抽出して提示してください。
      store_as: "auto_wbs_seed"
    - name: "prefill_wbs_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_wbs_seed}}
    - name: "show_inputs"
      action: "display"
      content: |
        🔎 参照候補（存在すれば自分で確認して要点を入力してください）
        - Flow/.../04_ストーリーマップ/story_map.yaml
        - Flow/.../07_開発タスク分解/dev_tasks.yaml（status付き）
        - Flow/.../10_タスクリファイン/progress_report.md
        - Flow/.../05_UIワイヤーフレーム/screen_map.yaml

    - name: "collect_wbs_scope"
      action: "ask_questions_with_template"
      template: |
        === WBS入力 ===
        1) プロジェクト名
        →
        2) 主要成果物（例：MVP改善版、Pilot準備、GA準備など）
        →
        3) 分解レベル（例：3階層）
        →
        4) 担当者/期間の前提（任意）
        →
        =================

    - name: "confirm"
      action: "confirm"
      message: "WBSドラフトを生成します。よろしいですか？"

    - name: "create_wbs_draft"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/focus2_wbs.md"
      content: |
        # WBS（Delivery後の本格Planning）
        - プロジェクト: {{wbs_scope.1}}
        - 成果物: {{wbs_scope.2}}
        - 分解レベル: {{wbs_scope.3}}
        - 前提: {{wbs_scope.4}}

        ## 階層構造（雛形）
        1. {{wbs_scope.1}}
          1.1 企画/調整
            1.1.1 スコープ合意
            1.1.2 成果物定義
          1.2 実装
            1.2.1 Common更新
            1.2.2 ストーリー実装
          1.3 検証
            1.3.1 受け入れ/ログ検証
          1.4 リリース準備
            1.4.1 資料/手順

        （必要に応じて分解を追加してください）

    - name: "notify"
      action: "notify"
      message: |
        ✅ 出力しました：
        - {{patterns.flow_date}}/focus2_wbs.md
```

## 次に実行
- `03_バックログ初期化`
- `04_リスク分析`
