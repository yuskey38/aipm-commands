# 10 Delivery Task Refine

## 目的
- **Discovery仕様（3_discovery）を再読み込み**し、何を作ろうとしているかを再認識（コンテキスト忘却防止）
- **Delivery実装ファイル（4_delivery）を再読み込み**し、現在の実装状況を正確に把握
- **仕様と実装の整合性**を確認し、ズレがあれば修正方針を提示
- 現在の成果物（dev_tasks.yaml / order / runbook等）を読み直し、進捗を可視化
- 残タスクの優先順位を再構成（依存関係と学習効果を両立）
- 必要に応じてタスク内容を微修正し、確定版を再出力
- 進捗申告を dev_tasks.yaml に status として反映し、ヒューマンフレンドリーな進捗レポートを生成

## 実行手順
```yaml
- trigger: "(進捗レビュー|RefineNow|ReFocus)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/dev_tasks.yaml','**/dev_tasks_order.md','**/progress_report.md','**/09_チケット実行と検証/check_*.md']))}}"]
      instructions: |
        直近の進捗から、DONE/IN_PROGRESS/TODOの概要と、推奨の残タスク順（上位5件）を要点だけで出力してください。
      store_as: "auto_refine_hints"
    - name: "prefill_suggestions"
      action: "display"
      content: |
        🔎 進捗スナップショット（自動）
        {{auto_refine_hints}}
    
    # 1) Discovery仕様の再読み込み（コンテキスト再認識）
    - name: "reload_discovery_context"
      action: "read_multiple_files"
      files:
        - "Flow/{{today}}/{{flow_dir}}/01_ペルソナ作成/persona_todo.md"
        - "Flow/{{today}}/{{flow_dir}}/01_ペルソナ作成/experience_map.yaml"
        - "Flow/{{today}}/{{flow_dir}}/02_課題定義/problem_map.yaml"
        - "Flow/{{today}}/{{flow_dir}}/02_課題定義/customer_problem_map.yaml"
        - "Flow/{{today}}/{{flow_dir}}/03_ソリューションマップ/solution_map.yaml"
        - "Flow/{{today}}/{{flow_dir}}/04_ストーリーマップ/story_map.yaml"
        - "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/screen_map.yaml"
        - "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/screen_flow.yaml"
        - "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/ui_wireframe_todo.md"
      message: "Discovery仕様を再読み込みして、何を作ろうとしているかを再認識します"
    
    # 2) Delivery実装ファイルの再読み込み（現状把握）
    - name: "reload_delivery_artifacts"
      action: "read_multiple_files"
      files:
        - "Flow/{{today}}/{{flow_dir}}/08_チケット開始/T_COMMON_ENV/index.html"
        - "Flow/{{today}}/{{flow_dir}}/08_チケット開始/T_COMMON_ENV/styles.css"
        - "Flow/{{today}}/{{flow_dir}}/08_チケット開始/T_COMMON_ENV/app.js"
        - "Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/dev_tasks.yaml"
        - "Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/dev_runbook.md"
      message: "現在の実装状況を把握し、仕様との整合性を確認します"
    
    # 3) コンテキスト再認識サマリの生成
    - name: "generate_context_summary"
      action: "analyze"
      data: ["{{reload_discovery_context}}", "{{reload_delivery_artifacts}}"]
      instructions: |
        Discovery仕様とDelivery実装を比較分析し、以下を明確にしてください：
        1. 何を作ろうとしているか（ペルソナ・課題・ソリューション・ストーリーの要約）
        2. 現在どこまで実装できているか（完了機能・未実装機能）
        3. 仕様と実装の整合性（ズレがあれば指摘）
        4. 次に優先すべき実装内容（ストーリー・画面・機能の観点から）
        出力は表示用の構造化テキスト（見出し・箇条書き）で。
      store_as: "context_analysis"
    
    - name: "display_context_rerecognition"
      action: "display"
      content: |
        🧭 コンテキスト再認識サマリー
        {{context_analysis}}
    
    # 4) 詳細ファイル一覧表示
    - name: "show_detailed_file_list"
      action: "display"
      content: |
        📋 読み込み完了ファイル一覧
        
        **Discovery仕様（何を作るか）:**
        - ペルソナ: `01_ペルソナ作成/persona_todo.md` + `experience_map.yaml`
        - 課題: `02_課題定義/problem_map.yaml` + `customer_problem_map.yaml`
        - 解決策: `03_ソリューションマップ/solution_map.yaml`
        - ストーリー: `04_ストーリーマップ/story_map.yaml`
        - UI設計: `05_UIワイヤーフレーム/screen_map.yaml` + `screen_flow.yaml` + `ui_wireframe_todo.md`
        
        **Delivery実装（現状の成果物）:**
        - 基本構造: `08_チケット開始/T_COMMON_ENV/index.html`
        - スタイル: `08_チケット開始/T_COMMON_ENV/styles.css`
        - ロジック: `08_チケット開始/T_COMMON_ENV/app.js`
        - タスク管理: `07_開発タスク分解/dev_tasks.yaml`
        - 実装ガイド: `07_開発タスク分解/dev_runbook.md`
    
    # 5) 進捗の自己申告（柔らかく）
    - name: "collect_progress_mark"
      action: "ask_questions_with_template"
      template: |
        === 進捗チェック ===
        1) 完了タスクID（カンマ区切り。例: T_COMMON_ENV, T_ST2_MODE）
        →
        2) 実行中タスクID（任意）
        →
        3) ブロッカー（任意: 技術/時間/理解）
        →
        =====================================
    
    - name: "wait_progress"
      action: "wait_for_all_answers"
    
    # 6) 再認識サマリの提示（何をすべきか）
    - name: "display_refocus_summary"
      action: "display"
      content: |
        🧭 再認識ポイント
        - ストーリー（story_map）とUI（screen_map）が示す到達像を再確認
        - 共通タスク（Common）が先、次に依存順でStories、最後にNon-Functional
        - 受け入れ基準（AC）と期待ログ（console）の観点を常に併記
        - 仕様と実装の整合性を保ち、ペルソナの課題解決に焦点を当てる
    
    # 7) 自動提案（残タスク順序の再構成）
    - name: "propose_refined_order"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/10_タスクリファイン/dev_tasks_order_refined.md"
      content: |
        # 提案: 残タスクの実行順（進捗反映）
        {{refined_order}}
        
        ルール:
        - 未完了のCommonを最優先
        - 各Storyは依存を解決しつつ、価値が高い順で
        - NFRは主要UIが成立後に着手
    
    # 8) ステータス反映プラン（DONE/IN_PROGRESS/TODO）
    - name: "display_status_plan"
      action: "display"
      content: |
        🗂 ステータス反映ポリシー
        - 完了（DONE）: 入力で申告した完了タスクID
        - 実行中（IN_PROGRESS）: 入力で申告した実行中タスクID
        - 未着手（TODO）: 上記以外
        dev_tasks.yaml の各タスク要素に `status:` を追加/更新します。
    
    # 9) タスク内容のリファイン入力（任意修正）
    - name: "collect_task_refinements"
      action: "ask_questions_with_template"
      template: |
        === タスク内容のリファイン（任意）===
        1) 変更したいタスクID
        →
        2) 変更内容（title/depends/test_points/estimate_h/status など）
        →
        3) 新規に追加したいタスク（任意: id/title/depends/estimate_h/status）
        →
        =====================================
    
    - name: "confirm_generation"
      action: "confirm"
      message: "進捗反映＋リファイン内容で dev_tasks.yaml / dev_tasks_order.md / dev_runbook.md / progress_report.md を再生成します。よろしいですか？"
    
    # 10) 成果物の再出力（status付き）
    - name: "write_dev_tasks_yaml"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/dev_tasks.yaml"
      content: |
        # status欄（DONE/IN_PROGRESS/TODO）を各タスクへ反映済み
        {{dev_tasks_after_refine_and_status_yaml}}
    
    - name: "write_dev_tasks_order"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/dev_tasks_order.md"
      content: |
        {{final_order_markdown}}
    
    - name: "write_runbook"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/dev_runbook.md"
      content: |
        # LLM実装Runbook（更新版）
        1) Commonを最優先（未完了分）
        2) 各STは依存順に、価値が高いものから
        3) 受け入れ基準と期待ログ（console）を毎タスクで確認
        4) 完了後は `09_チケット実行と検証` でチェック記録
        5) 必要に応じこの `進捗レビュー` を再実行
    
    # 11) 進捗レポート（人間向け）
    - name: "write_progress_report"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/10_タスクリファイン/progress_report.md"
      content: |
        # 進捗レポート（{{today}}）
        
        ## サマリ
        - 完了（DONE）数: {{done_count}}
        - 実行中（IN_PROGRESS）数: {{wip_count}}
        - 未着手（TODO）数: {{todo_count}}
        - 合計タスク数: {{total_count}}
        - 進捗率: {{progress_rate_pct}}%
        - 推奨次タスク（上位3）: {{next_top3}}
        
        ## 詳細一覧
        ### DONE
        {{done_list}}
        
        ### IN_PROGRESS
        {{wip_list}}
        
        ### TODO（優先順）
        {{todo_ordered_list}}
        
        ## ブロッカー/メモ
        {{blockers_text}}
        
        ## 参照
        - ../07_開発タスク分解/dev_tasks.yaml（status反映済）
        - ../07_開発タスク分解/dev_tasks_order.md（最終順序）
        - ../07_開発タスク分解/dev_runbook.md（更新版）
        - ../04_ストーリーマップ/story_map.yaml / ../05_UIワイヤーフレーム/screen_map.yaml / screen_flow.yaml
    
    # 12) 次アクション提案（提出/発表 もしくは 次チケット）
    - name: "write_next_action"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/10_タスクリファイン/next_action.md"
      content: |
        # 次アクション提案
        
        - 進捗率: {{progress_rate_pct}}%（{{done_count}} / {{total_count}}）
        - 残タスク: {{todo_count}}（実行中: {{wip_count}}）
        
        ## 推奨
        {{#progress_rate_pct >= 80}}
        ほぼ完了と判断。以下を実行してください：
        1. 発表: 「発表_Marpスライド生成」または `@aipm_9_01_hackathon_marp_slide_generation`
        2. 提出: 「提出_成果物パッケージング」または `@aipm_9_02_hackathon_deliverable_packaging`
        
        参考: submitフォルダ名は `submit_{{today}}` を推奨
        {{/progress_rate_pct >= 80}}
        {{^progress_rate_pct >= 80}}
        まだ実装継続を推奨。次に着手すべきチケット：
        - 推奨: {{next_top1}}
        実行コマンド: `@aipm_4_08_delivery_ticket_start` を実行し、上記 `task_id` を指定してください。
        {{/progress_rate_pct >= 80}}
    
    - name: "display_next_action"
      action: "display"
      content: |
        📣 進捗率: {{progress_rate_pct}}%（DONE {{done_count}} / {{total_count}}）
        
        {{#progress_rate_pct >= 80}}
        ✅ ほぼ完了です。次は `@aipm_9_01_hackathon_marp_slide_generation` → `@aipm_9_02_hackathon_deliverable_packaging` の順で仕上げましょう。
        {{/progress_rate_pct >= 80}}
        {{^progress_rate_pct >= 80}}
        ▶ 次チケットに着手: {{next_top1}}（`@aipm_4_08_delivery_ticket_start`で `task_id` 指定）
        {{/progress_rate_pct >= 80}}
    
    - name: "notify"
      action: "display"
      content: |
        ✅ 再生成しました：
        - ../07_開発タスク分解/dev_tasks.yaml（status反映） / dev_tasks_order.md / dev_runbook.md
        - 10_タスクリファイン/progress_report.md（サマリ＋一覧）
        - 10_タスクリファイン/next_action.md（次アクション提案）
        - 提案順序: 10_タスクリファイン/dev_tasks_order_refined.md
        次は `08_チケット開始` に戻って継続実行してください。
```

## メモ
- 既存ファイルが無い場合は、ある範囲で空を許容しつつ入力ベースで再生成します。
- 進捗記録は `../07_開発タスク分解/dev_tasks.yaml` へ `status` を追記して保持します（DONE/IN_PROGRESS/TODO）。
