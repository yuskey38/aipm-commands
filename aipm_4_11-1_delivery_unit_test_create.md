# 11-1 Delivery Unit Test Create
name: ユニットテスト作成（pytest自動生成）
---

# 11-1 : ユニットテスト作成（AIPMハッカソン）- pytest自動テストコード生成

## 前提
- 直前に `07_開発タスク分解` が完了済み（dev_tasks.yaml, total_development_spec.md が存在）
- 実装コードが存在する（dev/src/ または dev/ 配下）
- Python環境が利用可能
- **pytest + coverage を活用したユニットテスト自動生成**

## 目的
- ユニットテスト計画の自動作成
- pytest テストコードの自動生成（test_*.py）
- conftest.py, pytest.ini の自動設定
- カバレッジ80%以上を目標とした網羅的テスト設計
- モック戦略の明確化

## 実行手順
```yaml
- trigger: "(ユニットテスト作成|単体テスト作成|pytest作成|UnitTestCreate)"
  priority: high
  steps:
    # ========================================
    # Phase 0: 初期化・実装コード解析
    # ========================================
    - name: "load_implementation_context"
      action: "analyze"
      data: [
        "{{read_files(find_files(patterns=['**/total_development_spec.md']))}}",
        "{{read_files(find_files(patterns=['**/dev_tasks.yaml']))}}",
        "{{read_files(find_files(patterns=['**/dev/**/*.py']))}}",
        "{{read_files(find_files(patterns=['**/dev/**/*.js']))}}",
        "{{read_files(find_files(patterns=['**/dev_runbook.md']))}}"
      ]
      instructions: |
        実装コードを解析し、以下を抽出：
        1. **テスト対象モジュール・関数リスト**
           - Python: クラス・関数名、引数、戻り値
           - JavaScript: 関数名、主要なロジック
        2. **依存関係の特定**
           - 外部API呼び出し（Notion API等）
           - ファイルシステム操作
           - 環境変数読み込み
        3. **テストすべきエッジケース**
           - 境界値、異常系、空入力
           - エラーハンドリング
        4. **モック対象の特定**
           - API呼び出し → モック必須
           - ファイルI/O → tempfile使用
           - 環境変数 → monkeypatch使用
        
        効率的なテストグループ化戦略を提案。
      store_as: "test_analysis"
    
    - name: "create_test_structure"
      action: "execute_shell"
      command: |
        mkdir -p "Flow/{{today}}/{{flow_dir}}/11-1_ユニットテスト作成/tests" && \
        mkdir -p "Flow/{{today}}/{{flow_dir}}/11-1_ユニットテスト作成/test_data" && \
        mkdir -p "Flow/{{today}}/{{flow_dir}}/11-1_ユニットテスト作成/coverage_reports"
      message: "ユニットテスト用フォルダ構造を作成しました"
    
    - name: "display_test_analysis"
      action: "display"
      content: |
        📊 ユニットテスト作成計画
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        
        **解析結果サマリー**:
        {{test_analysis.summary}}
        
        **テスト対象モジュール**: {{test_analysis.total_modules}}個
        **テスト対象関数**: {{test_analysis.total_functions}}個
        **推定テストケース数**: {{test_analysis.estimated_test_cases}}件
        **目標カバレッジ**: 80%以上
        
        **テストグループ**:
        {{#each test_analysis.groups}}
        - **{{name}}**: {{description}} ({{test_count}}件)
        {{/each}}
        
        **モック戦略**:
        {{#each test_analysis.mock_strategy}}
        - {{target}}: {{method}}
        {{/each}}
    
    # ========================================
    # Phase 1: pytest設定ファイル生成
    # ========================================
    - name: "generate_pytest_ini"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-1_ユニットテスト作成/pytest.ini"
      content: |
        [pytest]
        # pytest設定ファイル
        
        # テストディレクトリ
        testpaths = tests
        
        # テストファイル名パターン
        python_files = test_*.py
        python_classes = Test*
        python_functions = test_*
        
        # 出力設定
        addopts = 
            -v
            --strict-markers
            --tb=short
            --cov=../dev
            --cov-report=html:coverage_reports/html
            --cov-report=term-missing
            --cov-report=json:coverage_reports/coverage.json
            --cov-branch
            --cov-fail-under=80
        
        # マーカー定義
        markers =
            unit: ユニットテスト（高速・モック使用）
            integration: 統合テスト（実API使用）
            slow: 実行に時間がかかるテスト
            skip_ci: CI環境ではスキップ
        
        # カバレッジ除外パターン
        [coverage:run]
        omit = 
            */tests/*
            */venv/*
            */__pycache__/*
            */test_*.py
        
        [coverage:report]
        precision = 2
        show_missing = True
        skip_covered = False
    
    - name: "generate_conftest"
      action: "analyze"
      data: ["{{test_analysis}}"]
      instructions: |
        test_analysis から共通フィクスチャを設計し、conftest.py を生成：
        
        1. **セットアップフィクスチャ**
           - 一時ディレクトリ作成
           - テスト用環境変数設定
           - モックオブジェクト共通初期化
        
        2. **ファイルシステムフィクスチャ**
           - temp_project_dir: 一時プロジェクトディレクトリ
           - sample_md_file: サンプルMarkdownファイル
           - sample_config: テスト用設定ファイル
        
        3. **モックフィクスチャ**
           - mock_notion_client: Notion APIモック
           - mock_env: 環境変数モック
        
        4. **データフィクスチャ**
           - sample_blocks: Notionブロックデータ
           - sample_page_response: APIレスポンスサンプル
        
        Python コードで返す。
      store_as: "conftest_code"
    
    - name: "create_conftest"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-1_ユニットテスト作成/tests/conftest.py"
      content: |
        """
        pytest共通フィクスチャ定義
        全テストファイルで利用可能な共通セットアップ
        """
        import pytest
        import tempfile
        from pathlib import Path
        import os
        import json
        from unittest.mock import Mock, MagicMock
        
        
        # ========================================
        # ファイルシステムフィクスチャ
        # ========================================
        
        @pytest.fixture
        def temp_project_dir():
            """一時プロジェクトディレクトリを作成"""
            with tempfile.TemporaryDirectory() as tmpdir:
                yield Path(tmpdir)
        
        
        @pytest.fixture
        def sample_md_file(temp_project_dir):
            """サンプルMarkdownファイルを作成"""
            md_file = temp_project_dir / "sample.md"
            content = """# サンプルドキュメント
        
        これはテスト用のMarkdownファイルです。
        
        ## セクション1
        - 項目1
        - 項目2
        
        ## セクション2
        
        ```python
        def hello():
            print("Hello, World!")
        ```
        
        **重要**: これは太字です。
        """
            md_file.write_text(content)
            return md_file
        
        
        @pytest.fixture
        def sample_config(temp_project_dir):
            """テスト用設定ファイルを作成"""
            config_dir = temp_project_dir / ".c2n"
            config_dir.mkdir()
            
            config = {
                "root_page_id": "test_root_page_id_123",
                "database_id": "test_database_id_456",
                "sync_mode": "push",
                "ignore_patterns": ["*.tmp", "__pycache__"]
            }
            
            config_file = config_dir / "config.json"
            config_file.write_text(json.dumps(config, indent=2))
            
            return config_file
        
        
        @pytest.fixture
        def sample_env_file(temp_project_dir):
            """テスト用.envファイルを作成"""
            env_file = temp_project_dir / ".env"
            env_content = "NOTION_TOKEN=secret_test_token_xxx\nWORKSPACE_ID=test_workspace_123\n"
            env_file.write_text(env_content)
            return env_file
        
        
        # ========================================
        # モックフィクスチャ
        # ========================================
        
        @pytest.fixture
        def mock_notion_client():
            """Notion APIクライアントのモック"""
            client = MagicMock()
            
            # pages.create のモック
            client.pages.create.return_value = {
                "id": "mock_page_id_123",
                "url": "https://notion.so/mock_page_123",
                "properties": {},
                "created_time": "2025-10-14T00:00:00.000Z",
                "last_edited_time": "2025-10-14T00:00:00.000Z"
            }
            
            # pages.update のモック
            client.pages.update.return_value = {
                "id": "mock_page_id_123",
                "url": "https://notion.so/mock_page_123"
            }
            
            # pages.retrieve のモック
            client.pages.retrieve.return_value = {
                "id": "mock_page_id_123",
                "properties": {
                    "title": {
                        "title": [{"text": {"content": "Test Page"}}]
                    }
                }
            }
            
            # blocks.children.append のモック
            client.blocks.children.append.return_value = {
                "results": [{"id": "block_id_1"}, {"id": "block_id_2"}]
            }
            
            return client
        
        
        @pytest.fixture
        def mock_env_vars(monkeypatch):
            """環境変数のモック"""
            test_vars = {
                "NOTION_TOKEN": "secret_test_token",
                "WORKSPACE_ID": "test_workspace",
                "DEBUG": "true"
            }
            
            for key, value in test_vars.items():
                monkeypatch.setenv(key, value)
            
            return test_vars
        
        
        # ========================================
        # データフィクスチャ
        # ========================================
        
        @pytest.fixture
        def sample_notion_blocks():
            """Notionブロックデータのサンプル"""
            return [
                {
                    "object": "block",
                    "type": "heading_1",
                    "heading_1": {
                        "rich_text": [{"type": "text", "text": {"content": "見出し1"}}]
                    }
                },
                {
                    "object": "block",
                    "type": "paragraph",
                    "paragraph": {
                        "rich_text": [{"type": "text", "text": {"content": "段落テキスト"}}]
                    }
                },
                {
                    "object": "block",
                    "type": "bulleted_list_item",
                    "bulleted_list_item": {
                        "rich_text": [{"type": "text", "text": {"content": "リスト項目1"}}]
                    }
                }
            ]
        
        
        @pytest.fixture
        def sample_page_response():
            """Notion APIのページレスポンスサンプル"""
            return {
                "object": "page",
                "id": "page_id_123",
                "created_time": "2025-10-14T00:00:00.000Z",
                "last_edited_time": "2025-10-14T01:00:00.000Z",
                "url": "https://notion.so/page_id_123",
                "properties": {
                    "title": {
                        "id": "title",
                        "type": "title",
                        "title": [{"type": "text", "text": {"content": "テストページ"}}]
                    }
                }
            }
        
        
        @pytest.fixture
        def sample_hash_cache(temp_project_dir):
            """SHA1ハッシュキャッシュのサンプル"""
            cache_file = temp_project_dir / ".c2n" / "hash_cache.json"
            cache_file.parent.mkdir(exist_ok=True)
            
            cache_data = {
                "files/doc1.md": "abc123def456",
                "files/doc2.md": "789xyz012uvw"
            }
            
            cache_file.write_text(json.dumps(cache_data, indent=2))
            return cache_file
        
        
        # ========================================
        # ヘルパー関数
        # ========================================
        
        def assert_blocks_equal(actual_blocks, expected_blocks):
            """Notionブロックリストの比較ヘルパー"""
            assert len(actual_blocks) == len(expected_blocks), \
                f"ブロック数が一致しません: {len(actual_blocks)} != {len(expected_blocks)}"
            
            for i, (actual, expected) in enumerate(zip(actual_blocks, expected_blocks)):
                assert actual['type'] == expected['type'], \
                    f"ブロック{i}のtypeが一致しません: {actual['type']} != {expected['type']}"
    
    - name: "display_pytest_config_complete"
      action: "display"
      content: |
        ✅ pytest設定ファイル生成完了
        
        **生成ファイル**:
        - pytest.ini: pytest設定（カバレッジ80%必須、マーカー定義）
        - tests/conftest.py: 共通フィクスチャ（ファイルシステム、モック、データ）
        
        **利用可能なフィクスチャ**:
        - temp_project_dir: 一時プロジェクトディレクトリ
        - sample_md_file: サンプルMarkdownファイル
        - sample_config: テスト用設定ファイル
        - mock_notion_client: Notion APIモック
        - mock_env_vars: 環境変数モック
        - sample_notion_blocks: Notionブロックサンプル
    
    # ========================================
    # Phase 2: テストコード自動生成
    # ========================================
    - name: "confirm_test_generation"
      action: "confirm"
      message: |
        Phase 2: テストコード自動生成を開始しますか？
        
        以下のテストファイルを生成します：
        {{#each test_analysis.test_files}}
        - {{filename}}: {{description}} ({{test_count}}件)
        {{/each}}
    
    # Batch 1: 環境変数読み込みテスト
    - name: "generate_test_env_loader"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-1_ユニットテスト作成/tests/test_env_loader.py"
      content: |
        """
        ユニットテスト: 環境変数読み込み機能
        対象: 環境変数読み込み関数（_load_env_file, _load_env_for_target等）
        """
        import pytest
        import tempfile
        from pathlib import Path
        import os
        
        # テスト対象のインポート（パスは実装に応じて調整）
        import sys
        sys.path.insert(0, str(Path(__file__).parents[4] / 'dev'))
        # from your_module import _load_env_file, _load_env_for_target
        
        
        @pytest.mark.unit
        class TestLoadEnvFile:
            """環境変数ファイル読み込みのテスト"""
            
            def test_normal_env_file(self, temp_project_dir):
                """正常系: 標準的な.envファイルを読み込める"""
                env_file = temp_project_dir / ".env"
                env_file.write_text("NOTION_TOKEN=secret_xxx\nWORKSPACE_ID=workspace_123\n")
                
                # TODO: 実装に応じて関数呼び出しを追加
                # result = _load_env_file(env_file)
                # assert result['NOTION_TOKEN'] == 'secret_xxx'
                # assert result['WORKSPACE_ID'] == 'workspace_123'
                pass
            
            def test_env_with_comments(self, temp_project_dir):
                """正常系: コメント行を無視する"""
                env_file = temp_project_dir / ".env"
                env_file.write_text("# コメント\nNOTION_TOKEN=secret_xxx\n# 別のコメント\n")
                
                # TODO: 実装に応じて関数呼び出しを追加
                # result = _load_env_file(env_file)
                # assert '# コメント' not in result
                # assert 'NOTION_TOKEN' in result
                pass
            
            def test_env_with_empty_lines(self, temp_project_dir):
                """正常系: 空行を無視する"""
                env_file = temp_project_dir / ".env"
                env_file.write_text("\nNOTION_TOKEN=secret_xxx\n\n\n")
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
            
            def test_file_not_found(self):
                """異常系: ファイルが存在しない場合"""
                # TODO: 実装に応じて関数呼び出しを追加
                # result = _load_env_file(Path('/nonexistent/.env'))
                # assert result == {}
                pass
            
            def test_malformed_line(self, temp_project_dir):
                """エッジケース: =がない行は無視"""
                env_file = temp_project_dir / ".env"
                env_file.write_text("NOTION_TOKEN=secret_xxx\nMALFORMED_LINE\n")
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
            
            def test_empty_value(self, temp_project_dir):
                """エッジケース: 値が空の変数"""
                env_file = temp_project_dir / ".env"
                env_file.write_text("EMPTY_VAR=\n")
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
        
        
        @pytest.mark.unit
        class TestLoadEnvForTarget:
            """環境変数優先順位読み込みのテスト"""
            
            def test_priority_env_variable(self, monkeypatch, temp_project_dir):
                """優先順位1: 環境変数が最優先"""
                monkeypatch.setenv('NOTION_TOKEN', 'env_token_highest')
                
                # 低優先度の設定も配置
                (temp_project_dir / ".env").write_text("NOTION_TOKEN=project_low")
                
                # TODO: 実装に応じて関数呼び出しを追加
                # token = _load_env_for_target(temp_project_dir)
                # assert token == 'env_token_highest'
                pass
            
            def test_priority_c2n_env(self, monkeypatch, temp_project_dir):
                """優先順位2: .c2n/.env"""
                monkeypatch.delenv('NOTION_TOKEN', raising=False)
                
                c2n_dir = temp_project_dir / ".c2n"
                c2n_dir.mkdir()
                (c2n_dir / ".env").write_text("NOTION_TOKEN=c2n_token_medium")
                (temp_project_dir / ".env").write_text("NOTION_TOKEN=project_low")
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
            
            def test_token_not_found(self, monkeypatch, temp_project_dir):
                """異常系: トークンが見つからない場合はValueError"""
                monkeypatch.delenv('NOTION_TOKEN', raising=False)
                
                # TODO: 実装に応じて関数呼び出しを追加
                # with pytest.raises(ValueError, match="NOTION_TOKEN not found"):
                #     _load_env_for_target(temp_project_dir)
                pass
        
        
        # pytest実行コマンド:
        # pytest tests/test_env_loader.py -v -m unit
    
    # Batch 2: Markdown変換テスト
    - name: "generate_test_markdown_converter"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-1_ユニットテスト作成/tests/test_markdown_converter.py"
      content: |
        """
        ユニットテスト: Markdown変換機能
        対象: Markdown→Notionブロック変換関数
        """
        import pytest
        from pathlib import Path
        import sys
        
        sys.path.insert(0, str(Path(__file__).parents[4] / 'dev'))
        # from your_module import convert_markdown_to_notion_blocks, parse_inline_formatting
        
        
        @pytest.mark.unit
        class TestConvertMarkdownToNotionBlocks:
            """Markdown→Notionブロック変換のテスト"""
            
            def test_heading_conversion(self):
                """正常系: 見出しの変換"""
                markdown = "# 見出し1\n## 見出し2\n### 見出し3"
                
                # TODO: 実装に応じて関数呼び出しを追加
                # blocks = convert_markdown_to_notion_blocks(markdown)
                # assert len(blocks) == 3
                # assert blocks[0]['type'] == 'heading_1'
                # assert blocks[1]['type'] == 'heading_2'
                # assert blocks[2]['type'] == 'heading_3'
                pass
            
            def test_list_conversion(self):
                """正常系: リストの変換"""
                markdown = "- 項目1\n- 項目2\n1. 番号1\n2. 番号2"
                
                # TODO: 実装に応じて関数呼び出しを追加
                # blocks = convert_markdown_to_notion_blocks(markdown)
                # assert blocks[0]['type'] == 'bulleted_list_item'
                # assert blocks[2]['type'] == 'numbered_list_item'
                pass
            
            def test_code_block_conversion(self):
                """正常系: コードブロックの変換"""
                markdown = "```python\nprint('Hello')\n```"
                
                # TODO: 実装に応じて関数呼び出しを追加
                # blocks = convert_markdown_to_notion_blocks(markdown)
                # assert blocks[0]['type'] == 'code'
                # assert blocks[0]['code']['language'] == 'python'
                pass
            
            def test_paragraph_conversion(self):
                """正常系: 通常段落の変換"""
                markdown = "これは段落です。"
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
            
            def test_empty_input(self):
                """エッジケース: 空文字列"""
                # TODO: 実装に応じて関数呼び出しを追加
                # blocks = convert_markdown_to_notion_blocks("")
                # assert blocks == []
                pass
            
            def test_mixed_content(self, sample_md_file):
                """正常系: 複数の要素が混在（フィクスチャ活用）"""
                markdown = sample_md_file.read_text()
                
                # TODO: 実装に応じて関数呼び出しを追加
                # blocks = convert_markdown_to_notion_blocks(markdown)
                # types = [b['type'] for b in blocks]
                # assert 'heading_1' in types
                # assert 'paragraph' in types
                # assert 'bulleted_list_item' in types
                # assert 'code' in types
                pass
            
            def test_nested_lists(self):
                """正常系: ネストされたリスト"""
                markdown = "- 親項目1\n  - 子項目1-1\n  - 子項目1-2\n- 親項目2"
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
            
            def test_table_conversion(self):
                """正常系: テーブルの変換"""
                markdown = "| Header1 | Header2 |\n|---------|----------|\n| Cell1   | Cell2    |"
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
        
        
        @pytest.mark.unit
        class TestParseInlineFormatting:
            """インラインフォーマット解析のテスト"""
            
            def test_bold_text(self):
                """正常系: 太字の解析"""
                text = "これは**太字**です"
                
                # TODO: 実装に応じて関数呼び出しを追加
                # rich_text = parse_inline_formatting(text)
                # assert rich_text[1]['annotations']['bold'] is True
                pass
            
            def test_italic_text(self):
                """正常系: 斜体の解析"""
                text = "これは*斜体*です"
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
            
            def test_code_inline(self):
                """正常系: インラインコードの解析"""
                text = "これは`code`です"
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
            
            def test_link(self):
                """正常系: リンクの解析"""
                text = "これは[リンク](https://example.com)です"
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
            
            def test_no_formatting(self):
                """正常系: フォーマットなし"""
                text = "プレーンテキスト"
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
            
            def test_multiple_bold(self):
                """正常系: 複数の太字"""
                text = "**太字1**と**太字2**"
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
            
            def test_empty_string(self):
                """エッジケース: 空文字列"""
                # TODO: 実装に応じて関数呼び出しを追加
                pass
        
        
        # pytest実行コマンド:
        # pytest tests/test_markdown_converter.py -v -m unit --cov=md_to_blocks
    
    # Batch 3: ハッシュキャッシュテスト
    - name: "generate_test_hash_cache"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-1_ユニットテスト作成/tests/test_hash_cache.py"
      content: |
        """
        ユニットテスト: SHA1ハッシュキャッシュ機能
        対象: ハッシュ計算・キャッシュ管理関数
        """
        import pytest
        from pathlib import Path
        import json
        import sys
        
        sys.path.insert(0, str(Path(__file__).parents[4] / 'dev'))
        # from your_module import compute_file_hash, load_hash_cache, save_hash_cache, is_file_changed
        
        
        @pytest.mark.unit
        class TestComputeFileHash:
            """ファイルハッシュ計算のテスト"""
            
            def test_compute_hash_normal(self, temp_project_dir):
                """正常系: 通常のファイルのハッシュ計算"""
                test_file = temp_project_dir / "test.txt"
                test_file.write_text("Hello, World!")
                
                # TODO: 実装に応じて関数呼び出しを追加
                # hash_value = compute_file_hash(test_file)
                # assert isinstance(hash_value, str)
                # assert len(hash_value) == 40  # SHA1は40文字
                pass
            
            def test_compute_hash_empty_file(self, temp_project_dir):
                """エッジケース: 空ファイル"""
                test_file = temp_project_dir / "empty.txt"
                test_file.write_text("")
                
                # TODO: 実装に応じて関数呼び出しを追加
                # hash_value = compute_file_hash(test_file)
                # 空ファイルのSHA1: da39a3ee5e6b4b0d3255bfef95601890afd80709
                # assert hash_value == "da39a3ee5e6b4b0d3255bfef95601890afd80709"
                pass
            
            def test_compute_hash_large_file(self, temp_project_dir):
                """正常系: 大きなファイル"""
                test_file = temp_project_dir / "large.txt"
                test_file.write_text("x" * 1_000_000)  # 1MB
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
            
            def test_compute_hash_binary_file(self, temp_project_dir):
                """正常系: バイナリファイル"""
                test_file = temp_project_dir / "binary.bin"
                test_file.write_bytes(bytes([0, 1, 2, 3, 255]))
                
                # TODO: 実装に応じて関数呼び出しを追加
                pass
            
            def test_file_not_found(self):
                """異常系: ファイルが存在しない"""
                # TODO: 実装に応じて関数呼び出しを追加
                # with pytest.raises(FileNotFoundError):
                #     compute_file_hash(Path('/nonexistent/file.txt'))
                pass
            
            def test_same_content_same_hash(self, temp_project_dir):
                """正常系: 同じ内容なら同じハッシュ"""
                file1 = temp_project_dir / "file1.txt"
                file2 = temp_project_dir / "file2.txt"
                content = "Same content"
                file1.write_text(content)
                file2.write_text(content)
                
                # TODO: 実装に応じて関数呼び出しを追加
                # hash1 = compute_file_hash(file1)
                # hash2 = compute_file_hash(file2)
                # assert hash1 == hash2
                pass
        
        
        @pytest.mark.unit
        class TestHashCache:
            """ハッシュキャッシュ管理のテスト"""
            
            def test_load_cache_existing(self, sample_hash_cache):
                """正常系: 既存のキャッシュを読み込む"""
                # TODO: 実装に応じて関数呼び出しを追加
                # cache = load_hash_cache(sample_hash_cache)
                # assert "files/doc1.md" in cache
                # assert cache["files/doc1.md"] == "abc123def456"
                pass
            
            def test_load_cache_not_found(self, temp_project_dir):
                """正常系: キャッシュファイルがない場合は空辞書"""
                cache_path = temp_project_dir / "nonexistent_cache.json"
                
                # TODO: 実装に応じて関数呼び出しを追加
                # cache = load_hash_cache(cache_path)
                # assert cache == {}
                pass
            
            def test_save_cache(self, temp_project_dir):
                """正常系: キャッシュを保存"""
                cache_path = temp_project_dir / "cache.json"
                cache_data = {
                    "file1.md": "hash1",
                    "file2.md": "hash2"
                }
                
                # TODO: 実装に応じて関数呼び出しを追加
                # save_hash_cache(cache_path, cache_data)
                # assert cache_path.exists()
                # loaded = json.loads(cache_path.read_text())
                # assert loaded == cache_data
                pass
            
            def test_save_cache_overwrites(self, sample_hash_cache):
                """正常系: 既存キャッシュを上書き"""
                new_cache = {"new_file.md": "new_hash"}
                
                # TODO: 実装に応じて関数呼び出しを追加
                # save_hash_cache(sample_hash_cache, new_cache)
                # loaded = load_hash_cache(sample_hash_cache)
                # assert loaded == new_cache
                pass
        
        
        @pytest.mark.unit
        class TestIsFileChanged:
            """ファイル変更検知のテスト"""
            
            def test_file_changed(self, temp_project_dir):
                """正常系: ファイルが変更された"""
                test_file = temp_project_dir / "test.md"
                test_file.write_text("Original content")
                
                # TODO: 実装に応じて関数呼び出しを追加
                # original_hash = compute_file_hash(test_file)
                # cache = {"test.md": original_hash}
                
                # ファイルを変更
                # test_file.write_text("Modified content")
                
                # changed = is_file_changed(test_file, cache)
                # assert changed is True
                pass
            
            def test_file_unchanged(self, temp_project_dir):
                """正常系: ファイルが変更されていない"""
                test_file = temp_project_dir / "test.md"
                test_file.write_text("Content")
                
                # TODO: 実装に応じて関数呼び出しを追加
                # hash_value = compute_file_hash(test_file)
                # cache = {"test.md": hash_value}
                
                # changed = is_file_changed(test_file, cache)
                # assert changed is False
                pass
            
            def test_file_not_in_cache(self, temp_project_dir):
                """正常系: キャッシュにない新規ファイル"""
                test_file = temp_project_dir / "new.md"
                test_file.write_text("New file")
                cache = {}
                
                # TODO: 実装に応じて関数呼び出しを追加
                # changed = is_file_changed(test_file, cache)
                # assert changed is True
                pass
        
        
        # pytest実行コマンド:
        # pytest tests/test_hash_cache.py -v -m unit --cov=hash_utils
    
    # ========================================
    # Phase 3: テスト実行スクリプト生成
    # ========================================
    - name: "generate_test_runner"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-1_ユニットテスト作成/run_unit_tests.sh"
      content: |
        #!/bin/bash
        # ユニットテスト実行スクリプト
        
        set -e
        
        echo "🧪 ユニットテスト実行開始..."
        echo ""
        
        # スクリプトのディレクトリに移動
        SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
        cd "$SCRIPT_DIR"
        
        # Python仮想環境の確認
        if [ ! -d "venv" ]; then
            echo "⚠️  仮想環境が見つかりません。作成します..."
            python3 -m venv venv
            source venv/bin/activate
            pip install -r requirements.txt
        else
            source venv/bin/activate
        fi
        
        # pytestのインストール確認
        if ! pip show pytest > /dev/null 2>&1; then
            echo "📦 pytestをインストールします..."
            pip install pytest pytest-cov pytest-mock
        fi
        
        # テスト実行
        echo "🏃 テスト実行中..."
        pytest tests/ -v \
          -m unit \
          --cov=../dev \
          --cov-report=html:coverage_reports/html \
          --cov-report=term-missing \
          --cov-report=json:coverage_reports/coverage.json \
          --junitxml=coverage_reports/junit.xml \
          --tb=short
        
        RESULT=$?
        
        echo ""
        if [ $RESULT -eq 0 ]; then
            echo "✅ 全テスト成功"
        else
            echo "❌ テスト失敗（詳細は上記ログを確認）"
        fi
        
        echo ""
        echo "📊 カバレッジレポート:"
        echo "  HTML: coverage_reports/html/index.html"
        echo "  JSON: coverage_reports/coverage.json"
        echo "  JUnit XML: coverage_reports/junit.xml"
        
        exit $RESULT
    
    - name: "generate_requirements_txt"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-1_ユニットテスト作成/requirements.txt"
      content: |
        # ユニットテスト用Python依存関係
        
        # テストフレームワーク
        pytest>=7.4.0
        pytest-cov>=4.1.0
        pytest-mock>=3.11.1
        
        # カバレッジ
        coverage>=7.2.0
        
        # アサーション拡張
        pytest-html>=3.2.0
        
        # 並列実行（オプション）
        pytest-xdist>=3.3.0
        
        # 実装コードの依存関係（必要に応じて追加）
        # requests>=2.31.0
        # notion-client>=2.0.0
    
    - name: "generate_test_readme"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-1_ユニットテスト作成/README.md"
      content: |
        # ユニットテスト実行ガイド
        
        ## 概要
        このディレクトリには、プロジェクトのユニットテストコードとpytest設定が含まれています。
        
        **目標カバレッジ**: 80%以上
        **テストフレームワーク**: pytest
        **実行環境**: Python 3.8+
        
        ## ディレクトリ構造
        ```
        11-1_ユニットテスト作成/
        ├── tests/
        │   ├── conftest.py            # 共通フィクスチャ
        │   ├── test_env_loader.py     # 環境変数読み込みテスト
        │   ├── test_markdown_converter.py  # Markdown変換テスト
        │   └── test_hash_cache.py     # ハッシュキャッシュテスト
        ├── test_data/                 # テスト用データ
        ├── coverage_reports/          # カバレッジレポート（実行後生成）
        ├── pytest.ini                 # pytest設定
        ├── requirements.txt           # Python依存関係
        ├── run_unit_tests.sh          # テスト実行スクリプト
        └── README.md                  # このファイル
        ```
        
        ## セットアップ
        
        ### 1. 仮想環境作成（推奨）
        ```bash
        cd 11-1_ユニットテスト作成
        python3 -m venv venv
        source venv/bin/activate  # Windows: venv\Scripts\activate
        ```
        
        ### 2. 依存関係インストール
        ```bash
        pip install -r requirements.txt
        ```
        
        ## テスト実行
        
        ### 方法1: スクリプト実行（推奨）
        ```bash
        ./run_unit_tests.sh
        ```
        
        ### 方法2: pytest直接実行
        ```bash
        # 全ユニットテスト実行
        pytest tests/ -v -m unit
        
        # カバレッジ付き実行
        pytest tests/ -v -m unit --cov=../dev --cov-report=html
        
        # 特定のテストファイルのみ
        pytest tests/test_env_loader.py -v
        
        # 特定のテストクラスのみ
        pytest tests/test_env_loader.py::TestLoadEnvFile -v
        
        # 特定のテスト関数のみ
        pytest tests/test_env_loader.py::TestLoadEnvFile::test_normal_env_file -v
        ```
        
        ### 方法3: 並列実行（高速化）
        ```bash
        pytest tests/ -v -m unit -n auto
        ```
        
        ## カバレッジ確認
        
        ### HTMLレポート
        ```bash
        pytest tests/ --cov=../dev --cov-report=html
        open coverage_reports/html/index.html  # ブラウザで開く
        ```
        
        ### ターミナル表示
        ```bash
        pytest tests/ --cov=../dev --cov-report=term-missing
        ```
        
        ### JSON出力（CI連携用）
        ```bash
        pytest tests/ --cov=../dev --cov-report=json:coverage_reports/coverage.json
        ```
        
        ## テストマーカー
        
        pytest.ini で定義されたマーカー:
        
        - `@pytest.mark.unit`: ユニットテスト（高速・モック使用）
        - `@pytest.mark.integration`: 統合テスト（実API使用）
        - `@pytest.mark.slow`: 実行に時間がかかるテスト
        - `@pytest.mark.skip_ci`: CI環境ではスキップ
        
        ### マーカー指定実行
        ```bash
        # ユニットテストのみ
        pytest -m unit
        
        # 遅いテストを除外
        pytest -m "not slow"
        
        # ユニットテストかつ遅くない
        pytest -m "unit and not slow"
        ```
        
        ## 利用可能なフィクスチャ
        
        conftest.py で定義された共通フィクスチャ:
        
        ### ファイルシステム
        - `temp_project_dir`: 一時プロジェクトディレクトリ
        - `sample_md_file`: サンプルMarkdownファイル
        - `sample_config`: テスト用設定ファイル
        - `sample_env_file`: テスト用.envファイル
        
        ### モック
        - `mock_notion_client`: Notion APIクライアントモック
        - `mock_env_vars`: 環境変数モック
        
        ### データ
        - `sample_notion_blocks`: Notionブロックサンプル
        - `sample_page_response`: APIレスポンスサンプル
        - `sample_hash_cache`: ハッシュキャッシュサンプル
        
        ## トラブルシューティング
        
        ### ImportError
        ```python
        # テストファイル先頭にパス追加
        import sys
        from pathlib import Path
        sys.path.insert(0, str(Path(__file__).parents[4] / 'dev'))
        ```
        
        ### カバレッジが低い
        - 異常系テストケースを追加
        - エッジケーステストを追加
        - `--cov-report=term-missing` で未カバー箇所を確認
        
        ### テスト実行が遅い
        - 並列実行を試す: `pytest -n auto`
        - `@pytest.mark.slow` でマーキングして除外
        
        ## CI/CD連携
        
        GitHub Actions等での実行例:
        ```yaml
        - name: Run Unit Tests
          run: |
            pip install -r requirements.txt
            pytest tests/ -v -m unit \
              --cov=../dev \
              --cov-report=xml \
              --junitxml=junit.xml
        
        - name: Upload Coverage
          uses: codecov/codecov-action@v3
          with:
            files: ./coverage.xml
        ```
        
        ## 次のステップ
        1. `TODO` コメントを実装コードに合わせて完成させる
        2. テストを実行してカバレッジを確認
        3. カバレッジ80%未満の箇所に追加テストを作成
        4. 統合テスト（11_QA実行）へ進む
    
    - name: "generate_unit_test_plan_document"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-1_ユニットテスト作成/unit_test_plan.md"
      content: |
        # ユニットテスト計画書
        
        **作成日**: {{today}}
        **プロジェクト**: {{project_name}}
        **目標カバレッジ**: 80%以上
        
        ## 1. テスト戦略
        
        ### 1.1 基本方針
        - **単体テスト**: 各関数・クラスを独立してテスト
        - **モック活用**: 外部依存を排除（Notion API、ファイルシステム）
        - **高速実行**: 全テスト < 30秒を目標
        - **自動化**: CI/CD統合を前提
        
        ### 1.2 テスト範囲
        {{test_analysis.summary}}
        
        **対象モジュール**: {{test_analysis.total_modules}}個
        **対象関数**: {{test_analysis.total_functions}}個
        **推定テストケース**: {{test_analysis.estimated_test_cases}}件
        
        ### 1.3 除外範囲
        - UI/CLI部分（統合テストで検証）
        - サードパーティライブラリ自体のテスト
        - 設定ファイル（YAML/JSON）の妥当性（バリデーションテストで検証）
        
        ## 2. テストグループ
        
        {{#each test_analysis.groups}}
        ### Group {{@index}}: {{name}}
        **優先度**: {{priority}}
        **説明**: {{description}}
        **テスト数**: {{test_count}}件
        **テストファイル**: `{{test_file}}`
        
        #### 対象関数
        {{#each functions}}
        - `{{name}}`: {{description}}
          - 正常系: {{normal_cases}}件
          - 異常系: {{error_cases}}件
          - エッジケース: {{edge_cases}}件
        {{/each}}
        
        {{/each}}
        
        ## 3. モック戦略
        
        ### 3.1 Notion API
        **方針**: `unittest.mock.MagicMock` で全エンドポイントをモック
        
        ```python
        @pytest.fixture
        def mock_notion_client():
            client = MagicMock()
            client.pages.create.return_value = {...}
            return client
        ```
        
        **対象エンドポイント**:
        - `pages.create`
        - `pages.update`
        - `pages.retrieve`
        - `blocks.children.append`
        - `databases.query`
        
        ### 3.2 ファイルシステム
        **方針**: `tempfile.TemporaryDirectory` で一時ディレクトリ使用
        
        ```python
        @pytest.fixture
        def temp_project_dir():
            with tempfile.TemporaryDirectory() as tmpdir:
                yield Path(tmpdir)
        ```
        
        ### 3.3 環境変数
        **方針**: `monkeypatch` で動的設定
        
        ```python
        def test_env_loading(monkeypatch):
            monkeypatch.setenv('NOTION_TOKEN', 'test_token')
        ```
        
        ## 4. カバレッジ目標
        
        | モジュール | 目標カバレッジ | 理由 |
        |-----------|---------------|------|
        | 環境変数読み込み | 90%+ | クリティカルパス |
        | Markdown変換 | 85%+ | コア機能 |
        | ハッシュキャッシュ | 85%+ | パフォーマンス重要 |
        | Push/Pull | 80%+ | 統合テストで補完 |
        | その他ユーティリティ | 70%+ | 低優先度 |
        
        **全体目標**: 80%以上
        
        ## 5. テストケース設計
        
        ### 5.1 正常系テスト
        - 標準的な入力での動作確認
        - 期待される出力の検証
        - 副作用（ファイル作成等）の確認
        
        ### 5.2 異常系テスト
        - ファイルが存在しない
        - APIエラーレスポンス
        - 環境変数未設定
        - ネットワークタイムアウト
        
        ### 5.3 エッジケーステスト
        - 空文字列
        - 空ファイル
        - 大きなファイル（1MB+）
        - 特殊文字を含む入力
        - 境界値（0, -1, MAX等）
        
        ## 6. 実行計画
        
        ### Phase 1: 基盤関数（優先度: 高）
        - `test_env_loader.py`: 環境変数読み込み
        - 実行時間: ~5秒
        - カバレッジ期待: 90%+
        
        ### Phase 2: コア機能（優先度: 高）
        - `test_markdown_converter.py`: Markdown変換
        - 実行時間: ~10秒
        - カバレッジ期待: 85%+
        
        ### Phase 3: パフォーマンス機能（優先度: 中）
        - `test_hash_cache.py`: ハッシュキャッシュ
        - 実行時間: ~5秒
        - カバレッジ期待: 85%+
        
        ### Phase 4: その他（優先度: 中）
        - 追加のユーティリティ関数テスト
        - 実行時間: ~10秒
        - カバレッジ期待: 70%+
        
        **総実行時間**: ~30秒
        
        ## 7. 品質基準
        
        ### 合格基準
        - [ ] 全テストが成功（0 failed）
        - [ ] カバレッジ80%以上
        - [ ] 実行時間 < 60秒
        - [ ] 警告（warnings）が0件
        
        ### レビュー観点
        - [ ] テストケース名がわかりやすい
        - [ ] アサーションが適切
        - [ ] モックが正しく設定されている
        - [ ] エッジケースがカバーされている
        
        ## 8. 次のステップ
        1. テストコードの `TODO` を実装
        2. `./run_unit_tests.sh` で実行
        3. カバレッジレポートを確認
        4. カバレッジ不足箇所に追加テスト
        5. 統合テスト（11_QA実行）へ進む
    
    - name: "notify_completion"
      action: "display"
      content: |
        ✅ ユニットテスト作成が完了しました
        
        **生成されたファイル**:
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        📄 設定ファイル:
        - pytest.ini: pytest設定（カバレッジ80%必須）
        - requirements.txt: Python依存関係
        
        📁 テストコード:
        - tests/conftest.py: 共通フィクスチャ
        - tests/test_env_loader.py: 環境変数読み込みテスト
        - tests/test_markdown_converter.py: Markdown変換テスト
        - tests/test_hash_cache.py: ハッシュキャッシュテスト
        
        🔧 実行スクリプト:
        - run_unit_tests.sh: ワンコマンド実行
        
        📖 ドキュメント:
        - README.md: 実行ガイド
        - unit_test_plan.md: テスト計画書
        
        **次のアクション**:
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        1. 実装コードに合わせてテストコードの `TODO` を完成させる
        2. テスト実行:
           ```bash
           cd 11-1_ユニットテスト作成
           ./run_unit_tests.sh
           ```
        3. カバレッジレポートを確認:
           ```bash
           open coverage_reports/html/index.html
           ```
        4. カバレッジ80%未満の箇所に追加テストを作成
        5. 全テスト成功後、`11_QA実行` で統合テストへ進む
        
        **重要**:
        - 現在のテストコードには `TODO` コメントが含まれています
        - 実際の実装コードに合わせて、関数呼び出しとアサーションを追加してください
        - モックフィクスチャ（conftest.py）は実装に応じてカスタマイズしてください
```

## 次のコマンド
→ `11_QA実行` で統合テスト・パフォーマンステストへ進む

## 特徴
- **pytest + coverage統合**: カバレッジ80%を強制する設定
- **conftest.py自動生成**: 再利用可能なフィクスチャを提供
- **モック戦略明確化**: Notion API、ファイルシステム、環境変数のモック方針
- **実行スクリプト**: ワンコマンドでテスト実行・カバレッジ確認
- **詳細ドキュメント**: README.mdとunit_test_plan.mdで運用ガイド完備
- **TODO形式**: 実装コードに合わせてカスタマイズ可能なテンプレート
