---
name: バグレポート生成（QAサマリー・ダッシュボード）
---

# 14 : バグレポート生成 - QAサマリーと統計レポート自動生成

## 前提
- バグトラッキングシステムが稼働中
- bug_tracking/bugs.yaml にバグデータが存在

## 目的
- 日次QAサマリーレポート生成
- バグ統計・トレンド分析
- Critical/Highバグの可視化
- リリース判定基準の自動評価

## 実行手順
```yaml
- trigger: "(バグレポート生成|Bug Report|QAサマリー|バグ統計)"
  priority: high
  steps:
    # ========================================
    # Phase 1: bugs.yaml読み込み・解析
    # ========================================
    - name: "load_and_analyze_bugs"
      action: "execute_python"
      script: |
        import yaml
        from datetime import datetime
        from pathlib import Path
        from collections import Counter
        
        bugs_yaml = Path("Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/bug_tracking/bugs.yaml")
        
        with open(bugs_yaml, 'r') as f:
            data = yaml.safe_load(f)
        
        bugs = data.get('bugs', [])
        metadata = data.get('metadata', {})
        
        # 統計計算
        total = len(bugs)
        by_status = Counter(b['status'] for b in bugs)
        by_severity = Counter(b['severity'] for b in bugs)
        by_type = Counter(b['type'] for b in bugs)
        by_phase = Counter(b.get('found_in_phase', 'unknown') for b in bugs)
        
        # Critical/Highバグ（未解決）
        critical_high_open = [
            b for b in bugs 
            if b.get('severity') in ['critical', 'high'] 
            and b.get('status') not in ['verified', 'closed', 'wont_fix']
        ]
        
        # 結果を変数に保存
        print(f"総バグ数: {total}")
        print(f"Critical/High（未解決）: {len(critical_high_open)}")
      
      store_as: "bug_analysis"
    
    # ========================================
    # Phase 2: Markdownレポート生成
    # ========================================
    - name: "generate_markdown_report"
      action: "create_markdown_file"
      path: "Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/bug_tracking/reports/bug_summary_{{today}}.md"
      content: |
        # QAバグサマリーレポート
        
        **生成日時**: {{now}}  
        **対象プロジェクト**: cursor_to_notion  
        **バージョン**: v1.0  
        **QA開始日**: {{metadata.qa_start_date}}
        
        ---
        
        ## 📊 統計サマリー
        
        ### 全体統計
        - **総バグ数**: {{bug_analysis.total}}
        - **Open**: {{bug_analysis.by_status.new + bug_analysis.by_status.open + bug_analysis.by_status.in_progress}}
        - **Fixed**: {{bug_analysis.by_status.fixed}}
        - **Verified**: {{bug_analysis.by_status.verified}}
        - **Closed**: {{bug_analysis.by_status.closed}}
        - **Won't Fix**: {{bug_analysis.by_status.wont_fix}}
        
        ### ステータス内訳
        | Status | Count |
        |--------|-------|
        {{#each bug_analysis.by_status}}
        | {{@key}} | {{this}} |
        {{/each}}
        
        ### 深刻度別
        | Severity | Count |
        |----------|-------|
        {{#each bug_analysis.by_severity}}
        | {{@key}} | {{this}} |
        {{/each}}
        
        ### タイプ別
        | Type | Count |
        |------|-------|
        {{#each bug_analysis.by_type}}
        | {{@key}} | {{this}} |
        {{/each}}
        
        ### フェーズ別発見数
        | Phase | Count |
        |-------|-------|
        {{#each bug_analysis.by_phase}}
        | {{@key}} | {{this}} |
        {{/each}}
        
        ---
        
        ## 🔴 Critical/High バグ（未解決）
        
        {{#if bug_analysis.critical_high_open}}
        | ID | Title | Severity | Status | Assignee | Found In |
        |-------|--------|----------|--------|----------|----------|
        {{#each bug_analysis.critical_high_open}}
        | {{this.id}} | {{this.title}} | {{this.severity}} | {{this.status}} | {{this.assignee || 'Unassigned'}} | {{this.found_in_test}} |
        {{/each}}
        {{else}}
        ✅ **Critical/Highバグはありません**
        {{/if}}
        
        ---
        
        ## 📋 全バグ一覧
        
        | ID | Type | Severity | Priority | Status | Title | Found In |
        |---|---|---|---|---|---|---|
        {{#each bugs_data.bugs}}
        | {{this.id}} | {{this.type}} | {{this.severity}} | {{this.priority}} | {{this.status}} | {{this.title}} | {{this.found_in_test}} |
        {{/each}}
        
        ---
        
        ## 🎯 リリース判定
        
        {{#if bug_analysis.critical_high_open}}
        ❌ **リリース不可**: Critical/High問題が未解決
        
        **ブロッカー**:
        {{#each bug_analysis.critical_high_open}}
        - {{this.id}}: {{this.title}}
        {{/each}}
        {{else}}
        ✅ **リリース可能**: Critical/High問題なし
        {{/if}}
        
        ---
        
        **生成日時**: {{now}}
    
    # ========================================
    # Phase 3: HTMLダッシュボード生成（オプション）
    # ========================================
    - name: "generate_html_dashboard"
      action: "create_file"
      path: "Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/bug_tracking/reports/dashboard.html"
      content: |
        <!DOCTYPE html>
        <html lang="ja">
        <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <title>QA Dashboard - cursor_to_notion</title>
            <style>
                * { margin: 0; padding: 0; box-sizing: border-box; }
                body { 
                    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
                    padding: 20px;
                    background: #f5f5f5;
                }
                h1 { margin-bottom: 30px; color: #333; }
                .metrics { display: flex; gap: 20px; margin-bottom: 30px; flex-wrap: wrap; }
                .metric {
                    background: white;
                    padding: 20px;
                    border-radius: 8px;
                    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
                    min-width: 150px;
                }
                .metric h3 { font-size: 14px; color: #666; margin-bottom: 10px; }
                .metric h2 { font-size: 32px; color: #333; }
                .critical { background: #ff4444; color: white; }
                .critical h3 { color: #ffe0e0; }
                .critical h2 { color: white; }
                .high { background: #ff8800; color: white; }
                .high h3 { color: #ffe8cc; }
                .high h2 { color: white; }
                table {
                    width: 100%;
                    background: white;
                    border-radius: 8px;
                    overflow: hidden;
                    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
                }
                th, td {
                    padding: 12px;
                    text-align: left;
                    border-bottom: 1px solid #eee;
                }
                th {
                    background: #f8f8f8;
                    font-weight: 600;
                    color: #333;
                }
                .status-new { color: #666; }
                .status-open { color: #0066cc; }
                .status-in-progress { color: #ff8800; }
                .status-fixed { color: #00aa00; }
                .status-verified { color: #006600; }
                .status-closed { color: #999; }
            </style>
        </head>
        <body>
            <h1>🐛 QA Dashboard - cursor_to_notion</h1>
            
            <div class="metrics">
                <div class="metric">
                    <h3>Total Bugs</h3>
                    <h2>{{bug_analysis.total}}</h2>
                </div>
                <div class="metric">
                    <h3>Open</h3>
                    <h2>{{bug_analysis.by_status.new + bug_analysis.by_status.open + bug_analysis.by_status.in_progress}}</h2>
                </div>
                <div class="metric critical">
                    <h3>Critical</h3>
                    <h2>{{bug_analysis.by_severity.critical || 0}}</h2>
                </div>
                <div class="metric high">
                    <h3>High</h3>
                    <h2>{{bug_analysis.by_severity.high || 0}}</h2>
                </div>
                <div class="metric">
                    <h3>Fixed</h3>
                    <h2>{{bug_analysis.by_status.fixed}}</h2>
                </div>
                <div class="metric">
                    <h3>Verified</h3>
                    <h2>{{bug_analysis.by_status.verified}}</h2>
                </div>
            </div>
            
            <h2>📋 全バグ一覧</h2>
            <table>
                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Title</th>
                        <th>Type</th>
                        <th>Severity</th>
                        <th>Status</th>
                        <th>Assignee</th>
                    </tr>
                </thead>
                <tbody>
                    {{#each bugs_data.bugs}}
                    <tr>
                        <td><strong>{{this.id}}</strong></td>
                        <td>{{this.title}}</td>
                        <td>{{this.type}}</td>
                        <td>{{this.severity}}</td>
                        <td class="status-{{this.status}}">{{this.status}}</td>
                        <td>{{this.assignee || 'Unassigned'}}</td>
                    </tr>
                    {{/each}}
                </tbody>
            </table>
            
            <p style="margin-top: 30px; color: #666; font-size: 14px;">
                生成日時: {{now}}
            </p>
        </body>
        </html>
    
    # ========================================
    # Phase 4: 完了通知・レポート表示
    # ========================================
    - name: "notify_and_display"
      action: "notify"
      message: |
        ✅ バグレポート生成完了
        
        📊 統計サマリー:
        - 総バグ数: {{bug_analysis.total}}
        - Open: {{bug_analysis.open_count}}
        - Critical/High（未解決）: {{bug_analysis.critical_high_open.length}}
        
        📁 生成されたファイル:
        - Markdown: bug_tracking/reports/bug_summary_{{today}}.md
        - HTML: bug_tracking/reports/dashboard.html
        
        {{#if bug_analysis.critical_high_open}}
        ⚠️  **リリースブロッカー**:
        {{#each bug_analysis.critical_high_open}}
        - {{this.id}}: {{this.title}}
        {{/each}}
        {{else}}
        ✅ リリースブロッカーなし
        {{/if}}
        
        📝 次のステップ:
        - レポートを確認: open bug_tracking/reports/bug_summary_{{today}}.md
        - ダッシュボード確認: open bug_tracking/reports/dashboard.html
        - リリース判定: `/aipm/aipm_4_15_delivery_リリース判定`
```

## 成果物
- `bug_tracking/reports/bug_summary_YYYYMMDD.md` - 日次サマリー
- `bug_tracking/reports/dashboard.html` - HTMLダッシュボード
- コンソール出力: 統計サマリー・リリース判定

## 次のステップ
- レポートを確認
- Critical/Highバグがある場合: トリアージ会議
- バグゼロの場合: `/aipm/aipm_4_15_delivery_リリース判定` に進む






















