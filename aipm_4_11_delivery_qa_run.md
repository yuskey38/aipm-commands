# 11 Delivery QA Run
name: QA実行（品質保証・テスト実行）
---

# 11 : QA実行（AIPMハッカソン）- 段階的品質保証プロセス

## 前提
- 直前に `07_開発タスク分解` が完了済み（dev_tasks.yaml, total_development_spec.md が存在）
- 各開発タスクの受け入れテストは完了済み
- **total_development_spec.md のテスト戦略をベース**に、包括的なQAを実施
- **HITL（Human-In-The-Loop）を意識**した段階的実行
- pytest（ユニットテスト）、chrome-mcp（統合テスト）を活用

## 目的
- 人間のQA担当者と同じような多角的な品質保証を実現
- ユニットテスト → 統合テスト → パフォーマンステスト → セキュリティテストの段階的実行
- 効率的かつ網羅的なテストカバレッジ
- テスト結果の可視化とレポート生成

## 実行手順
```yaml
- trigger: "(QA実行|QA開始|品質保証|Quality Assurance|テスト実行)"
  priority: high
  steps:
    # ========================================
    # Phase 0: 初期化・準備
    # ========================================
    - name: "load_test_strategy"
      action: "analyze"
      data: [
        "{{read_files(find_files(patterns=['**/total_development_spec.md']))}}",
        "{{read_files(find_files(patterns=['**/dev_tasks.yaml']))}}",
        "{{read_files(find_files(patterns=['**/dev_tasks_order.md']))}}",
        "{{read_files(find_files(patterns=['**/user_story_map.yaml']))}}"
      ]
      instructions: |
        total_development_spec.md のテスト戦略を解析し、以下を抽出：
        1. ユニットテスト要件（pytestカバレッジ目標、重要関数リスト）
        2. 統合テスト要件（シナリオ、エンドポイント）
        3. パフォーマンステスト要件（規模、測定項目）
        4. セキュリティ要件（認証、API保護）
        
        dev_tasks.yaml から全タスクの成果物ファイルとテスト観点を抽出。
        効率的なテスト順序を提案（依存関係考慮）。
      store_as: "test_strategy"
    
    - name: "create_qa_structure"
      action: "execute_shell"
      command: |
        mkdir -p "Flow/{{today}}/{{flow_dir}}/11_QA実行/unit_tests" && \
        mkdir -p "Flow/{{today}}/{{flow_dir}}/11_QA実行/integration_tests" && \
        mkdir -p "Flow/{{today}}/{{flow_dir}}/11_QA実行/performance_tests" && \
        mkdir -p "Flow/{{today}}/{{flow_dir}}/11_QA実行/test_reports" && \
        mkdir -p "Flow/{{today}}/{{flow_dir}}/11_QA実行/test_data"
      message: "QA実行用フォルダ構造を作成しました"
    
    - name: "display_test_overview"
      action: "display"
      content: |
        📊 QA実行計画概要
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        
        **テスト戦略サマリー**:
        {{test_strategy.summary}}
        
        **対象タスク**: {{test_strategy.total_tasks}}件
        **ユニットテスト対象関数**: {{test_strategy.unit_test_targets}}件
        **統合テストシナリオ**: {{test_strategy.integration_scenarios}}件
        **推定実行時間**: {{test_strategy.estimated_time}}
        
        **段階的実行プラン**:
        1. Phase 1: ユニットテスト（pytest）- 基盤関数から順次
        2. Phase 2: 統合テスト（chrome-mcp）- ユーザーシナリオベース
        3. Phase 3: パフォーマンステスト - 大規模データでの動作確認
        4. Phase 4: セキュリティテスト - 認証・API保護確認
        
        各Phase完了時に人間確認ポイントがあります。
    
    # ========================================
    # Phase 1: ユニットテスト設計・実行
    # ========================================
    - name: "confirm_unit_test_start"
      action: "confirm"
      message: |
        Phase 1: ユニットテスト を開始しますか？
        
        対象: {{test_strategy.unit_test_targets}}件の関数
        - 基盤関数（環境変数読み込み、ハッシュ計算など）
        - Markdown変換関数
        - Push/Pull関数
        - マージロジック
        
        pytestでカバレッジ80%以上を目標とします。
    
    - name: "generate_unit_test_plan"
      action: "analyze"
      data: ["{{test_strategy}}", "{{read_files(find_files(patterns=['**/dev_tasks.yaml']))}}"]
      instructions: |
        dev_tasks.yaml の全タスクから成果物ファイル・関数を抽出し、
        効率的なユニットテスト計画を作成：
        
        1. **テストグループ化**: 類似機能をまとめる（例: 環境変数系、Markdown変換系）
        2. **優先順位付け**: クリティカルパスの関数を優先
        3. **依存関係考慮**: 依存関数を先にテスト
        4. **モック戦略**: 外部API（Notion API）はモック化
        5. **エッジケース抽出**: 各関数の境界値・異常系テスト
        
        出力形式:
        - テストファイル名（test_xxx.py）
        - テスト対象関数
        - テストケース（正常系・異常系）
        - モック対象
        - 期待カバレッジ
      store_as: "unit_test_plan"
    
    - name: "create_unit_test_master_plan"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11_QA実行/unit_test_master_plan.md"
      content: |
        # ユニットテストマスタープラン
        
        ## 概要
        **目標カバレッジ**: 80%以上
        **実行環境**: Python 3.8+, pytest
        **総テスト数**: {{unit_test_plan.total_tests}}件
        **推定実行時間**: {{unit_test_plan.estimated_time}}
        
        ## テストグループ
        {{#each unit_test_plan.groups}}
        ### Group {{@index}}: {{name}}
        **優先度**: {{priority}}
        **対象ファイル**: {{target_files}}
        **テスト数**: {{test_count}}件
        
        #### テストケース
        {{#each test_cases}}
        - **{{function_name}}**
          - 正常系: {{normal_cases}}
          - 異常系: {{error_cases}}
          - エッジケース: {{edge_cases}}
          - モック: {{mocks}}
        {{/each}}
        {{/each}}
        
        ## 実行順序（依存関係考慮）
        {{#each unit_test_plan.execution_order}}
        {{@index}}. {{test_file}} - {{description}}
        {{/each}}
        
        ## モック戦略
        - **Notion API**: `unittest.mock` で全エンドポイントをモック
        - **ファイルシステム**: `tempfile` で一時ディレクトリ使用
        - **環境変数**: `monkeypatch` で動的設定
        
        ## カバレッジ測定
        ```bash
        pytest --cov=. --cov-report=html --cov-report=term
        ```
    
    - name: "generate_unit_tests_batch1"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11_QA実行/unit_tests/test_env_loader.py"
      content: |
        """
        ユニットテスト: 環境変数読み込み関数
        対象: c2n.py::_load_env_file(), _load_env_for_target()
        """
        import pytest
        import tempfile
        from pathlib import Path
        import os
        
        # テスト対象のインポート（パスは調整）
        import sys
        sys.path.insert(0, str(Path(__file__).parents[4] / 'dev'))
        from c2n import _load_env_file, _load_env_for_target
        
        
        class TestLoadEnvFile:
            """_load_env_file() のテスト"""
            
            def test_normal_case(self):
                """正常系: 標準的な.envファイルを読み込める"""
                with tempfile.NamedTemporaryFile(mode='w', delete=False, suffix='.env') as f:
                    f.write("NOTION_TOKEN=secret_xxx\n")
                    f.write("OTHER_VAR=value123\n")
                    env_path = Path(f.name)
                
                try:
                    env = _load_env_file(env_path)
                    assert env['NOTION_TOKEN'] == 'secret_xxx'
                    assert env['OTHER_VAR'] == 'value123'
                    assert len(env) == 2
                finally:
                    env_path.unlink()
            
            def test_with_comments(self):
                """正常系: コメント行を無視する"""
                with tempfile.NamedTemporaryFile(mode='w', delete=False, suffix='.env') as f:
                    f.write("# コメント\n")
                    f.write("NOTION_TOKEN=secret_xxx\n")
                    f.write("# 別のコメント\n")
                    env_path = Path(f.name)
                
                try:
                    env = _load_env_file(env_path)
                    assert '# コメント' not in env
                    assert 'NOTION_TOKEN' in env
                finally:
                    env_path.unlink()
            
            def test_empty_lines(self):
                """正常系: 空行を無視する"""
                with tempfile.NamedTemporaryFile(mode='w', delete=False, suffix='.env') as f:
                    f.write("\n")
                    f.write("NOTION_TOKEN=secret_xxx\n")
                    f.write("\n\n")
                    env_path = Path(f.name)
                
                try:
                    env = _load_env_file(env_path)
                    assert len(env) == 1
                finally:
                    env_path.unlink()
            
            def test_file_not_found(self):
                """異常系: ファイルが存在しない場合は空辞書を返す"""
                env = _load_env_file(Path('/nonexistent/.env'))
                assert env == {}
            
            def test_malformed_line(self):
                """エッジケース: =がない行は無視"""
                with tempfile.NamedTemporaryFile(mode='w', delete=False, suffix='.env') as f:
                    f.write("NOTION_TOKEN=secret_xxx\n")
                    f.write("MALFORMED_LINE\n")
                    env_path = Path(f.name)
                
                try:
                    env = _load_env_file(env_path)
                    assert 'NOTION_TOKEN' in env
                    assert 'MALFORMED_LINE' not in env
                finally:
                    env_path.unlink()
        
        
        class TestLoadEnvForTarget:
            """_load_env_for_target() のテスト"""
            
            def test_from_environment_variable(self, monkeypatch):
                """優先順位1: 環境変数から読み込む"""
                monkeypatch.setenv('NOTION_TOKEN', 'env_token_123')
                
                with tempfile.TemporaryDirectory() as tmpdir:
                    target_dir = Path(tmpdir)
                    token = _load_env_for_target(target_dir)
                    assert token == 'env_token_123'
            
            def test_from_c2n_env(self, monkeypatch):
                """優先順位2: target_dir/.c2n/.envから読み込む"""
                monkeypatch.delenv('NOTION_TOKEN', raising=False)
                
                with tempfile.TemporaryDirectory() as tmpdir:
                    target_dir = Path(tmpdir)
                    c2n_dir = target_dir / '.c2n'
                    c2n_dir.mkdir()
                    
                    c2n_env = c2n_dir / '.env'
                    c2n_env.write_text('NOTION_TOKEN=c2n_token_456')
                    
                    token = _load_env_for_target(target_dir)
                    assert token == 'c2n_token_456'
            
            def test_priority_order(self, monkeypatch):
                """優先順位: 環境変数 > .c2n/.env > プロジェクト/.env"""
                monkeypatch.setenv('NOTION_TOKEN', 'env_highest_priority')
                
                with tempfile.TemporaryDirectory() as tmpdir:
                    target_dir = Path(tmpdir)
                    c2n_dir = target_dir / '.c2n'
                    c2n_dir.mkdir()
                    
                    # 低優先度の設定
                    (target_dir / '.env').write_text('NOTION_TOKEN=project_low_priority')
                    (c2n_dir / '.env').write_text('NOTION_TOKEN=c2n_medium_priority')
                    
                    # 環境変数が最優先
                    token = _load_env_for_target(target_dir)
                    assert token == 'env_highest_priority'
            
            def test_token_not_found(self, monkeypatch):
                """異常系: トークンが見つからない場合はValueError"""
                monkeypatch.delenv('NOTION_TOKEN', raising=False)
                
                with tempfile.TemporaryDirectory() as tmpdir:
                    target_dir = Path(tmpdir)
                    
                    with pytest.raises(ValueError, match="NOTION_TOKEN not found"):
                        _load_env_for_target(target_dir)
        
        
        # pytest実行コマンド:
        # pytest test_env_loader.py -v --cov=c2n --cov-report=term
    
    - name: "checkpoint_unit_test_batch1"
      action: "display"
      content: |
        ✅ Batch 1のユニットテストファイルを生成しました
        
        **生成ファイル**:
        - test_env_loader.py（環境変数読み込み系）
        
        **次のアクション**:
        1. 生成されたテストファイルを確認
        2. 必要に応じて修正
        3. pytest実行: `cd dev && pytest unit_tests/test_env_loader.py -v`
        4. カバレッジ確認
        
        続けて次のBatchを生成しますか？
    
    - name: "confirm_continue_unit_tests"
      action: "confirm"
      message: "Batch 2（Markdown変換系）のユニットテストを生成しますか？"
    
    - name: "generate_unit_tests_batch2"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11_QA実行/unit_tests/test_markdown_converter.py"
      content: |
        """
        ユニットテスト: Markdown変換関数
        対象: md_to_blocks.py::convert_markdown_to_notion_blocks(), parse_inline_formatting()
        """
        import pytest
        import sys
        from pathlib import Path
        
        sys.path.insert(0, str(Path(__file__).parents[4] / 'dev'))
        from md_to_blocks import convert_markdown_to_notion_blocks, parse_inline_formatting
        
        
        class TestConvertMarkdownToNotionBlocks:
            """convert_markdown_to_notion_blocks() のテスト"""
            
            def test_heading_conversion(self):
                """正常系: 見出しの変換"""
                markdown = "# 見出し1\n## 見出し2\n### 見出し3"
                blocks = convert_markdown_to_notion_blocks(markdown)
                
                assert len(blocks) == 3
                assert blocks[0]['type'] == 'heading_1'
                assert blocks[0]['heading_1']['rich_text'][0]['text']['content'] == '見出し1'
                assert blocks[1]['type'] == 'heading_2'
                assert blocks[2]['type'] == 'heading_3'
            
            def test_list_conversion(self):
                """正常系: リストの変換"""
                markdown = "- 項目1\n- 項目2\n1. 番号1\n2. 番号2"
                blocks = convert_markdown_to_notion_blocks(markdown)
                
                assert blocks[0]['type'] == 'bulleted_list_item'
                assert blocks[1]['type'] == 'bulleted_list_item'
                assert blocks[2]['type'] == 'numbered_list_item'
                assert blocks[3]['type'] == 'numbered_list_item'
            
            def test_code_block_conversion(self):
                """正常系: コードブロックの変換"""
                markdown = "```python\nprint('Hello')\n```"
                blocks = convert_markdown_to_notion_blocks(markdown)
                
                assert blocks[0]['type'] == 'code'
                assert blocks[0]['code']['language'] == 'python'
                assert "print('Hello')" in blocks[0]['code']['rich_text'][0]['text']['content']
            
            def test_paragraph_conversion(self):
                """正常系: 通常段落の変換"""
                markdown = "これは段落です。"
                blocks = convert_markdown_to_notion_blocks(markdown)
                
                assert blocks[0]['type'] == 'paragraph'
                assert blocks[0]['paragraph']['rich_text'][0]['text']['content'] == 'これは段落です。'
            
            def test_empty_input(self):
                """エッジケース: 空文字列"""
                blocks = convert_markdown_to_notion_blocks("")
                assert blocks == []
            
            def test_mixed_content(self):
                """正常系: 複数の要素が混在"""
                markdown = """# タイトル
        
        段落です。
        
        - リスト1
        - リスト2
        
        ```python
        code
        ```
        """
                blocks = convert_markdown_to_notion_blocks(markdown)
                
                types = [b['type'] for b in blocks]
                assert 'heading_1' in types
                assert 'paragraph' in types
                assert 'bulleted_list_item' in types
                assert 'code' in types
        
        
        class TestParseInlineFormatting:
            """parse_inline_formatting() のテスト"""
            
            def test_bold_text(self):
                """正常系: 太字の解析"""
                text = "これは**太字**です"
                rich_text = parse_inline_formatting(text)
                
                assert len(rich_text) == 3
                assert rich_text[1]['annotations']['bold'] is True
                assert rich_text[1]['text']['content'] == '太字'
            
            def test_no_formatting(self):
                """正常系: フォーマットなしのテキスト"""
                text = "プレーンテキスト"
                rich_text = parse_inline_formatting(text)
                
                assert len(rich_text) == 1
                assert rich_text[0]['text']['content'] == 'プレーンテキスト'
                assert 'annotations' not in rich_text[0] or not rich_text[0].get('annotations', {}).get('bold')
            
            def test_multiple_bold(self):
                """正常系: 複数の太字"""
                text = "**太字1**と**太字2**"
                rich_text = parse_inline_formatting(text)
                
                bold_count = sum(1 for r in rich_text if r.get('annotations', {}).get('bold'))
                assert bold_count == 2
            
            def test_empty_string(self):
                """エッジケース: 空文字列"""
                rich_text = parse_inline_formatting("")
                assert len(rich_text) == 1
                assert rich_text[0]['text']['content'] == ''
        
        
        # pytest実行コマンド:
        # pytest test_markdown_converter.py -v --cov=md_to_blocks
    
    - name: "generate_unit_test_runner"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11_QA実行/run_unit_tests.sh"
      content: |
        #!/bin/bash
        # ユニットテスト実行スクリプト
        
        set -e
        
        echo "🧪 ユニットテスト実行開始..."
        
        # テストディレクトリに移動
        cd "$(dirname "$0")/unit_tests"
        
        # pytest実行（カバレッジ付き）
        pytest . -v \
          --cov=../../dev \
          --cov-report=html:../test_reports/coverage_html \
          --cov-report=term \
          --cov-report=json:../test_reports/coverage.json \
          --junitxml=../test_reports/junit.xml
        
        echo ""
        echo "✅ ユニットテスト完了"
        echo "📊 カバレッジレポート: test_reports/coverage_html/index.html"
        echo "📄 JUnit XML: test_reports/junit.xml"
    
    # ========================================
    # Phase 2: 統合テスト設計・実行
    # ========================================
    - name: "confirm_integration_test_start"
      action: "confirm"
      message: |
        Phase 2: 統合テスト を開始しますか？
        
        対象: ユーザーシナリオベースのエンドツーエンドテスト
        - 新規プロジェクト立ち上げフロー
        - 既存プロジェクト同期フロー
        - コンフリクト解決フロー
        
        chrome-mcp を活用してCLI操作を自動化します。
    
    - name: "generate_integration_test_scenarios"
      action: "analyze"
      data: [
        "{{test_strategy}}",
        "{{read_files(find_files(patterns=['**/total_development_spec.md']))}}",
        "{{read_files(find_files(patterns=['**/user_story_map.yaml']))}}"
      ]
      instructions: |
        total_development_spec.md の「画面フロー」と user_story_map.yaml のストーリーから、
        統合テストシナリオを生成：
        
        1. **シナリオ1: 新規プロジェクト立ち上げ**
           - install.sh実行
           - nit init
           - .env設定
           - nit repo clone
           - ローカル編集
           - nit push
        
        2. **シナリオ2: 既存プロジェクト同期**
           - Notion側で編集
           - nit pull --full
           - コンフリクト発生
           - 手動解決
           - nit push
        
        3. **シナリオ3: 大規模プロジェクト**
           - 100ファイル以上のディレクトリ
           - nit push --changed-only
           - 差分同期確認
        
        各シナリオに対して：
        - 前提条件
        - 実行ステップ
        - 期待結果
        - 検証ポイント
        - chrome-mcp での自動化手順
      store_as: "integration_scenarios"
    
    - name: "create_integration_test_plan"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11_QA実行/integration_test_plan.md"
      content: |
        # 統合テスト計画
        
        ## 概要
        **テスト環境**: 実際のNotion API（テスト用ワークスペース）
        **自動化ツール**: chrome-mcp（CLI操作）
        **総シナリオ数**: {{integration_scenarios.total_scenarios}}件
        **推定実行時間**: {{integration_scenarios.estimated_time}}
        
        ## テストシナリオ
        {{#each integration_scenarios.scenarios}}
        ### シナリオ {{@index}}: {{name}}
        **優先度**: {{priority}}
        **ストーリー**: {{related_stories}}
        
        #### 前提条件
        {{#each preconditions}}
        - {{.}}
        {{/each}}
        
        #### 実行ステップ
        {{#each steps}}
        {{@index}}. {{action}}
           - コマンド: `{{command}}`
           - 期待結果: {{expected_result}}
        {{/each}}
        
        #### 検証ポイント
        {{#each verification_points}}
        - {{.}}
        {{/each}}
        
        #### chrome-mcp 自動化
        ```javascript
        // シナリオ{{@index}}の自動化スクリプト
        {{automation_script}}
        ```
        {{/each}}
        
        ## 実行順序
        1. 環境準備（Notion テストワークスペース作成）
        2. シナリオ1実行 → 人間確認
        3. シナリオ2実行 → 人間確認
        4. シナリオ3実行 → 人間確認
        5. 結果レポート生成
    
    - name: "generate_integration_test_scenario1"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11_QA実行/integration_tests/scenario1_new_project.md"
      content: |
        # 統合テストシナリオ1: 新規プロジェクト立ち上げ
        
        ## 目的
        ゼロから新規プロジェクトを立ち上げ、Notionにアップロードするまでの全フローを検証
        
        ## 前提条件
        - Notion APIトークンが有効
        - Notionテストワークスペースが存在
        - ローカルに`cursor_to_notion`がインストール済み
        
        ## 実行ステップ
        
        ### Step 1: install.sh実行
        ```bash
        cd /path/to/cursor_to_notion
        ./install.sh
        ```
        
        **期待結果**:
        - 仮想環境 `venv_cursor_notion` が作成される
        - `nit --help` が動作する
        - エイリアス登録完了メッセージが表示される
        
        **検証**:
        ```bash
        source venv_cursor_notion/bin/activate
        nit --help  # ヘルプが表示されることを確認
        ```
        
        ### Step 2: テストプロジェクト作成
        ```bash
        mkdir -p ~/test_project
        cd ~/test_project
        ```
        
        ### Step 3: nit init
        ```bash
        nit init
        ```
        
        **期待結果**:
        - `.c2n/config.json` が生成される
        - `.c2n/index.yaml` が生成される
        - `.c2n_ignore` が生成される
        
        **検証**:
        ```bash
        ls -la .c2n/
        cat .c2n/config.json  # デフォルト設定を確認
        ```
        
        ### Step 4: .env設定
        ```bash
        echo "NOTION_TOKEN=secret_xxx" > .env
        ```
        
        **期待結果**:
        - `.env` ファイルが作成される
        
        ### Step 5: サンプルMarkdownファイル作成
        ```bash
        mkdir docs
        cat > docs/README.md << 'EOF'
        # テストプロジェクト
        
        これはテストです。
        
        ## セクション1
        - 項目1
        - 項目2
        
        ```python
        print("Hello, World!")
        ```
        EOF
        ```
        
        ### Step 6: nit dryrun
        ```bash
        nit dryrun
        ```
        
        **期待結果**:
        - `[DRYRUN] Would create: docs/README.md` が表示される
        - 実際のファイル作成は行われない
        
        ### Step 7: nit push
        ```bash
        nit push
        ```
        
        **期待結果**:
        - 進捗表示が表示される
        - `✅ Created: https://notion.so/xxxx` が表示される
        - Notion側にページが作成される
        
        **検証**:
        1. Notionワークスペースにアクセス
        2. 新規ページが作成されていることを確認
        3. 見出し、リスト、コードブロックが正しく表示されていることを確認
        
        ### Step 8: 同期状態確認
        ```bash
        nit status
        ```
        
        **期待結果**:
        - `docs/README.md: Synced` が表示される
        - `last_sync_at` が更新されている
        
        ## 成功基準
        - [ ] 全ステップがエラーなく完了
        - [ ] Notion側にページが正しく作成される
        - [ ] Markdownの各要素が正確に変換される
        - [ ] 同期状態が正しく記録される
        
        ## chrome-mcp 自動化スクリプト
        
        **注**: chrome-mcpはブラウザ操作用のため、CLI操作は通常のシェルスクリプトで自動化。
        Notion側の表示確認のみchrome-mcpを使用。
        
        ```javascript
        // Notion側の表示確認（chrome-mcp）
        const page = await browser.newPage();
        await page.goto('https://notion.so/{{page_id}}');
        
        // 見出しの確認
        const heading = await page.$('h1');
        const headingText = await heading.textContent();
        assert(headingText === 'テストプロジェクト');
        
        // リストの確認
        const listItems = await page.$$('li');
        assert(listItems.length === 2);
        
        // コードブロックの確認
        const codeBlock = await page.$('pre code');
        const codeText = await codeBlock.textContent();
        assert(codeText.includes('print("Hello, World!")'));
        
        console.log('✅ シナリオ1: 全検証完了');
        ```
    
    - name: "checkpoint_integration_scenario1"
      action: "display"
      content: |
        ✅ 統合テストシナリオ1を生成しました
        
        **次のアクション**:
        1. シナリオ1ドキュメントを確認
        2. Notionテストワークスペースを準備
        3. シナリオ実行: 手動またはスクリプト
        4. 結果をレポートに記録
        
        続けてシナリオ2を生成しますか？
    
    # ========================================
    # Phase 3: パフォーマンステスト
    # ========================================
    - name: "confirm_performance_test_start"
      action: "confirm"
      message: |
        Phase 3: パフォーマンステスト を開始しますか？
        
        対象:
        - 100ファイル以上の大規模プロジェクト
        - 同期時間の測定
        - メモリ使用量の確認
        - SHA1ハッシュ・差分検出の効率確認
    
    - name: "generate_performance_test_plan"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11_QA実行/performance_test_plan.md"
      content: |
        # パフォーマンステスト計画
        
        ## 概要
        **目標**: 大規模プロジェクトでの実用性確認
        **測定項目**:
        1. 同期時間（全ファイル vs 差分のみ）
        2. メモリ使用量
        3. SHA1ハッシュ計算時間
        4. Notion API呼び出し回数
        
        ## テストケース
        
        ### Case 1: 100ファイルの初回Push
        **条件**:
        - 100個の.mdファイル（各1KB）
        - 階層構造: 10ディレクトリ × 10ファイル
        
        **測定**:
        ```bash
        time nit push
        ```
        
        **期待結果**:
        - 実行時間: < 3分
        - メモリ使用: < 200MB
        - API呼び出し: ~100回（ファイル数相当）
        
        ### Case 2: 差分同期（10%変更）
        **条件**:
        - 100ファイル中10ファイルのみ変更
        - `--changed-only` フラグ使用
        
        **測定**:
        ```bash
        time nit push --changed-only
        ```
        
        **期待結果**:
        - 実行時間: < 30秒（70%短縮）
        - 変更ファイルのみ同期
        - SHA1ハッシュキャッシュが有効
        
        ### Case 3: Pull（大量ページ）
        **条件**:
        - Notion側に100ページ存在
        - 全ページ取得
        
        **測定**:
        ```bash
        time nit pull --full
        ```
        
        **期待結果**:
        - 実行時間: < 5分
        - メモリ使用: < 300MB
        - last_edited_time 並列取得が有効
        
        ## 測定スクリプト
        ```bash
        #!/bin/bash
        # performance_test.sh
        
        echo "パフォーマンステスト開始"
        
        # テストデータ生成
        ./generate_test_data.sh 100
        
        # Case 1: 初回Push
        echo "Case 1: 初回Push"
        /usr/bin/time -v nit push 2>&1 | tee perf_case1.log
        
        # Case 2: 差分同期
        echo "Case 2: 差分同期"
        # 10ファイルを変更
        ./modify_files.sh 10
        /usr/bin/time -v nit push --changed-only 2>&1 | tee perf_case2.log
        
        # Case 3: Pull
        echo "Case 3: Pull"
        /usr/bin/time -v nit pull --full 2>&1 | tee perf_case3.log
        
        # レポート生成
        python generate_perf_report.py perf_case*.log
        ```
        
        ## 成功基準
        - [ ] 全ケースが目標時間内に完了
        - [ ] メモリ使用量が許容範囲内
        - [ ] 差分同期で70%以上の時間短縮
        - [ ] API呼び出し回数が最適化されている
    
    # ========================================
    # Phase 4: 総合レポート生成
    # ========================================
    - name: "generate_qa_summary_report"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11_QA実行/qa_summary_report.md"
      content: |
        # QA実行サマリーレポート
        
        **実行日時**: {{today}}
        **プロジェクト**: {{project_name}}
        **テスト対象**: {{test_strategy.total_tasks}}タスク
        
        ## Phase 1: ユニットテスト結果
        **実行テスト数**: {{unit_test_results.total_tests}}件
        **成功**: {{unit_test_results.passed}}件
        **失敗**: {{unit_test_results.failed}}件
        **スキップ**: {{unit_test_results.skipped}}件
        **カバレッジ**: {{unit_test_results.coverage}}%
        
        ### 詳細
        {{#each unit_test_results.details}}
        - **{{test_file}}**: {{status}} ({{execution_time}}秒)
        {{/each}}
        
        ### 失敗したテスト
        {{#each unit_test_results.failures}}
        - {{test_name}}: {{error_message}}
        {{/each}}
        
        ## Phase 2: 統合テスト結果
        **実行シナリオ数**: {{integration_test_results.total_scenarios}}件
        **成功**: {{integration_test_results.passed}}件
        **失敗**: {{integration_test_results.failed}}件
        
        ### 詳細
        {{#each integration_test_results.scenarios}}
        - **シナリオ{{@index}}**: {{name}} - {{status}}
          - 実行時間: {{execution_time}}
          - 検証ポイント: {{verified_points}}/{{total_points}}
        {{/each}}
        
        ## Phase 3: パフォーマンステスト結果
        **Case 1 (初回Push)**: {{perf_case1.time}}秒, {{perf_case1.memory}}MB
        **Case 2 (差分同期)**: {{perf_case2.time}}秒, {{perf_case2.memory}}MB
        **Case 3 (Pull)**: {{perf_case3.time}}秒, {{perf_case3.memory}}MB
        
        **時間短縮率**: {{time_reduction}}%
        
        ## 総合評価
        {{#if all_tests_passed}}
        ✅ **合格**: 全テストが成功しました
        {{else}}
        ⚠️ **不合格**: 一部テストが失敗しています
        {{/if}}
        
        ### 推奨アクション
        {{#each recommended_actions}}
        - {{.}}
        {{/each}}
        
        ## 次のステップ
        1. 失敗したテストの修正
        2. カバレッジ80%未満の箇所の追加テスト
        3. パフォーマンス改善（必要に応じて）
        4. リリース判定
    
    - name: "notify_qa_completion"
      action: "display"
      content: |
        ✅ QA実行プロセスが完了しました
        
        **生成されたドキュメント**:
        - unit_test_master_plan.md（ユニットテスト計画）
        - unit_tests/*.py（個別ユニットテスト）
        - run_unit_tests.sh（実行スクリプト）
        - integration_test_plan.md（統合テスト計画）
        - integration_tests/scenario*.md（シナリオ詳細）
        - performance_test_plan.md（パフォーマンステスト計画）
        - qa_summary_report.md（総合レポート）
        
        **次のアクション**:
        1. 各Phaseのテストを段階的に実行
        2. 人間確認ポイントで結果をレビュー
        3. qa_summary_report.md に結果を記録
        4. 必要に応じてテストを追加・修正
        
        全テストが完了したら、リリース判定に進んでください。
```

## 次のコマンド
→ `12_リリース判定` でリリース可否を総合評価

## 変更点（v1.0 - QA実行プロセス）
- **段階的実行**: HITL意識のPhase分割（Unit → Integration → Performance）
- **効率的設計**: dev_tasks.yaml全タスクを解析して重複なく網羅
- **pytest統合**: カバレッジ測定、JUnit XMLレポート生成
- **chrome-mcp活用**: 統合テストのNotion表示確認自動化
- **人間確認ポイント**: 各Phase完了時の確認プロセス
- **詳細レポート**: 総合QAサマリーレポート自動生成





















