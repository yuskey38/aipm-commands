# 12 成果物パッケージング（AIPMハッカソン）

## 目的
最終成果物（dev/src/のHTML/CSS/JS）、トータル開発仕様書、ドキュメント（README、実装メモ、チェック結果、Marpスライド等）をStock側のプロジェクト配下に集約（コピー）します。初期化時に作成した Program/Project 配下に、提出フォルダを生成し、Flow の作業成果を整理してパッケージングします。

## 実行手順
```yaml
- trigger: "(提出_成果物パッケージング|PackageArtifacts)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['Flow/**/hackathon/**/README.md','Flow/**/hackathon/**/slides_*.md','Flow/**/hackathon/**/check_*.md']))}}"]
      instructions: |
        Program/Project/flow_dir/submit_folder/提出メモ/目的/背景 の初期値候補を抽出し、display用に箇条書きで提示してください。
      store_as: "auto_submit_seed"
    - name: "prefill_submit_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_submit_seed}}
    # 成果物の事前確認
    - name: "check_artifacts"
      action: "display"
      content: |
        📋 パッケージング対象の確認
        
        **必須ファイル**:
        - ✅ dev/src/index.html（実装コード）
        - ✅ 07_開発タスク分解/total_development_spec.md（開発仕様書）
        - ✅ 11_Marpスライド生成/slides_mvp.marp.md（発表資料）
        
        **推奨ファイル**:
        - Discovery設計資料（ペルソナ・課題・ソリューション・ストーリー・UI）
        - 実装記録（チケット作業メモ・検証結果・進捗レポート）
        
        上記ファイルがない場合、該当コマンドを先に実行してください。
    
    - name: "ask_submission_meta"
      action: "ask_questions_with_template"
      template: |
        === 提出メタ情報 ===
        1. Program（既定: AIPM_Hackathon）
        →
        2. Project（アプリ名。例: SmartTodo-Mama）
        →
        3. Flowの作業ディレクトリ名（既定: hackathon_new）
        →
        4. 提出フォルダ名（既定: submit_{{today}}）
        →
        5. READMEに記載する「提出メモ」（任意）
        →
        6. 目的（現時点の仮説。既定は初期化時の内容で可）
        →
        7. 背景（既定は初期化時の内容で可）
        →
        =====================================
    
    - name: "wait_meta"
      action: "wait_for_all_answers"
    
    - name: "confirm_packaging"
      action: "confirm"
      message: "Flow/{{today}}/{{flow_dir}} の中身を Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}} へコピーします。よろしいですか？"
    
    - name: "show_parallel_shells"
      action: "display"
      content: |
        💻 実行環境メモ（Windows対応・手動時の参考）

        Bash (macOS/Linux):
        ```bash
        mkdir -p "Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}}"
        cp -R "Flow/{{today}}/{{flow_dir}}"/* "Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}}/"
        ```

        PowerShell (Windows/macOSのpwsh):
        ```powershell
        $dst = "Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}}"
        New-Item -ItemType Directory -Path $dst -Force | Out-Null
        Copy-Item -Path "Flow/{{today}}/{{flow_dir}}/*" -Destination $dst -Recurse -Force
        ```

    # 1) 提出先フォルダ生成
    - name: "create_submission_dir"
      action: "execute_shell"
      command: |
        mkdir -p "Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}}"
    
    # 2) Flow → Stock へ成果物をコピー（フォルダごと）
    - name: "copy_flow_to_stock"
      action: "execute_shell"
      command: |
        cp -R "Flow/{{today}}/{{flow_dir}}"/* "Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}}/"
    
    # 3) 提出用README（Stock側）を生成（既存READMEは残し、提出用README_submission.mdを作る）
    - name: "write_submission_readme"
      action: "create_markdown_file"
      path: "Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}}/README_submission.md"
      content: |
        # 提出パッケージ README
        
        - Program: {{program_id}}
        - Project: {{project_id}}
        - 提出フォルダ: submissions/{{submit_folder}}
        - 作成日: {{today}}
        
        ## 概要
        本フォルダは Flow の作業成果をそのまま集約した提出パッケージです。
        
        ## 目的（現時点の仮説）
        {{purpose}}
        
        ## 背景
        {{background}}
        
        ## 提出メモ
        {{submission_note}}
        
        ## 参考（想定主要ファイル）
        
        **実装コード（dev/src/）:**
        - dev/src/index.html / dev/src/styles.css / dev/src/app.js
        - dev/README.md（開発ディレクトリ説明）
        
        **開発仕様・計画:**
        - 07_開発タスク分解/total_development_spec.md（トータル開発仕様書）
        - 07_開発タスク分解/dev_tasks.yaml（開発タスク一覧）
        - 07_開発タスク分解/dev_tasks_order.md / dev_runbook.md
        
        **Discovery設計:**
        - 01_ペルソナ作成/persona_todo.md + experience_map.yaml
        - 02_課題定義/problem_map.yaml + customer_problem_map.yaml
        - 03_ソリューションマップ/solution_map.yaml
        - 04_ストーリーマップ/story_map.yaml
        - 05_UIワイヤーフレーム/screen_map.yaml + screen_flow.yaml + ui_wireframe_todo.md
        
        **実装記録:**
        - 08_チケット開始/*/work_*.md（作業メモ）
        - 09_チケット実行と検証/check_*.md + implementation_*.md
        - 10_タスクリファイン/progress_report.md
        
        **発表資料:**
        - 11_Marpスライド生成/slides_mvp.marp.md
        
        実体は下記マニフェストを参照してください。
    
    # 4) マニフェスト雛形を生成（実体列挙は必要に応じて追記）
    - name: "write_manifest"
      action: "create_markdown_file"
      path: "Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}}/MANIFEST.md"
      content: |
        # MANIFEST（{{today}}）
        
        ## 実装コード
        - dev/src/index.html（メインHTML）
        - dev/src/styles.css（スタイルシート）
        - dev/src/app.js（アプリケーションロジック）
        - dev/README.md（開発ディレクトリ説明）
        
        ## 開発仕様・計画
        - 07_開発タスク分解/total_development_spec.md（トータル開発仕様書）
        - 07_開発タスク分解/dev_tasks.yaml（開発タスク一覧）
        
        ## Discovery設計
        - 01_ペルソナ作成/persona_todo.md + experience_map.yaml
        - 02_課題定義/problem_map.yaml + customer_problem_map.yaml
        - 03_ソリューションマップ/solution_map.yaml
        - 04_ストーリーマップ/story_map.yaml
        - 05_UIワイヤーフレーム/screen_map.yaml + screen_flow.yaml
        
        ## 実装記録
        - 09_チケット実行と検証/implementation_*.md
        - 10_タスクリファイン/progress_report.md
        
        ## 発表資料
        - 11_Marpスライド生成/slides_mvp.marp.md
    
    - name: "notify_done"
      action: "display"
      content: |
        ✅ 提出パッケージを作成しました。
        
        📁 **提出先**: Stock/programs/{{program_id}}/projects/{{project_id}}/submissions/{{submit_folder}}/
        
        📋 **主要成果物**:
        - **実装コード**: dev/src/index.html, styles.css, app.js
        - **開発仕様書**: 07_開発タスク分解/total_development_spec.md
        - **設計資料**: Discovery フェーズの全YAML/MD
        - **実装記録**: チケット作業メモ・検証結果
        - **発表資料**: 11_Marpスライド生成/slides_mvp.marp.md
        
        🎯 **デモ準備**: dev/src/index.html をブラウザで開いてライブデモ可能
        
        次は `/aipm/aipm_9_00_hackathon_GoogleDrive提出` を実行して提出してください。
```

## 次のコマンド
→ `@aipm_9_00_hackathon_GoogleDrive提出` で提出完了
