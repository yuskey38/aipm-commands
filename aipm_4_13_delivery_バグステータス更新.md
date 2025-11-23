---
name: バグステータス更新（バグライフサイクル管理）
---

# 13 : バグステータス更新 - バグのライフサイクル管理

## 前提
- バグ登録（12_バグ登録）でバグが登録済み
- bug_tracking/bugs.yaml にバグエントリが存在

## 目的
- バグのステータスをライフサイクルに沿って更新
- 修正進捗の追跡とコメント記録
- 担当者・修正バージョンの管理
- リリース判定のための状態管理

## バグライフサイクル
```
New → Open → In Progress → Fixed → Verified → Closed
                                              ↓
                                          Won't Fix
```

## 実行手順
```yaml
- trigger: "(バグステータス更新|Bug Status|ステータス変更|バグ更新)"
  priority: high
  steps:
    # ========================================
    # Phase 1: bugs.yaml読み込み・バグ一覧表示
    # ========================================
    - name: "load_bugs"
      action: "read_file"
      file: "Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/bug_tracking/bugs.yaml"
      store_as: "bugs_data"
    
    - name: "display_bug_list"
      action: "display"
      content: |
        📋 登録済みバグ一覧:
        
        {{#each bugs_data.bugs}}
        {{status_icon(this.status)}} {{this.id}}: {{this.title}} ({{this.status}}, {{this.severity}})
        {{/each}}
    
    # ========================================
    # Phase 2: 更新対象バグ選択
    # ========================================
    - name: "select_bug"
      action: "ask_question"
      question: "更新するバグIDを入力してください（例: BUG-001）:"
      required: true
      store_as: "target_bug_id"
    
    - name: "find_bug"
      action: "filter_array"
      array: "{{bugs_data.bugs}}"
      condition: "id == {{target_bug_id}}"
      store_as: "target_bug"
    
    - name: "display_current_status"
      action: "display"
      content: |
        🔍 {{target_bug.id}}: {{target_bug.title}}
        
        現在のステータス: {{target_bug.status}}
        深刻度: {{target_bug.severity}}
        優先度: {{target_bug.priority}}
        発見日: {{target_bug.found_date}}
        発見テスト: {{target_bug.found_in_test}}
        担当者: {{target_bug.assignee || "未アサイン"}}
    
    # ========================================
    # Phase 3: 新しいステータス入力
    # ========================================
    - name: "input_new_status"
      action: "ask_questions"
      questions:
        - key: "new_status"
          question: |
            新しいステータスを選択してください:
            1) new - 新規登録
            2) open - トリアージ完了
            3) in_progress - 修正作業中
            4) fixed - 修正完了（QA検証待ち）
            5) verified - QA検証完了（リリース待ち）
            6) closed - リリース完了
            7) wont_fix - 修正しない判断
          options: ["new", "open", "in_progress", "fixed", "verified", "closed", "wont_fix"]
          required: true
        
        - key: "comment"
          question: "コメントを入力してください（状況説明、修正内容等）:"
          required: false
        
        - key: "assignee"
          question: "担当者を入力してください（in_progress/fixedの場合）:"
          required: false
          condition: "{{new_status}} == 'in_progress' || {{new_status}} == 'fixed'"
        
        - key: "fix_version"
          question: "修正予定バージョンを入力してください（例: v1.0.1）:"
          required: false
          condition: "{{new_status}} == 'fixed'"
        
        - key: "fix_commit"
          question: "修正コミットハッシュを入力してください（fixedの場合）:"
          required: false
          condition: "{{new_status}} == 'fixed'"
      
      store_as: "update_info"
    
    # ========================================
    # Phase 4: bugs.yaml更新
    # ========================================
    - name: "update_bugs_yaml"
      action: "execute_python"
      script: |
        import yaml
        from datetime import datetime
        from pathlib import Path
        
        bugs_yaml = Path("Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/bug_tracking/bugs.yaml")
        
        # データ読み込み
        with open(bugs_yaml, 'r') as f:
            data = yaml.safe_load(f)
        
        # バグを検索して更新
        for bug in data['bugs']:
            if bug['id'] == "{{target_bug_id}}":
                bug['status'] = "{{update_info.new_status}}"
                
                # コメント追加
                if "{{update_info.comment}}":
                    if 'comments' not in bug:
                        bug['comments'] = []
                    
                    bug['comments'].append({
                        'date': datetime.now().isoformat(),
                        'author': 'QA Engineer',
                        'comment': "{{update_info.comment}}"
                    })
                
                # 担当者設定
                if "{{update_info.assignee}}":
                    bug['assignee'] = "{{update_info.assignee}}"
                
                # fixedステータスの場合
                if "{{update_info.new_status}}" == "fixed":
                    if not bug.get('fix_date'):
                        bug['fix_date'] = datetime.now().strftime('%Y-%m-%d')
                    if "{{update_info.fix_version}}":
                        bug['fix_version'] = "{{update_info.fix_version}}"
                    if "{{update_info.fix_commit}}":
                        bug['fix_commit'] = "{{update_info.fix_commit}}"
                
                break
        
        # メタデータ更新
        data['metadata']['last_updated'] = datetime.now().isoformat()
        data['metadata']['open_bugs'] = sum(
            1 for b in data['bugs'] 
            if b['status'] in ['new', 'open', 'in_progress']
        )
        data['metadata']['fixed_bugs'] = sum(
            1 for b in data['bugs'] 
            if b['status'] in ['fixed', 'verified']
        )
        data['metadata']['closed_bugs'] = sum(
            1 for b in data['bugs'] 
            if b['status'] == 'closed'
        )
        
        # 保存
        with open(bugs_yaml, 'w') as f:
            yaml.dump(data, f, allow_unicode=True, default_flow_style=False, sort_keys=False)
        
        print(f"✅ {bug['id']} のステータスを {bug['status']} に更新しました")
    
    # ========================================
    # Phase 5: 完了通知
    # ========================================
    - name: "notify_completion"
      action: "notify"
      message: |
        ✅ バグステータス更新完了
        
        バグID: {{target_bug_id}}
        新ステータス: {{update_info.new_status}}
        
        📊 現在の統計:
        - Open: {{metadata.open_bugs}}
        - Fixed/Verified: {{metadata.fixed_bugs}}
        - Closed: {{metadata.closed_bugs}}
        
        📝 次のステップ:
        - `/aipm/aipm_4_14_delivery_バグレポート生成` でサマリー確認
        - fixedの場合: リグレッションテスト追加を検討
        - verifiedの場合: リリース判定に進む
```

## 成果物
- `bug_tracking/bugs.yaml` - バグステータス更新
- `bug_tracking/bug_reports/{{BUG_ID}}.md` - レポート自動更新（推奨）

## 次のステップ
- fixedステータスの場合: リグレッションテスト追加
- verifiedステータスの場合: `/aipm/aipm_4_15_delivery_リリース判定`
- `/aipm/aipm_4_14_delivery_バグレポート生成` で全体状況確認






















