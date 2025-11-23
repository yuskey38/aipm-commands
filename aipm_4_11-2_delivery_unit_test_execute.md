# 11-2 Delivery Unit Test Execute
name: ユニットテスト実施（pytest実行・カバレッジ測定）
---

# 11-2 : ユニットテスト実施（AIPMハッカソン）- pytest実行とカバレッジレポート生成

## 前提
- `11_QA実行` でユニットテストマスタープラン・テストコードが生成済み
- pytest、pytest-cov がインストール済み
- テストコードが `Flow/{{today}}/{{flow_dir}}/11_QA実行/unit_tests/` に配置済み
- **カバレッジ目標: 80%以上**

## 目的
- pytestを実行してユニットテストを実施
- カバレッジレポートを生成（HTML・JSON・ターミナル表示）
- 失敗したテストを特定し、修正ループを実行
- テスト結果サマリーを自動生成

## 実行手順
```yaml
- trigger: "(ユニットテスト実施|単体テスト実行|pytest実行|Run Unit Tests)"
  priority: high
  steps:
    # ========================================
    # Phase 0: 環境確認・準備
    # ========================================
    - name: "check_test_environment"
      action: "execute_shell"
      command: |
        python --version && \
        pytest --version && \
        pytest --cov --version
      message: "テスト環境を確認しています..."
      store_as: "env_check"
    
    - name: "display_environment_status"
      action: "display"
      content: |
        🔧 テスト環境
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        {{env_check}}
        
        **テスト対象ディレクトリ**:
        `Flow/{{today}}/{{flow_dir}}/11_QA実行/unit_tests/`
        
        **カバレッジ目標**: 80%以上
        **レポート出力先**: `Flow/{{today}}/{{flow_dir}}/11_QA実行/test_reports/`
    
    - name: "load_test_plan"
      action: "analyze"
      data: [
        "{{read_files(find_files(patterns=['**/11_QA実行/unit_test_master_plan.md']))}}",
        "{{list_files('Flow/{{today}}/{{flow_dir}}/11_QA実行/unit_tests/', '*.py')}}"
      ]
      instructions: |
        ユニットテストマスタープランから以下を抽出：
        1. テストファイル一覧（test_*.py）
        2. 総テスト数の推定
        3. 実行順序（依存関係考慮）
        4. カバレッジ目標
        5. モック戦略
        
        テストファイルの実在も確認し、存在しないファイルがあれば警告。
      store_as: "test_plan"
    
    - name: "display_test_overview"
      action: "display"
      content: |
        📊 ユニットテスト実行計画
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        
        **テストファイル数**: {{test_plan.test_files_count}}件
        **推定テストケース数**: {{test_plan.estimated_tests}}件
        **カバレッジ目標**: {{test_plan.coverage_target}}%
        
        **実行順序**（依存関係考慮）:
        {{#each test_plan.execution_order}}
        {{@index}}. {{file}} - {{description}}
        {{/each}}
        
        **注意事項**:
        - Notion APIはモック化されます
        - 一時ファイルは tempfile を使用します
        - 環境変数は monkeypatch で設定します
    
    - name: "confirm_test_execution"
      action: "confirm"
      message: |
        pytestを実行してユニットテストを開始しますか？
        
        実行コマンド:
        pytest unit_tests/ -v --cov=../../dev --cov-report=html --cov-report=term --cov-report=json --junitxml=test_reports/junit.xml
    
    # ========================================
    # Phase 1: pytest実行（初回）
    # ========================================
    - name: "execute_pytest_initial"
      action: "execute_shell"
      command: |
        cd "Flow/{{today}}/{{flow_dir}}/11_QA実行" && \
        mkdir -p test_reports && \
        pytest unit_tests/ -v \
          --cov=../../dev \
          --cov-report=html:test_reports/coverage_html \
          --cov-report=term-missing \
          --cov-report=json:test_reports/coverage.json \
          --junitxml=test_reports/junit.xml \
          --tb=short 2>&1 | tee test_reports/pytest_output.log
      message: "pytest実行中（初回）..."
      timeout: 300000
      store_as: "pytest_result_initial"
      on_error: "continue"
    
    - name: "parse_pytest_results"
      action: "analyze"
      data: [
        "{{pytest_result_initial}}",
        "{{read_files(find_files(patterns=['**/test_reports/coverage.json']))}}"
      ]
      instructions: |
        pytest実行結果を解析し、以下を抽出：
        1. 総テスト数、成功数、失敗数、スキップ数
        2. 実行時間
        3. カバレッジ率（全体・モジュール別）
        4. 失敗したテストの詳細（テスト名・エラーメッセージ・ファイル名・行番号）
        5. カバレッジ目標（80%）に対する達成状況
        6. カバレッジが不足している箇所（未テスト関数・未到達行）
        
        JSON形式で返す: {"summary": {...}, "failures": [...], "coverage": {...}, "missing_coverage": [...]}
      store_as: "test_results"
    
    - name: "display_initial_results"
      action: "display"
      content: |
        📈 pytest実行結果（初回）
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        
        **サマリー**:
        - 総テスト数: {{test_results.summary.total_tests}}件
        - ✅ 成功: {{test_results.summary.passed}}件
        - ❌ 失敗: {{test_results.summary.failed}}件
        - ⏭️ スキップ: {{test_results.summary.skipped}}件
        - ⏱️ 実行時間: {{test_results.summary.duration}}秒
        
        **カバレッジ**:
        - 全体: {{test_results.coverage.total}}% {{#if test_results.coverage.meets_goal}}✅ 目標達成{{else}}⚠️ 目標未達（80%）{{/if}}
        
        **モジュール別カバレッジ**:
        {{#each test_results.coverage.by_module}}
        - {{module_name}}: {{coverage}}%
        {{/each}}
        
        {{#if test_results.failures}}
        **失敗したテスト**:
        {{#each test_results.failures}}
        - {{test_name}} ({{file}}:{{line}})
          エラー: {{error_message}}
        {{/each}}
        {{/if}}
        
        **カバレッジレポート**:
        - HTML: `test_reports/coverage_html/index.html` をブラウザで開いて詳細確認
        - JSON: `test_reports/coverage.json`
        - JUnit XML: `test_reports/junit.xml`
    
    # ========================================
    # Phase 2: 失敗テスト修正ループ
    # ========================================
    - name: "check_if_fixes_needed"
      action: "conditional"
      condition: "{{test_results.summary.failed > 0}}"
      if_true: "proceed_to_fix_loop"
      if_false: "skip_to_coverage_check"
    
    - name: "proceed_to_fix_loop"
      action: "display"
      content: |
        🔧 失敗テスト修正ループ
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        
        {{test_results.summary.failed}}件の失敗テストを修正します。
    
    - name: "analyze_failures_and_suggest_fixes"
      action: "analyze"
      data: [
        "{{test_results.failures}}",
        "{{read_files(find_files(patterns=['**/unit_tests/*.py']))}}"
      ]
      instructions: |
        失敗したテスト各件について、以下を分析：
        1. 失敗理由の特定（アサーション失敗・例外発生・インポートエラーなど）
        2. 原因の推定（テストコードの誤り・テスト対象コードの不具合・モック設定ミスなど）
        3. 修正案の提示（テストコード修正・本体コード修正・モック追加など）
        4. 優先順位（クリティカル・高・中・低）
        
        修正可能なものは具体的なコード差分を提示。
        JSON形式で返す: {"fixes": [{"test_name": "...", "reason": "...", "fix_type": "...", "priority": "...", "code_diff": "..."}, ...]}
      store_as: "fix_suggestions"
    
    - name: "display_fix_suggestions"
      action: "display"
      content: |
        💡 修正提案
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        
        {{#each fix_suggestions.fixes}}
        ### {{@index}}. {{test_name}} (優先度: {{priority}})
        
        **失敗理由**: {{reason}}
        **修正タイプ**: {{fix_type}}
        
        **修正案**:
        ```{{code_language}}
        {{code_diff}}
        ```
        
        ---
        {{/each}}
    
    - name: "confirm_apply_fixes"
      action: "confirm"
      message: |
        上記の修正案を適用しますか？
        
        修正対象: {{fix_suggestions.fixes.length}}件
        - テストコード修正: {{fix_suggestions.test_code_fixes}}件
        - 本体コード修正: {{fix_suggestions.source_code_fixes}}件
        - モック追加: {{fix_suggestions.mock_additions}}件
    
    - name: "apply_fixes"
      action: "apply_code_changes"
      changes: "{{fix_suggestions.fixes}}"
      message: "修正を適用しています..."
    
    - name: "record_fixes"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11_QA実行/test_reports/fix_log_{{timestamp}}.md"
      content: |
        # テスト修正ログ {{timestamp}}
        
        ## 修正内容
        {{#each fix_suggestions.fixes}}
        ### {{test_name}}
        - **失敗理由**: {{reason}}
        - **修正タイプ**: {{fix_type}}
        - **優先度**: {{priority}}
        - **修正ファイル**: {{file}}
        
        **適用した修正**:
        ```{{code_language}}
        {{code_diff}}
        ```
        
        ---
        {{/each}}
    
    - name: "rerun_pytest_after_fixes"
      action: "execute_shell"
      command: |
        cd "Flow/{{today}}/{{flow_dir}}/11_QA実行" && \
        pytest unit_tests/ -v \
          --cov=../../dev \
          --cov-report=html:test_reports/coverage_html \
          --cov-report=term-missing \
          --cov-report=json:test_reports/coverage.json \
          --junitxml=test_reports/junit.xml \
          --tb=short 2>&1 | tee test_reports/pytest_output_after_fix.log
      message: "pytest再実行中（修正後）..."
      timeout: 300000
      store_as: "pytest_result_after_fix"
      on_error: "continue"
    
    - name: "parse_rerun_results"
      action: "analyze"
      data: [
        "{{pytest_result_after_fix}}",
        "{{read_files(find_files(patterns=['**/test_reports/coverage.json']))}}"
      ]
      instructions: |
        修正後のpytest実行結果を解析し、初回結果と比較：
        1. 改善された失敗テスト数
        2. 残存する失敗テスト
        3. カバレッジの変化
        4. 新たに発生した問題（リグレッション）
        
        JSON形式で返す: {"summary": {...}, "comparison": {...}, "remaining_failures": [...]}
      store_as: "rerun_results"
    
    - name: "display_rerun_comparison"
      action: "display"
      content: |
        📊 修正後の結果比較
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        
        **改善状況**:
        - 初回失敗: {{test_results.summary.failed}}件
        - 修正後失敗: {{rerun_results.summary.failed}}件
        - 改善: {{rerun_results.comparison.fixed_tests}}件 ✅
        
        {{#if rerun_results.remaining_failures}}
        **残存する失敗**:
        {{#each rerun_results.remaining_failures}}
        - {{test_name}}: {{error_message}}
        {{/each}}
        {{/if}}
        
        **カバレッジ変化**:
        - 初回: {{test_results.coverage.total}}%
        - 修正後: {{rerun_results.coverage.total}}% ({{rerun_results.comparison.coverage_change}}%)
    
    - name: "check_if_more_fixes_needed"
      action: "conditional"
      condition: "{{rerun_results.summary.failed > 0}}"
      if_true: "ask_continue_fix_loop"
      if_false: "proceed_to_coverage_check"
    
    - name: "ask_continue_fix_loop"
      action: "confirm"
      message: |
        まだ {{rerun_results.summary.failed}}件の失敗テストが残っています。
        
        修正ループを続けますか？（最大3回まで）
      store_as: "continue_fix_loop"
    
    - name: "repeat_fix_loop"
      action: "conditional"
      condition: "{{continue_fix_loop && fix_loop_count < 3}}"
      if_true: "analyze_failures_and_suggest_fixes"
      if_false: "proceed_to_coverage_check"
    
    # ========================================
    # Phase 3: カバレッジ不足箇所の補完
    # ========================================
    - name: "skip_to_coverage_check"
      action: "display"
      content: "全テストが成功しました。カバレッジ確認に進みます。"
    
    - name: "proceed_to_coverage_check"
      action: "analyze"
      data: [
        "{{read_files(find_files(patterns=['**/test_reports/coverage.json']))}}",
        "{{read_files(find_files(patterns=['**/unit_test_master_plan.md']))}}"
      ]
      instructions: |
        カバレッジレポートを解析し、80%目標に対する不足箇所を特定：
        1. カバレッジが80%未満のモジュール
        2. 未テストの関数
        3. 到達していないコード行（分岐・例外処理など）
        4. 追加テストの優先順位
        5. 推奨されるテストケース
        
        JSON形式で返す: {"meets_goal": bool, "missing_coverage": [...], "recommended_tests": [...]}
      store_as: "coverage_analysis"
    
    - name: "display_coverage_analysis"
      action: "display"
      content: |
        📊 カバレッジ分析
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        
        **目標達成状況**: {{#if coverage_analysis.meets_goal}}✅ 達成（80%以上）{{else}}⚠️ 未達成（80%未満）{{/if}}
        
        {{#unless coverage_analysis.meets_goal}}
        **カバレッジ不足箇所**:
        {{#each coverage_analysis.missing_coverage}}
        - {{module_name}}: {{coverage}}% (目標: 80%)
          - 未テスト関数: {{untested_functions}}
          - 未到達行: {{uncovered_lines}}
        {{/each}}
        
        **追加推奨テスト**:
        {{#each coverage_analysis.recommended_tests}}
        {{@index}}. **{{test_name}}** (優先度: {{priority}})
           - 対象関数: {{target_function}}
           - テストケース: {{test_cases}}
           - 期待カバレッジ向上: +{{coverage_increase}}%
        {{/each}}
        {{/unless}}
    
    - name: "ask_add_coverage_tests"
      action: "conditional"
      condition: "{{!coverage_analysis.meets_goal}}"
      if_true: "confirm_add_coverage_tests"
      if_false: "skip_coverage_tests"
    
    - name: "confirm_add_coverage_tests"
      action: "confirm"
      message: |
        カバレッジ目標未達成です。追加テストを生成しますか？
        
        追加推奨: {{coverage_analysis.recommended_tests.length}}件
        期待カバレッジ向上: +{{coverage_analysis.total_coverage_increase}}%
      store_as: "add_coverage_tests"
    
    - name: "generate_additional_tests"
      action: "analyze"
      data: [
        "{{coverage_analysis.recommended_tests}}",
        "{{read_files(find_files(patterns=['**/unit_tests/*.py']))}}"
      ]
      instructions: |
        推奨された追加テストの実装コードを生成：
        1. 既存テストファイルへの追記または新規ファイル作成
        2. エッジケース・異常系の網羅
        3. モック戦略の継承
        4. 既存テストとの整合性
        
        JSON形式で返す: {"test_files": [{"path": "...", "content": "...", "description": "..."}, ...]}
      store_as: "additional_tests"
    
    - name: "create_additional_test_files"
      action: "create_or_update_files"
      files: "{{additional_tests.test_files}}"
      base_path: "Flow/{{today}}/{{flow_dir}}/11_QA実行/unit_tests/"
      message: "追加テストファイルを作成しています..."
    
    - name: "rerun_pytest_with_additional_tests"
      action: "execute_shell"
      command: |
        cd "Flow/{{today}}/{{flow_dir}}/11_QA実行" && \
        pytest unit_tests/ -v \
          --cov=../../dev \
          --cov-report=html:test_reports/coverage_html \
          --cov-report=term-missing \
          --cov-report=json:test_reports/coverage.json \
          --junitxml=test_reports/junit.xml \
          --tb=short 2>&1 | tee test_reports/pytest_output_final.log
      message: "pytest最終実行中（追加テスト含む）..."
      timeout: 300000
      store_as: "pytest_result_final"
    
    - name: "parse_final_results"
      action: "analyze"
      data: [
        "{{pytest_result_final}}",
        "{{read_files(find_files(patterns=['**/test_reports/coverage.json']))}}"
      ]
      instructions: |
        最終pytest実行結果を解析：
        1. 総テスト数（追加後）
        2. 全テストの成功/失敗状況
        3. 最終カバレッジ率
        4. 目標（80%）達成状況
        
        JSON形式で返す: {"summary": {...}, "coverage": {...}, "goal_achieved": bool}
      store_as: "final_results"
    
    - name: "skip_coverage_tests"
      action: "display"
      content: "カバレッジ目標達成済みです。最終レポート生成に進みます。"
    
    # ========================================
    # Phase 4: テスト結果サマリー生成
    # ========================================
    - name: "generate_test_summary_report"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11_QA実行/test_reports/unit_test_summary.md"
      content: |
        # ユニットテスト実施サマリー
        
        **実行日時**: {{timestamp}}
        **プロジェクト**: {{project_name}}
        **カバレッジ目標**: 80%
        
        ## 最終結果
        
        ### テスト実行サマリー
        - **総テスト数**: {{final_results.summary.total_tests}}件
        - **✅ 成功**: {{final_results.summary.passed}}件
        - **❌ 失敗**: {{final_results.summary.failed}}件
        - **⏭️ スキップ**: {{final_results.summary.skipped}}件
        - **⏱️ 実行時間**: {{final_results.summary.duration}}秒
        
        ### カバレッジ結果
        - **全体カバレッジ**: {{final_results.coverage.total}}%
        - **目標達成**: {{#if final_results.goal_achieved}}✅ 達成（80%以上）{{else}}⚠️ 未達成{{/if}}
        
        **モジュール別カバレッジ**:
        {{#each final_results.coverage.by_module}}
        | {{module_name}} | {{coverage}}% | {{#if meets_goal}}✅{{else}}⚠️{{/if}} |
        {{/each}}
        
        ## 実行履歴
        
        ### 初回実行
        - テスト数: {{test_results.summary.total_tests}}件
        - 失敗: {{test_results.summary.failed}}件
        - カバレッジ: {{test_results.coverage.total}}%
        
        {{#if fix_suggestions}}
        ### 修正ループ（{{fix_loop_count}}回）
        {{#each fix_logs}}
        #### ループ{{@index}}
        - 修正件数: {{fixes_applied}}件
        - 改善テスト数: {{fixed_tests}}件
        - カバレッジ変化: {{coverage_before}}% → {{coverage_after}}%
        {{/each}}
        {{/if}}
        
        {{#if additional_tests}}
        ### カバレッジ補完
        - 追加テスト: {{additional_tests.test_files.length}}件
        - カバレッジ向上: +{{coverage_increase}}%
        {{/if}}
        
        ## 成功基準
        - [{{#if final_results.summary.failed == 0}}x{{else}} {{/if}}] 全テストが成功
        - [{{#if final_results.goal_achieved}}x{{else}} {{/if}}] カバレッジ80%以上達成
        - [{{#if final_results.summary.duration < 300}}x{{else}} {{/if}}] 実行時間5分以内
        
        ## 詳細レポート
        - **HTMLカバレッジ**: `test_reports/coverage_html/index.html`
        - **JSONカバレッジ**: `test_reports/coverage.json`
        - **JUnit XML**: `test_reports/junit.xml`
        - **pytest出力ログ**: `test_reports/pytest_output*.log`
        
        {{#if fix_suggestions}}
        ## 修正ログ
        {{#each fix_logs}}
        - `test_reports/fix_log_{{timestamp}}.md`
        {{/each}}
        {{/if}}
        
        ## 推奨アクション
        {{#if final_results.summary.failed > 0}}
        ⚠️ **失敗テストが残っています**:
        {{#each final_results.failures}}
        - {{test_name}}: {{error_message}}
        {{/each}}
        
        次のステップ:
        1. 失敗テストの原因を詳細調査
        2. 本体コードまたはテストコードを修正
        3. pytest再実行
        {{/if}}
        
        {{#unless final_results.goal_achieved}}
        ⚠️ **カバレッジ目標未達成**:
        - 現在: {{final_results.coverage.total}}%
        - 目標: 80%
        - 不足: {{80 - final_results.coverage.total}}%
        
        次のステップ:
        1. HTMLカバレッジレポートで未到達箇所を確認
        2. 優先度の高い関数・分岐のテスト追加
        3. pytest再実行
        {{/unless}}
        
        {{#if final_results.goal_achieved && final_results.summary.failed == 0}}
        ✅ **全基準達成**: 統合テストに進むことができます
        
        次のステップ:
        1. `11_QA実行` で統合テストを実施
        2. パフォーマンステストを実施
        3. QA総合レポートを生成
        {{/if}}
        
        ## 参照
        - ユニットテストマスタープラン: `../unit_test_master_plan.md`
        - トータル開発仕様書: `../../07_開発タスク分解/total_development_spec.md`
    
    - name: "display_completion_summary"
      action: "display"
      content: |
        ✅ ユニットテスト実施完了
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        
        **最終結果**:
        - 総テスト数: {{final_results.summary.total_tests}}件
        - 成功率: {{final_results.summary.success_rate}}%
        - カバレッジ: {{final_results.coverage.total}}% {{#if final_results.goal_achieved}}✅{{else}}⚠️{{/if}}
        
        **生成ファイル**:
        - サマリーレポート: `test_reports/unit_test_summary.md`
        - HTMLカバレッジ: `test_reports/coverage_html/index.html`
        - JUnit XML: `test_reports/junit.xml`
        {{#if fix_suggestions}}
        - 修正ログ: `test_reports/fix_log_*.md`
        {{/if}}
        
        **次のアクション**:
        {{#if final_results.goal_achieved && final_results.summary.failed == 0}}
        ✅ 全基準達成 → `11_QA実行` で統合テストに進んでください
        {{else}}
        ⚠️ 基準未達成 → 上記の推奨アクションを実施してください
        {{/if}}
        
        **カバレッジレポート確認**:
        ブラウザで `test_reports/coverage_html/index.html` を開いて詳細確認できます。
```

## 次のコマンド
→ `11_QA実行` で統合テスト・パフォーマンステストに進む
→ `15_リリース判定` でリリース可否を総合評価

## 変更点（v1.0 - ユニットテスト実施）
- **pytest自動実行**: カバレッジ測定付きでpytestを自動実行
- **修正ループ**: 失敗テストを自動解析・修正提案・再実行（最大3回）
- **カバレッジ補完**: 80%未達の場合、追加テスト自動生成
- **詳細レポート**: HTML・JSON・JUnit XML・ターミナル出力の統合
- **段階的実行**: 初回実行 → 修正ループ → カバレッジ補完 → 最終実行
- **成功基準チェック**: 全テスト成功・カバレッジ80%・実行時間5分の自動判定
