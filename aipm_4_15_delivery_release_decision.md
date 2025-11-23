# 15 Delivery Release Decision
name: リリース判定（最終品質評価・リリース可否決定）
---

# 15 : リリース判定 - 最終品質評価とリリース可否決定

## 前提
- QA実行（11_QA実行）完了
- バグトラッキングシステム稼働中
- 全テスト結果が記録済み
- バグレポート生成（14_バグレポート生成）実行済み

## 目的
- テスト結果・バグ状況を総合評価
- リリース基準に照らして合否判定
- リリースノート生成
- リリースブロッカーの特定と対応計画

## 実行手順
```yaml
- trigger: "(リリース判定|Release Decision|品質評価|リリース可否)"
  priority: critical
  steps:
    # ========================================
    # Phase 1: 全QAデータ収集
    # ========================================
    - name: "collect_qa_data"
      action: "read_files"
      files:
        - "Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/QA_COMPLETION_REPORT.md"
        - "Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/FINAL_QA_REPORT.md"
        - "Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/bug_tracking/bugs.yaml"
        - "Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/bug_tracking/reports/bug_summary_{{today}}.md"
      store_as: "qa_data"
    
    # ========================================
    # Phase 2: リリース基準評価
    # ========================================
    - name: "evaluate_release_criteria"
      action: "execute_python"
      script: |
        import yaml
        from pathlib import Path
        
        bugs_yaml = Path("Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/bug_tracking/bugs.yaml")
        
        with open(bugs_yaml, 'r') as f:
            data = yaml.safe_load(f)
        
        bugs = data.get('bugs', [])
        
        # リリース基準チェック
        criteria = {
            'critical_bugs': [],
            'high_bugs': [],
            'blocking_bugs': [],
            'open_p0_bugs': []
        }
        
        for bug in bugs:
            # Critical/High未解決バグ
            if bug.get('severity') == 'critical' and bug.get('status') not in ['verified', 'closed', 'wont_fix']:
                criteria['critical_bugs'].append(bug)
            
            if bug.get('severity') == 'high' and bug.get('status') not in ['verified', 'closed', 'wont_fix']:
                criteria['high_bugs'].append(bug)
            
            # P0優先度バグ
            if bug.get('priority') == 'P0' and bug.get('status') not in ['verified', 'closed', 'wont_fix']:
                criteria['open_p0_bugs'].append(bug)
        
        # リリース判定
        release_decision = {
            'can_release': len(criteria['critical_bugs']) == 0 and len(criteria['open_p0_bugs']) == 0,
            'mvp_release': len(criteria['critical_bugs']) == 0,
            'ga_release': len(criteria['critical_bugs']) == 0 and len(criteria['high_bugs']) == 0,
            'blockers': criteria['critical_bugs'] + criteria['open_p0_bugs'],
            'warnings': criteria['high_bugs']
        }
        
        print(f"Critical未解決: {len(criteria['critical_bugs'])}")
        print(f"High未解決: {len(criteria['high_bugs'])}")
        print(f"P0未解決: {len(criteria['open_p0_bugs'])}")
        print(f"リリース可能: {release_decision['can_release']}")
      
      store_as: "release_evaluation"
    
    # ========================================
    # Phase 3: テストカバレッジ・成功率評価
    # ========================================
    - name: "evaluate_test_coverage"
      action: "analyze"
      data: "{{qa_data.QA_COMPLETION_REPORT}}"
      instructions: |
        QA完了レポートから以下を抽出:
        1. ユニットテスト成功率
        2. 統合テスト準備完了率
        3. カバレッジ達成率
        4. 必須条件達成状況（M1-M6）
        5. 推奨条件達成状況（R1-R4）
        
        各指標が目標値を達成しているか評価。
      store_as: "test_metrics"
    
    # ========================================
    # Phase 4: リリース判定レポート生成
    # ========================================
    - name: "generate_release_decision_report"
      action: "create_markdown_file"
      path: "Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/RELEASE_DECISION_REPORT.md"
      content: |
        # リリース判定レポート - cursor_to_notion v1.0
        
        **判定日時**: {{now}}  
        **判定者**: QA Team  
        **対象バージョン**: v1.0 MVP
        
        ---
        
        ## 🎯 リリース判定結果
        
        {{#if release_evaluation.can_release}}
        ### ✅ **リリース承認**
        
        cursor_to_notion v1.0 MVP は、以下の理由によりリリース可能と判定します：
        
        - ✅ Criticalバグ: 0件
        - ✅ P0ブロッカー: 0件
        - ✅ ユニットテスト成功率: {{test_metrics.unit_test_success_rate}}%
        - ✅ 必須条件達成: {{test_metrics.mandatory_criteria_met}}/6
        {{else}}
        ### ❌ **リリース不可**
        
        以下のブロッカーが未解決のため、リリースできません：
        
        {{#each release_evaluation.blockers}}
        - **{{this.id}}**: {{this.title}} ({{this.severity}}, {{this.priority}})
          - 状態: {{this.status}}
          - 発見: {{this.found_in_test}}
        {{/each}}
        {{/if}}
        
        ---
        
        ## 📊 品質指標サマリー
        
        ### ユニットテスト
        | 指標 | 実績 | 目標 | 判定 |
        |------|------|------|------|
        | テストケース数 | {{test_metrics.total_test_cases}} | 80+ | {{test_metrics.test_cases_pass}} |
        | 成功率 | {{test_metrics.success_rate}}% | 95%+ | {{test_metrics.success_rate_pass}} |
        | カバレッジ | {{test_metrics.coverage}}% | 80%+ | {{test_metrics.coverage_pass}} |
        
        ### バグ統計
        | カテゴリ | 件数 | 判定 |
        |----------|------|------|
        | Critical（未解決） | {{release_evaluation.blockers.length}} | {{#if release_evaluation.can_release}}✅{{else}}❌{{/if}} |
        | High（未解決） | {{release_evaluation.warnings.length}} | {{#if release_evaluation.ga_release}}✅{{else}}⚠️{{/if}} |
        | 総バグ数 | {{qa_data.bugs.length}} | - |
        | 修正済み | {{test_metrics.fixed_bugs}} | - |
        
        ### 必須条件（M1-M6）
        {{#each test_metrics.mandatory_criteria}}
        - {{#if this.met}}[x]{{else}}[ ]{{/if}} **{{this.code}}**: {{this.title}} - {{this.actual}}
        {{/each}}
        
        ### 推奨条件（R1-R4）
        {{#each test_metrics.recommended_criteria}}
        - {{#if this.met}}[x]{{else}}[ ]{{/if}} **{{this.code}}**: {{this.title}} - {{this.actual}}
        {{/each}}
        
        ---
        
        ## 🐛 バグ状況詳細
        
        ### Critical/High バグ（未解決）
        {{#if release_evaluation.blockers.length > 0 or release_evaluation.warnings.length > 0}}
        {{#each release_evaluation.blockers}}
        #### 🔴 {{this.id}}: {{this.title}}
        - **深刻度**: {{this.severity}}
        - **優先度**: {{this.priority}}
        - **状態**: {{this.status}}
        - **発見**: {{this.found_in_test}}
        - **影響**: {{this.affected_components}}
        {{/each}}
        
        {{#each release_evaluation.warnings}}
        #### 🟠 {{this.id}}: {{this.title}}
        - **深刻度**: {{this.severity}}
        - **優先度**: {{this.priority}}
        - **状態**: {{this.status}}
        - **発見**: {{this.found_in_test}}
        {{/each}}
        {{else}}
        ✅ **Critical/Highバグなし**
        {{/if}}
        
        ---
        
        ## 📋 リリース推奨形態
        
        ### v1.0 MVP（即座リリース）
        {{#if release_evaluation.mvp_release}}
        ✅ **推奨**: 初期アダプター向けMVPリリース
        
        **対象ユーザー**: 評価ユーザー、内部チーム  
        **リスク**: 低（Critical問題なし）  
        **制限事項**: 
        - Highバグ {{release_evaluation.warnings.length}}件が未解決
        - 統合テスト未実施（準備完了）
        {{else}}
        ❌ **非推奨**: Criticalバグが未解決
        
        **対応必要**:
        {{#each release_evaluation.blockers}}
        - {{this.id}}: {{this.title}}
        {{/each}}
        {{/if}}
        
        ### v1.0 GA（一般リリース）
        {{#if release_evaluation.ga_release}}
        ✅ **推奨**: 統合テスト1シナリオ実施後
        
        **対象ユーザー**: 一般ユーザー  
        **追加作業**: scenario1_new_project.md 実施（30分）  
        **リスク**: 極低
        {{else}}
        ⚠️  **条件付き**: Highバグ {{release_evaluation.warnings.length}}件の対応後
        
        **対応推奨**:
        {{#each release_evaluation.warnings}}
        - {{this.id}}: {{this.title}}
        {{/each}}
        {{/if}}
        
        ---
        
        ## 📝 リリースノート（自動生成）
        
        ### Fixed Bugs
        {{#each qa_data.fixed_bugs}}
        - **{{this.id}}**: {{this.title}}
          - 深刻度: {{this.severity}}
          - 修正バージョン: {{this.fix_version}}
        {{/each}}
        
        ### Improvements
        {{#each qa_data.improvements}}
        - **{{this.id}}**: {{this.title}}
        {{/each}}
        
        ### Known Issues
        {{#each release_evaluation.warnings}}
        - **{{this.id}}**: {{this.title}} ({{this.fix_version}}で修正予定)
        {{/each}}
        
        ---
        
        ## 🎯 アクションアイテム
        
        ### 即時実施（リリース前）
        {{#if release_evaluation.can_release}}
        - [ ] リリースノート最終確認
        - [ ] ドキュメント更新
        - [ ] バージョンタグ作成
        - [ ] リリースアナウンス準備
        {{else}}
        - [ ] ブロッカーバグ修正
        {{#each release_evaluation.blockers}}
        - [ ] {{this.id}}: {{this.title}}
        {{/each}}
        - [ ] 修正後に再QA実施
        {{/if}}
        
        ### 短期実施（次バージョン）
        - [ ] 統合テスト4シナリオ実施
        - [ ] Highバグ修正
        {{#each release_evaluation.warnings}}
        - [ ] {{this.id}}: {{this.title}}
        {{/each}}
        - [ ] パフォーマンステスト実施
        
        ---
        
        ## ✍️ 承認署名
        
        **QA責任者**: _______________  日付: _______  
        **開発リーダー**: _______________  日付: _______  
        **プロダクトオーナー**: _______________  日付: _______
        
        ---
        
        **判定日時**: {{now}}  
        **次回レビュー**: v1.1リリース前
    
    # ========================================
    # Phase 5: 完了通知・判定結果表示
    # ========================================
    - name: "notify_decision"
      action: "notify"
      message: |
        🎯 リリース判定完了
        
        {{#if release_evaluation.can_release}}
        ✅ **リリース承認**
        
        cursor_to_notion v1.0 MVP は、リリース可能です。
        
        📊 品質指標:
        - ユニットテスト: {{test_metrics.total_test_cases}}ケース、成功率{{test_metrics.success_rate}}%
        - バグ: Critical 0件、High {{release_evaluation.warnings.length}}件
        - 必須条件: {{test_metrics.mandatory_criteria_met}}/6 達成
        
        📝 次のステップ:
        1. リリースノート確認: open RELEASE_DECISION_REPORT.md
        2. バージョンタグ作成: git tag v1.0
        3. リリース実施
        {{else}}
        ❌ **リリース不可**
        
        以下のブロッカーが未解決です:
        {{#each release_evaluation.blockers}}
        - {{this.id}}: {{this.title}}
        {{/each}}
        
        📝 対応必要:
        1. ブロッカーバグ修正
        2. 修正後に再QA実施
        3. 再度リリース判定実施
        {{/if}}
        
        📄 詳細レポート: Stock/programs/Tools/projects/cursor_to_notion/documents/11_QA実行/RELEASE_DECISION_REPORT.md
```

## 成果物
- `RELEASE_DECISION_REPORT.md` - リリース判定レポート
- コンソール出力: リリース可否・アクションアイテム

## リリース基準
### MVP（最小限）
- ✅ Criticalバグ: 0件
- ✅ P0ブロッカー: 0件
- ✅ ユニットテスト成功率: 95%以上

### GA（一般向け）
- ✅ Criticalバグ: 0件
- ✅ Highバグ: 0件
- ✅ 統合テスト1シナリオ実施
- ✅ ユニットテスト成功率: 95%以上
- ✅ 必須条件: 6/6達成

## 次のステップ
- リリース可の場合: バージョンタグ作成・リリース実施
- リリース不可の場合: ブロッカー修正 → 再QA → 再判定





















