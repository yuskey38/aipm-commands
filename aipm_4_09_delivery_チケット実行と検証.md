# 09 : チケット実行と検証（AIPMハッカソン）

## 目的
対象タスク（task_id）の実装を進め、受け入れ基準に沿ってチェックリストで検証します。08で生成したユーザー受け入れガイドを基に、確認手順をチャットで案内します。

## 実行手順
```yaml
- trigger: "(実装_チケット実行と検証|TicketRun)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/dev_tasks.yaml','**/08_チケット開始/**/work_*.md','**/08_チケット開始/**/acceptance_user_guide_*.md','**/09_チケット実行と検証/check_*.md']))}}"]
      instructions: |
        直近の作業から、本コマンドで実行すべき task_id 候補を抽出し、優先順で1-3件提示してください。各候補に関連ストーリー/確認観点があれば併記。
      store_as: "auto_run_candidates"
    - name: "prefill_suggestions"
      action: "display"
      content: |
        🔎 実行候補（自動）
        {{auto_run_candidates}}
    
    # トータル開発仕様書の読み込み（実装方向性の確認）
    - name: "load_total_development_spec"
      action: "read_file"
      file: "Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/total_development_spec.md"
      message: "トータル開発仕様書を読み込み、実装の目的と方向性を再確認します"
      store_as: "total_spec"
    
    - name: "display_implementation_context"
      action: "display"
      content: |
        📋 実装コンテキスト（total_development_spec.md より）
        
        **🎯 何を作るか**: {{total_spec | extract_section: "プロダクト概要"}}
        **👤 誰の課題を解決するか**: {{total_spec | extract_section: "ターゲットペルソナ"}}
        **📊 どのタスクが全体の何を担うか**: {{total_spec | extract_section: "開発タスク概要"}}
        **✅ 最終的にどうなれば成功か**: {{total_spec | extract_section: "完成時の期待状態"}}
        
        ---
        **重要**: 実装中は常にペルソナの課題解決を意識し、仕様書の期待状態に向かって進めてください。
    
    - name: "ask_task_id"
      action: "ask_question"
      question: "実行する task_id は？（例: T_COMMON_ENV）"
      required: true
      store_as: "task_id"
    
    - name: "show_plan"
      action: "display"
      content: |
        ✅ 手順
        1) トータル開発仕様書とタスク詳細を確認
        2) **実装を開始**（HTML/CSS/JSを実際に編集・作成）
        3) 動作確認
        4) 受け入れ基準に沿ってチェック
        5) OKなら dev_tasks.yaml 上の該当タスクをDONEとして扱い、次のタスクへ
    
    # 実装開始（AI支援による実際のコード生成・編集）
    - name: "start_implementation"
      action: "analyze"
      data: ["{{total_spec}}", "{{read_files(find_files(patterns=['**/08_チケット開始/{{task_id}}/work_{{task_id}}.md','**/08_チケット開始/{{task_id}}/work_{{task_id}}__design.md']))}}"]
      instructions: |
        トータル開発仕様書とタスク詳細を基に、{{task_id}} の実装を開始してください。
        
        **実装方針**:
        1. トータル開発仕様書の技術構成・ペルソナ課題・ストーリー要件を必ず反映
        2. タスクの受け入れ基準を満たす具体的なコード実装
        3. 観測ログ（Console出力）を適切に配置
        4. 既存ファイルがある場合は差分更新、ない場合は新規作成
        
        **出力形式**:
        - 変更/作成するファイルごとに、ファイルパスと完全なコード内容を提示
        - HTML: 構造とアクセシビリティを考慮
        - CSS: レスポンシブとユーザビリティを考慮  
        - JS: 状態管理とログ出力を考慮
        
        JSON形式で返す: {"files": [{"path": "...", "content": "...", "description": "..."}, ...], "implementation_summary": "実装概要"}
      store_as: "implementation"
    
    - name: "execute_implementation"
      action: "display"
      content: |
        🔧 実装開始: {{task_id}}
        
        {{implementation.implementation_summary}}
        
        以下のファイルを作成・更新します：
        {{#each implementation.files}}
        
        ### {{path}}
        {{description}}
        
        ```{{file_extension}}
        {{content}}
        ```
        {{/each}}
    
    - name: "confirm_implementation"
      action: "confirm"
      message: "上記の実装内容でファイルを作成・更新しますか？"
    
    # 実際のファイル作成・更新（devフォルダに配置）
    - name: "create_or_update_files"
      action: "create_multiple_files"
      files: "{{implementation.files}}"
      base_path: "Flow/{{today}}/{{flow_dir}}/dev/src/"
      message: "実装ファイルをdev/src/に作成・更新しています..."
    
    # 実装完了の記録
    - name: "record_implementation_completion"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/09_チケット実行と検証/implementation_{{task_id}}.md"
      content: |
        # 実装完了記録 {{task_id}}
        
        ## 実装内容
        {{implementation.implementation_summary}}
        
        ## 作成・更新ファイル
        {{#each implementation.files}}
        ### {{path}}
        - 配置先: `dev/src/{{path}}`
        - 内容: {{description}}
        {{/each}}
        
        ## 動作確認手順
        1. `Flow/{{today}}/{{flow_dir}}/dev/src/index.html` をブラウザで開く
        2. 開発者ツール（F12）→ Console タブを表示
        3. 動作・ログ・エラーを確認
        
        ## 参照
        - トータル開発仕様書: `../07_開発タスク分解/total_development_spec.md`
    
    - name: "show_user_acceptance_steps"
      action: "display"
      content: |
        🧪 ユーザー受け入れ確認（ガイド）
        ガイドファイル: `Flow/{{today}}/{{flow_dir}}/08_チケット開始/acceptance_user_guide_{{task_id}}.md`
        
        以下を順にご確認ください（Chrome推奨）。
        - `index.html` をブラウザで開く → DevTools Console 表示
        - 表示確認: タイトル/見出し/入力+追加ボタン/CSSの適用
        - 操作確認: 入力→追加→リスト反映、チェックON/OFF、Consoleエラーなし
        - 期待ログの確認:
          - `[click] add` が出力される
          - `[state] mode.ariaPressed = true/false` が押下毎に切替表示
          - 必要に応じ `[state] candidates = [...]` 等の状態ログ
        - 受け入れ基準（AC）をYes/Noで判定
        
        判定結果を次のフォームで回答してください。
    
    # 実装完了確認とブラウザテスト案内
    - name: "guide_browser_testing"
      action: "display"
      content: |
        🌐 ブラウザテスト実行
        
        1. **ファイルを開く**:
           - `Flow/{{today}}/{{flow_dir}}/dev/src/index.html` をブラウザで開く
           - 開発者ツール（F12）→ Console タブを表示
        
        2. **動作確認**:
           - 画面表示: タイトル・UI要素が正しく表示される
           - 操作確認: ボタンクリック・入力・モード切替などが動作する
           - ログ確認: Console に期待されるログが出力される
           - エラー確認: Console にエラーが出ていない
        
        3. **受け入れ基準チェック**:
           - トータル開発仕様書の「完成時の期待状態」と照らし合わせる
           - ペルソナの課題解決に寄与しているかを確認
    
    - name: "acceptance_check"
      action: "ask_questions_with_template"
      template: |
        === 受け入れチェック（{{task_id}}）===
        1. 実装実行結果（成功/失敗/部分的）
        → 【あなたの回答】：
        
        2. 実装概要（何を変更/追加したか）
        → 【あなたの回答】：
        
        3. ファイル作成確認（dev/src/配下の作成されたファイルパス）
        → 【あなたの回答】：
        
        4. 受け入れ基準の確認（1つずつYes/No）
        - 基準1（表示/操作/エラーなし）： 
        - 基準2（機能要件）： 
        - （任意）基準3： 
        
        5. 期待ログの確認（Yes/No）
        - `[click] add` の出力： 
        - `[state] mode.ariaPressed` の切替表示： 
        - （任意）その他の状態ログ： 
        
        6. 動作確認メモ
        → 【あなたの回答】：
        
        7. 未解決の懸念（あれば）
        → 【あなたの回答】：
        =====================================
    
    - name: "write_check_result"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/09_チケット実行と検証/check_{{task_id}}.md"
      content: |
        # 受け入れチェック結果 {{task_id}}
        
        ## 実装実行結果
        {{impl_result}}
        
        ## 概要
        {{impl_summary}}
        
        ## 作成ファイル
        {{created_files}}
        
        ## 基準判定
        - 基準1: {{ac1}}
        - 基準2: {{ac2}}
        - 基準3: {{ac3}}
        
        ## 期待ログ確認
        - click add: {{log_click_add}}
        - mode.ariaPressed: {{log_mode_toggle}}
        - その他: {{log_others}}
        
        ## 動作確認メモ
        {{test_notes}}
        
        ## 懸念
        {{risks}}
        
        ## トータル仕様書との整合性
        - ペルソナ課題解決への寄与: {{persona_alignment}}
        - ストーリー要件の達成度: {{story_alignment}}
        - 技術構成の準拠: {{tech_alignment}}
    
    - name: "show_next_step"
      action: "display"
      content: |
        📝 次ステップ
        - `../07_開発タスク分解/dev_tasks.yaml` の該当タスク（{{task_id}}）をDONE扱いにして、依存タスクに進んでください。
        - 連続実行する場合は、再度 `08_チケット開始` で次の task_id を選択してください。
```

## 次のコマンド
→ 別の `08_チケット開始` / `09_チケット実行と検証` を繰り返す
→ 完了後は `12_成果物パッケージング` へ
