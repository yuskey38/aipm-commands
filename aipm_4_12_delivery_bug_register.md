# 12 Delivery Bug Register
name: バグ登録（QA中のバグ・改善点登録）
---

# 12 : バグ登録 - QA実行中のバグ・改善点を体系的に記録

## 前提
- QA実行（11_QA実行）が実施中または完了済み
- bug_tracking/bugs.yaml が存在
- 統合テスト・ユニットテスト実行中にバグ・改善点を発見

## 目的
- QA実行中に発見したバグ・改善点を即座に記録
- 再現手順・環境情報を含む詳細なバグレポート作成
- bugs.yamlへの自動追記と個別レポートファイル生成
- バグID自動採番（BUG-XXX, IMP-XXX, TF-XXX）

## 実行手順
```yaml
- trigger: "(バグ登録|Bug Report|改善登録|テスト失敗登録)"
  priority: high
  steps:
    # ========================================
    # Phase 1: バグ情報収集
    # ========================================
    - name: "collect_bug_info"
      action: "ask_questions"
      questions:
        - key: "bug_type"
          question: |
            バグのタイプを選択してください:
            1) bug - 実装の不具合
            2) improvement - 改善提案
            3) test_failure - テスト失敗
          options: ["bug", "improvement", "test_failure"]
          required: true
        
        - key: "title"
          question: "タイトル（1行要約）を入力してください:"
          required: true
        
        - key: "severity"
          question: |
            深刻度を選択してください:
            1) critical - システム停止、データ損失
            2) high - 主要機能が使用不可
            3) medium - 機能の一部が動作しない
            4) low - 軽微な問題、回避策あり
          options: ["critical", "high", "medium", "low"]
          required: true
        
        - key: "priority"
          question: |
            優先度を選択してください:
            1) P0 - 即座に修正（ブロッカー）
            2) P1 - 次リリースまでに修正
            3) P2 - 修正推奨
            4) P3 - 将来対応
          options: ["P0", "P1", "P2", "P3"]
          required: true
        
        - key: "found_in_test"
          question: "発見したテスト名を入力してください（例: scenario1_new_project）:"
          required: true
        
        - key: "description"
          question: "詳細説明を入力してください:"
          required: true
          multiline: true
        
        - key: "reproduction_steps"
          question: |
            再現手順を入力してください（1行ずつ、空行で終了）:
            例）
            1. テストプロジェクトを作成
            2. nit init を実行
            3. nit push を実行
          required: false
          multiline: true
        
        - key: "expected_behavior"
          question: "期待される動作を入力してください:"
          required: false
        
        - key: "actual_behavior"
          question: "実際の動作を入力してください:"
          required: false
        
        - key: "error_log"
          question: "エラーログがあれば貼り付けてください（空欄可）:"
          required: false
          multiline: true
      
      store_as: "bug_info"
    
    # ========================================
    # Phase 2: バグID自動生成
    # ========================================
    - name: "generate_bug_id"
      action: "execute_python"
      script: |
        import yaml
        from pathlib import Path
        
        bugs_yaml = Path("Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/bug_tracking/bugs.yaml")
        
        # バグタイプからプレフィックス決定
        type_prefix = {
            "bug": "BUG",
            "improvement": "IMP",
            "test_failure": "TF"
        }
        prefix = type_prefix.get("{{bug_info.bug_type}}", "BUG")
        
        # 既存バグIDから次のIDを生成
        try:
            with open(bugs_yaml, 'r') as f:
                data = yaml.safe_load(f)
            
            existing_ids = [
                b['id'] for b in data.get('bugs', []) 
                if b['id'].startswith(prefix)
            ]
            
            if existing_ids:
                last_num = max(int(bid.split('-')[1]) for bid in existing_ids)
                next_num = last_num + 1
            else:
                next_num = 1
            
            bug_id = f"{prefix}-{next_num:03d}"
        except:
            bug_id = f"{prefix}-001"
        
        print(f"生成されたバグID: {bug_id}")
      
      store_as: "bug_id"
    
    # ========================================
    # Phase 3: bugs.yamlに追記
    # ========================================
    - name: "append_to_bugs_yaml"
      action: "execute_python"
      script: |
        import yaml
        from datetime import datetime
        from pathlib import Path
        import os
        
        bugs_yaml = Path("Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/bug_tracking/bugs.yaml")
        
        # 既存データ読み込み
        with open(bugs_yaml, 'r') as f:
            data = yaml.safe_load(f)
        
        # 再現手順を配列に変換
        repro_steps = []
        if "{{bug_info.reproduction_steps}}":
            steps_text = "{{bug_info.reproduction_steps}}"
            repro_steps = [line.strip() for line in steps_text.split('\n') if line.strip()]
        
        # 新しいバグエントリ作成
        new_bug = {
            'id': "{{bug_id}}",
            'type': "{{bug_info.bug_type}}",
            'severity': "{{bug_info.severity}}",
            'priority': "{{bug_info.priority}}",
            'status': 'new',
            'title': "{{bug_info.title}}",
            'found_by': os.getenv('USER', 'QA Engineer'),
            'found_date': datetime.now().strftime('%Y-%m-%d'),
            'found_in_test': "{{bug_info.found_in_test}}",
            'found_in_phase': 'integration_test',
            'description': "{{bug_info.description}}",
            'reproduction_steps': repro_steps,
            'expected_behavior': "{{bug_info.expected_behavior}}",
            'actual_behavior': "{{bug_info.actual_behavior}}",
            'environment': {
                'os': os.uname().sysname + ' ' + os.uname().release,
                'python_version': '',
                'nit_version': 'v1.0-dev'
            },
            'affected_components': [],
            'affected_scenarios': [],
            'assignee': '',
            'fix_version': '',
            'fix_commit': '',
            'fix_date': '',
            'comments': [],
            'attachments': []
        }
        
        # バグリストに追加
        if 'bugs' not in data:
            data['bugs'] = []
        data['bugs'].append(new_bug)
        
        # メタデータ更新
        data['metadata']['total_bugs'] = len(data['bugs'])
        data['metadata']['open_bugs'] = sum(
            1 for b in data['bugs'] 
            if b['status'] in ['new', 'open', 'in_progress']
        )
        data['metadata']['last_updated'] = datetime.now().isoformat()
        
        # 保存
        with open(bugs_yaml, 'w') as f:
            yaml.dump(data, f, allow_unicode=True, default_flow_style=False, sort_keys=False)
        
        print(f"✅ bugs.yamlに {new_bug['id']} を追加しました")
    
    # ========================================
    # Phase 4: 個別バグレポート作成
    # ========================================
    - name: "create_bug_report_file"
      action: "create_markdown_file"
      path: "Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/bug_tracking/bug_reports/{{bug_id}}.md"
      content: |
        # {{bug_id}}: {{bug_info.title}}
        
        **タイプ**: {{bug_info.bug_type}}  
        **深刻度**: {{bug_info.severity}}  
        **優先度**: {{bug_info.priority}}  
        **ステータス**: new  
        **発見日**: {{today}}  
        **発見テスト**: {{bug_info.found_in_test}}  
        **発見者**: QA Engineer
        
        ---
        
        ## 📝 説明
        
        {{bug_info.description}}
        
        ---
        
        ## 🔄 再現手順
        
        {{bug_info.reproduction_steps}}
        
        ---
        
        ## ✅ 期待される動作
        
        {{bug_info.expected_behavior}}
        
        ---
        
        ## ❌ 実際の動作
        
        {{bug_info.actual_behavior}}
        
        ---
        
        ## 💻 環境情報
        
        - **OS**: macOS
        - **Python**: 3.13.3
        - **nit Version**: v1.0-dev
        
        ---
        
        ## 📋 エラーログ
        
        ```
        {{bug_info.error_log}}
        ```
        
        ---
        
        ## 📷 スクリーンショット
        
        （添付ファイルをattachments/screenshots/に保存して参照）
        
        ---
        
        ## 💬 調査メモ
        
        <!-- 調査・修正過程のコメントをここに追記 -->
        
        ---
        
        **作成日**: {{today}}  
        **最終更新**: {{today}}  
        **ステータス**: new
    
    # ========================================
    # Phase 5: 完了通知
    # ========================================
    - name: "notify_completion"
      action: "notify"
      message: |
        ✅ バグチケット {{bug_id}} を作成しました
        
        📁 作成されたファイル:
        - YAML: bug_tracking/bugs.yaml
        - レポート: bug_tracking/bug_reports/{{bug_id}}.md
        
        📝 次のステップ:
        1. レポートファイルを開いて詳細を確認・編集
        2. スクリーンショット・ログを添付
        3. `/aipm/aipm_4_14_delivery_bug_report_generate` でサマリー確認
        
        🔧 バグステータス更新:
        `/aipm/aipm_4_13_delivery_bug_status_update`
```

## 成果物
- `bug_tracking/bugs.yaml` - バグエントリ追加
- `bug_tracking/bug_reports/{{BUG_ID}}.md` - 個別バグレポート

## 次のステップ
- バグレポートファイルを編集して詳細を追記
- `/aipm/aipm_4_13_delivery_bug_status_update` でステータス管理
- `/aipm/aipm_4_14_delivery_bug_report_generate` で日次サマリー確認





















