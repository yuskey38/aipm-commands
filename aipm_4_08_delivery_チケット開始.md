# 08 : チケット開始（AIPMハッカソン）

## 目的
`07_開発タスク分解` で生成された `dev_tasks.yaml` / `dev_tasks_order.md` / `dev_runbook.md` / **`total_development_spec.md`** を基に、実行するタスク（task_id）を選び、作業メモ/受け入れ確認ガイド/実装タスクリストを生成します。**トータル開発仕様書を必ず読み込み**、何を作ろうとしているかのコンテキストを維持しながら実装を進めます。さらに、関連するスクリーン（SC）のデザイン仕様とDraw.io図面を参照し、設計バンドルを自動生成します。

## 実行手順
```yaml
- trigger: "(実装_チケット開始|TicketStart)"
  priority: high
  steps:
    # 前回実装完了の自動検出とステータス更新
    - name: "detect_previous_completion"
      action: "analyze"
      data: ["{{thread_messages}}", "{{read_files(find_files(patterns=['**/09_チケット実行と検証/check_*.md']))}}"]
      instructions: |
        直近のスレッドメッセージと検証結果から、以下を判定してください：
        1. 前回のコマンドが「09_チケット実行と検証」だったか
        2. ユーザーの入力に「OK」「完了」「成功」等の完了を示す言葉があるか
        3. 完了したtask_idは何か（check_*.mdのファイル名から推定）
        4. そのタスクをDONEにすべきか
        
        出力形式: {"has_previous_completion": true/false, "completed_task_id": "T_XXX", "should_update_status": true/false, "completion_evidence": "根拠"}
      store_as: "prev_completion"
    
    - name: "handle_previous_completion"
      action: "conditional_execute"
      condition: "{{prev_completion.should_update_status}}"
      steps:
        - name: "update_previous_task_status"
          action: "update_yaml_file"
          file: "Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/dev_tasks.yaml"
          update_instruction: |
            {{prev_completion.completed_task_id}} のstatusを「DONE」に変更してください。
            変更理由: {{prev_completion.completion_evidence}}
          message: "前回完了タスク（{{prev_completion.completed_task_id}}）のステータスをDONEに更新しました"
        
        - name: "display_status_update"
          action: "display"
          content: |
            ✅ 前回タスク完了を検出・反映
            - 完了タスク: {{prev_completion.completed_task_id}}
            - 根拠: {{prev_completion.completion_evidence}}
            - ステータス: TODO → DONE に更新
    
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}", "{{read_files(find_files(patterns=['**/dev_tasks.yaml','**/story_map.yaml','**/screen_map.yaml','**/acceptance_user_guide_*.md','**/check_*.md']))}}"]
      instructions: |
        直近のスレッド/生成物から、task_idの候補、関連ストーリー、関連スクリーン、受け入れ基準の要点を1-3行で抽出してください。
        出力は箇条書きで簡潔に。
      store_as: "auto_ticket_defaults"
    - name: "prefill_suggestions"
      action: "display"
      content: |
        🔎 自動候補
        {{auto_ticket_defaults}}
    
    # トータル開発仕様書の読み込み（コンテキスト維持）
    - name: "load_total_development_spec"
      action: "read_file"
      file: "Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/total_development_spec.md"
      message: "トータル開発仕様書を読み込み、何を作ろうとしているかを再確認します"
      store_as: "total_spec"
    
    - name: "display_development_context"
      action: "display"
      content: |
        📋 開発コンテキスト（total_development_spec.md より）
        
        **🎯 今回作るもの**: {{total_spec | extract_section: "プロダクト概要"}}
        **👤 誰のために**: {{total_spec | extract_section: "ターゲットペルソナ"}}
        **🎨 どんな画面**: {{total_spec | extract_section: "UI設計概要"}}
        **🔧 技術構成**: {{total_spec | extract_section: "技術構成"}}
        **✅ 完成時の状態**: {{total_spec | extract_section: "完成時の期待状態"}}
    
    # 現在の進捗状況表示
    - name: "show_current_progress"
      action: "analyze"
      data: ["{{read_file('Flow/' + today + '/' + flow_dir + '/07_開発タスク分解/dev_tasks.yaml')}}"]
      instructions: |
        dev_tasks.yamlを読み込み、現在の進捗状況を要約してください：
        - DONE: 完了済みタスク数とID一覧
        - IN_PROGRESS: 実行中タスク数とID一覧  
        - TODO: 未着手タスク数とID一覧
        - 進捗率: DONE数 / 全タスク数 × 100%
        - 推奨次タスク: 依存関係を考慮した次の候補（上位3個）
        
        出力は表示用の構造化テキストで。
      store_as: "current_progress"
    
    - name: "display_progress_summary"
      action: "display"
      content: |
        📊 現在の開発進捗
        {{current_progress}}
    
    - name: "show_dev_tasks_overview"
      action: "display"
      content: |
        📄 対象ファイル
        - **`Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/total_development_spec.md`**（重要・必読）
        - `Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/dev_tasks.yaml`
        - `Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/dev_tasks_order.md`
        - `Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/dev_runbook.md`
        - `Flow/{{today}}/{{flow_dir}}/04_ストーリーマップ/story_map.yaml`
        - `Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/screen_map.yaml`
        - `Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/design/SCx_design.yaml`, `design/SCx_wire_aa.md`
        - `Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/drawio/SCx.drawio`
        
        推奨実行順: Common → Stories → Non-Functional
        まず `T_COMMON_ENV` → `T_COMMON_STORAGE` を完了してからストーリーに進んでください。
    
    - name: "ask_task_id"
      action: "ask_questions_with_template"
      template: |
        === 選択 ===
        1. 実行する task_id（例: T_COMMON_ENV）
        → 【あなたの回答】：
        2. 関連ストーリー（任意: ST1/ST2/ST3。未入力時は dev_tasks.yaml から自動推測）
        → 【あなたの回答】：
        =====================================
    
    - name: "warn_if_common_unfinished"
      action: "display"
      content: |
        ⚠️ 注意: Commonタスク未完了のままStories/NFRを選ぶと、依存不足で失敗することがあります。
        未完の場合は先に `T_COMMON_ENV` → `T_COMMON_STORAGE` を実行してください。（この注意は自動判定ではなく運用ガイドです）

        💻 実行環境メモ（Windows対応・手動時の参考）
        Bash:
        ```bash
        # ブラウザ起動（VS Code Live Server 等）
        # Console: Chrome → Option+Command+I → Console
        ```
        PowerShell:
        ```powershell
        # ブラウザ起動は各自の環境に依存（Edge/Chromeのパス）
        # Console: F12 → Console（開発者ツール）
        ```
    
    - name: "create_task_folder"
      action: "execute_shell"
      command: |
        mkdir -p "Flow/{{today}}/{{flow_dir}}/08_チケット開始/{{task_id}}"
    
    # --- ここから: 関連スクリーンの設計読み込みと生成物出力 ---
    - name: "resolve_story_and_screens"
      action: "display"
      content: |
        🔗 紐づけ解決
        - 参照: `../07_開発タスク分解/dev_tasks.yaml` から {{task_id}} の所属ストーリーを自動推測（例: ST2）。未取得時は入力の関連ストーリーを使用。
        - 参照: `../../05_UIワイヤーフレーム/screen_map.yaml > mapping` から STx → SCy を解決
        - 結果: {{resolved_story}} → {{resolved_screens}}（例: ST2 → [SC1]）
    
    - name: "emit_design_bundle"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/08_チケット開始/{{task_id}}/work_{{task_id}}__design.md"
      content: |
        # 設計バンドル {{task_id}}
        
        ## 関連ストーリー/スクリーン
        - Story: {{resolved_story}}
        - Screens: {{resolved_screens}}
        
        ## スクリーン設計（YAML 抜粋）
        ### SC1
        ```yaml
        {{sc1_design_yaml}}
        ```
        
        ### SC2
        ```yaml
        {{sc2_design_yaml}}
        ```
        
        ## AAワイヤー
        ### SC1
        {{sc1_wire_aa}}
        
        ### SC2
        {{sc2_wire_aa}}
        
        ## Draw.io
        - 図面: `../../05_UIワイヤーフレーム/drawio/SC1.drawio`, `SC2.drawio`（存在する場合）
        - 出力先（任意）: `drawio/exports/SC1.png`, `drawio/exports/SC2.png`
        
        ## 反映メモ
        - コンポーネントIDとDOMの紐付け
        - アクセシビリティ（focus順/ARIA）の確認
        - スクリーンフロー整合性（`../../05_UIワイヤーフレーム/screen_flow.yaml`）
    
    - name: "create_instrumentation_snippets"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/08_チケット開始/{{task_id}}/instrumentation_snippets.md"
      content: |
        # Console観測ログ用スニペット
        
        ## クリック/入力
        ```js
        addBtn.addEventListener('click', () => {
          console.log('[click] add', { text: input.value });
        });
        list.addEventListener('click', (e) => {
          console.log('[click] list', { targetRole: e.target?.role || e.target?.tagName });
        });
        ```
        
        ## モード切替（aria-pressed）
        ```js
        function toggleMode(btn){
          const next = btn.getAttribute('aria-pressed') !== 'true';
          btn.setAttribute('aria-pressed', String(next));
          console.log('[state] mode.ariaPressed =', next);
        }
        modeBtn.addEventListener('click', () => toggleMode(modeBtn));
        ```
        
        ## 候補/状態の可視化
        ```js
        console.log('[state] candidates =', candidates.slice(0, 3));
        console.log('[state] storage.size =', (JSON.parse(localStorage.getItem('tasks')||'[]')).length);
        ```
        
        ## 例外の捕捉
        ```js
        try { /* 処理 */ } catch (err) {
          console.error('[error] exception', err);
        }
        ```
    
    - name: "create_work_note"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/08_チケット開始/{{task_id}}/work_{{task_id}}.md"
      content: |
        # 作業メモ {{task_id}}
        
        ## 目的
        {{task_title}}
        
        ## 受け入れ基準（AC）
        {{acceptance_criteria}}
        
        ## 実装方針（HTML/CSS/JS, localStorage）
        - HTML:
        - CSS:
        - JS:
        
        ## インストルメンテーション（Console出力）
        - スニペット: `./instrumentation_snippets.md` を参照して貼り付け
        - 最低限の観測ログ:
          - `[click] add { text: "..." }`
          - `[state] mode.ariaPressed = true/false`
          - `[state] candidates = [...]`
          - `[error] exception ...`（例外時のみ）
        
        ## 参照（設計）
        - **トータル開発仕様書**: `../../07_開発タスク分解/total_development_spec.md`（必読・全体コンテキスト）
        - 設計バンドル: `./work_{{task_id}}__design.md`
        - Story/Screen対応: `../../05_UIワイヤーフレーム/screen_map.yaml`
        - スクリーンフロー: `../../05_UIワイヤーフレーム/screen_flow.yaml` / `screen_flow_mermaid.md`
        - Draw.io: `../../05_UIワイヤーフレーム/drawio/SC1.drawio`, `SC2.drawio`（該当がある場合）
        
        ## タスク分解（小さく）
        - [ ] dev/src/index.html にDOMにプレースホルダを追加
        - [ ] dev/src/app.js にエントリポイントを追加
        - [ ] dev/src/styles.css にスタイルを追加
        - [ ] 観測ログを追加（上記スニペット）
        - [ ] 動作確認（手動）
        
        ## 動作確認観点（最低限）
        - 表示できる / 主要要素が見える
        - クリック/入力に反応する
        - CSSが適用されている
        - Consoleにエラーが出ていない
        
        ## 実装先
        - **メインディレクトリ**: `../../dev/src/`
        - **動作確認**: `../../dev/src/index.html` をブラウザで開く
        
        ---
        
        ## ユーザー受け入れ確認ガイド（ブラウザ手順）
        以下を順にご確認ください（Chrome推奨）。
        
        1) ファイルを開く（Cursor からの開き方）
        - エクスプローラで `../../dev/src/index.html` を右クリック → 「Open in Browser」 または Live Server で起動
        - ブラウザが起動したら、開発者ツールを開く（Chrome: Option+Command+I）→ Console タブを選択
        
        2) 表示確認（タイトル/見出し/入力+追加/CSS）
        3) 操作確認（追加/チェック/エラー無し）
        4) 期待ログの確認
        - `[click] add` が出力される
        - `[state] mode.ariaPressed = true/false` が押下に応じて切替表示
        - 必要に応じ `[state] candidates = [...]` などの状態ログ
        5) ACをYes/Noで判定
    
    - name: "touch_target_files"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/08_チケット開始/{{task_id}}/dev_tasks_{{task_id}}.md"
      content: |
        # 実装タスクリスト {{task_id}}
        - [ ] HTML編集: index.html
        - [ ] CSS編集:  styles.css
        - [ ] JS編集:   app.js
        - [ ] 観測ログを追加（console.log / console.error）
    
    - name: "create_user_acceptance_guide"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/08_チケット開始/{{task_id}}/acceptance_user_guide_{{task_id}}.md"
      content: |
        # ユーザー受け入れ確認ガイド（{{task_id}}）
        
        推奨実行順に留意してください: Common → Stories → Non-Functional。
        まずCommon（`T_COMMON_ENV` → `T_COMMON_STORAGE`）を完了してから本タスクを実行してください。
        
        以降はブラウザ確認手順：
        
        1) ファイルを開く（Cursor からの開き方）
        - エクスプローラで `../../dev/src/index.html` を右クリック → 「Open in Browser」 または Live Server で起動
        - ブラウザが起動したら、開発者ツールを開く（Chrome: Option+Command+I）→ Console タブを選択
        
        2) 表示確認（タイトル/見出し/入力+追加/CSS）
        3) 操作確認（追加/チェック/エラー無し）
        4) 期待ログの確認
        - `[click] add` が出力される
        - `[state] mode.ariaPressed = true/false` が押下に応じて切替表示
        - 必要に応じ `[state] candidates = [...]` などの状態ログ
        5) ACをYes/Noで判定
```

## 次のコマンド
→ `実装_チケット実行と検証` で実装支援と受け入れチェック
