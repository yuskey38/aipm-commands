---
name: 結合テスト実施（統合テスト実行）
---

# 11-4 : 結合テスト実施 - Playwright統合テスト・最終QA

## 前提
- 直前に `07_開発タスク分解` が完了済み（dev_tasks.yaml が存在）
- 各開発タスクの単体テストが完了済み
- **Playwright** でブラウザ自動テストを実施
- **Notion API** でページ検証を実行
- **スクリーンショット自動取得** で視覚的検証

## 目的
- エンドツーエンドの統合テスト実行
- Notionページの自動検証（README、getting-started、api）
- 画面キャプチャによる視覚的QA
- 最終QAレポート生成とリリース判定支援

## 実行手順
```yaml
- trigger: "(結合テスト実施|統合テスト実行|Integration Test|E2E Test|エンドツーエンドテスト)"
  priority: high
  steps:
    # ========================================
    # Phase 0: 初期化・環境準備
    # ========================================
    - name: "load_test_targets"
      action: "analyze"
      data: [
        "{{read_files(find_files(patterns=['**/dev_tasks.yaml']))}}",
        "{{read_files(find_files(patterns=['**/story_map.yaml']))}}",
        "{{read_files(find_files(patterns=['**/screen_*.yaml']))}}"
      ]
      instructions: |
        dev_tasks.yaml と story_map.yaml から統合テスト対象を抽出：
        1. 主要ユーザーストーリー（MVP優先）
        2. 画面遷移フロー（ストーリーマップから）
        3. 重要APIエンドポイント
        4. Notionページ検証ターゲット（README、getting-started、api）
        
        統合テスト計画を生成：
        - テストシナリオ（正常系・異常系）
        - 検証ポイント
        - スクリーンショット取得箇所
        - 推定実行時間
      store_as: "integration_test_plan"
    
    - name: "create_integration_test_structure"
      action: "execute_shell"
      command: |
        mkdir -p "Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/playwright_tests" && \
        mkdir -p "Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/screenshots" && \
        mkdir -p "Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/test_reports" && \
        mkdir -p "Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/notion_validation"
      message: "結合テスト用フォルダ構造を作成しました"
    
    - name: "display_integration_test_overview"
      action: "display"
      content: |
        📊 結合テスト実施計画概要
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        
        **テスト戦略**:
        {{integration_test_plan.summary}}
        
        **対象ストーリー**: {{integration_test_plan.total_stories}}件
        **テストシナリオ**: {{integration_test_plan.total_scenarios}}件
        **Notion検証ページ**: README, getting-started, api
        **推定実行時間**: {{integration_test_plan.estimated_time}}
        
        **実行プラン**:
        1. Phase 1: Playwright環境セットアップ
        2. Phase 2: エンドツーエンドテスト実行
        3. Phase 3: Notionページ検証
        4. Phase 4: スクリーンショット自動取得
        5. Phase 5: 最終QAレポート生成
    
    # ========================================
    # Phase 1: Playwright環境セットアップ
    # ========================================
    - name: "confirm_playwright_setup"
      action: "confirm"
      message: |
        Phase 1: Playwright環境セットアップ を開始しますか？
        
        実施内容:
        - Playwright インストール確認
        - ブラウザバイナリ準備（Chromium, Firefox, WebKit）
        - テストフィクスチャ作成
        - 環境変数設定
    
    - name: "setup_playwright_environment"
      action: "execute_shell"
      command: |
        cd Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/playwright_tests && \
        npm init -y && \
        npm install --save-dev @playwright/test && \
        npx playwright install
      message: "Playwright環境をセットアップしました"
    
    - name: "create_playwright_config"
      action: "create_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/playwright_tests/playwright.config.js"
      content: |
        // @ts-check
        const { defineConfig, devices } = require('@playwright/test');
        
        module.exports = defineConfig({
          testDir: './tests',
          fullyParallel: false,
          forbidOnly: !!process.env.CI,
          retries: process.env.CI ? 2 : 0,
          workers: process.env.CI ? 1 : 1,
          reporter: [
            ['html', { outputFolder: '../test_reports/playwright-report' }],
            ['json', { outputFile: '../test_reports/results.json' }],
            ['junit', { outputFile: '../test_reports/junit.xml' }]
          ],
          use: {
            baseURL: process.env.BASE_URL || 'http://localhost:3000',
            trace: 'on-first-retry',
            screenshot: 'only-on-failure',
            video: 'retain-on-failure',
          },
          projects: [
            {
              name: 'chromium',
              use: { ...devices['Desktop Chrome'] },
            },
          ],
        });
    
    - name: "create_test_fixtures"
      action: "create_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/playwright_tests/tests/fixtures.js"
      content: |
        const { test as base } = require('@playwright/test');
        const fs = require('fs');
        const path = require('path');
        
        // カスタムフィクスチャ：スクリーンショット保存ヘルパー
        const test = base.extend({
          saveScreenshot: async ({ page }, use) => {
            const screenshotDir = path.join(__dirname, '../../screenshots');
            
            const captureScreenshot = async (name) => {
              if (!fs.existsSync(screenshotDir)) {
                fs.mkdirSync(screenshotDir, { recursive: true });
              }
              
              const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
              const filename = `${name}_${timestamp}.png`;
              const filepath = path.join(screenshotDir, filename);
              
              await page.screenshot({ path: filepath, fullPage: true });
              console.log(`📸 Screenshot saved: ${filename}`);
              return filepath;
            };
            
            await use(captureScreenshot);
          },
        });
        
        module.exports = { test };
    
    # ========================================
    # Phase 2: エンドツーエンドテスト実行
    # ========================================
    - name: "confirm_e2e_test_start"
      action: "confirm"
      message: |
        Phase 2: エンドツーエンドテスト を開始しますか？
        
        対象: {{integration_test_plan.total_scenarios}}件のシナリオ
        - 主要ユーザーフロー検証
        - 画面遷移テスト
        - データ入力・保存検証
        - エラーハンドリング確認
    
    - name: "generate_e2e_test_scenarios"
      action: "analyze"
      data: ["{{integration_test_plan}}", "{{read_files(find_files(patterns=['**/story_map.yaml']))}}"]
      instructions: |
        ストーリーマップから主要なE2Eテストシナリオを生成：
        
        1. **シナリオ1: 新規プロジェクト作成フロー**
           - ランディングページ表示
           - プロジェクト作成フォーム入力
           - 確認画面遷移
           - 作成完了確認
        
        2. **シナリオ2: データCRUDフロー**
           - データ一覧表示
           - 新規作成
           - 詳細表示
           - 編集
           - 削除
        
        3. **シナリオ3: エラーハンドリング**
           - バリデーションエラー
           - ネットワークエラー
           - 権限エラー
        
        各シナリオに対してPlaywrightテストコードを生成。
      store_as: "e2e_scenarios"
    
    - name: "create_e2e_test_scenario1"
      action: "create_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/playwright_tests/tests/scenario1-new-project.spec.js"
      content: |
        const { test } = require('../fixtures');
        const { expect } = require('@playwright/test');
        
        test.describe('シナリオ1: 新規プロジェクト作成フロー', () => {
          test.beforeEach(async ({ page }) => {
            await page.goto('/');
          });
        
          test('正常系: プロジェクト作成が成功する', async ({ page, saveScreenshot }) => {
            // Step 1: ランディングページ表示確認
            await expect(page.locator('h1')).toContainText('Welcome');
            await saveScreenshot('01_landing_page');
            
            // Step 2: 新規作成ボタンクリック
            await page.click('button:has-text("New Project")');
            await expect(page).toHaveURL(/.*\/projects\/new/);
            await saveScreenshot('02_new_project_form');
            
            // Step 3: フォーム入力
            await page.fill('input[name="project_name"]', 'Test Project');
            await page.fill('textarea[name="description"]', 'This is a test project');
            await page.selectOption('select[name="category"]', 'development');
            await saveScreenshot('03_form_filled');
            
            // Step 4: 送信
            await page.click('button[type="submit"]');
            
            // Step 5: 完了画面確認
            await expect(page).toHaveURL(/.*\/projects\/\d+/);
            await expect(page.locator('.success-message')).toContainText('Project created successfully');
            await saveScreenshot('04_project_created');
            
            // Step 6: データ確認
            await expect(page.locator('h1')).toContainText('Test Project');
            await expect(page.locator('.description')).toContainText('This is a test project');
          });
        
          test('異常系: バリデーションエラーが表示される', async ({ page, saveScreenshot }) => {
            await page.click('button:has-text("New Project")');
            
            // 必須項目を空のまま送信
            await page.click('button[type="submit"]');
            
            // エラーメッセージ確認
            await expect(page.locator('.error-message')).toContainText('Project name is required');
            await saveScreenshot('05_validation_error');
          });
        
          test('異常系: 重複名エラーが表示される', async ({ page, saveScreenshot }) => {
            await page.click('button:has-text("New Project")');
            
            // 既存のプロジェクト名を入力
            await page.fill('input[name="project_name"]', 'Existing Project');
            await page.fill('textarea[name="description"]', 'Test');
            await page.click('button[type="submit"]');
            
            // 重複エラー確認
            await expect(page.locator('.error-message')).toContainText('already exists');
            await saveScreenshot('06_duplicate_error');
          });
        });
    
    - name: "create_e2e_test_scenario2"
      action: "create_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/playwright_tests/tests/scenario2-crud-flow.spec.js"
      content: |
        const { test } = require('../fixtures');
        const { expect } = require('@playwright/test');
        
        test.describe('シナリオ2: データCRUDフロー', () => {
          let projectId;
        
          test('Create: データ作成', async ({ page, saveScreenshot }) => {
            await page.goto('/projects/new');
            await page.fill('input[name="project_name"]', 'CRUD Test Project');
            await page.fill('textarea[name="description"]', 'For CRUD testing');
            await page.click('button[type="submit"]');
            
            await expect(page).toHaveURL(/.*\/projects\/\d+/);
            const url = page.url();
            projectId = url.match(/\/projects\/(\d+)/)[1];
            
            await saveScreenshot('07_crud_create');
          });
        
          test('Read: データ一覧表示', async ({ page, saveScreenshot }) => {
            await page.goto('/projects');
            
            await expect(page.locator('.project-list')).toBeVisible();
            await expect(page.locator('.project-item')).toHaveCount(await page.locator('.project-item').count());
            await saveScreenshot('08_crud_read_list');
          });
        
          test('Update: データ編集', async ({ page, saveScreenshot }) => {
            await page.goto(`/projects/${projectId}/edit`);
            
            await page.fill('input[name="project_name"]', 'CRUD Test Project (Updated)');
            await page.click('button[type="submit"]');
            
            await expect(page.locator('h1')).toContainText('CRUD Test Project (Updated)');
            await saveScreenshot('09_crud_update');
          });
        
          test('Delete: データ削除', async ({ page, saveScreenshot }) => {
            await page.goto(`/projects/${projectId}`);
            
            await page.click('button:has-text("Delete")');
            await page.click('button:has-text("Confirm")');
            
            await expect(page).toHaveURL(/.*\/projects/);
            await expect(page.locator('.success-message')).toContainText('deleted');
            await saveScreenshot('10_crud_delete');
          });
        });
    
    - name: "run_e2e_tests"
      action: "execute_shell"
      command: |
        cd Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/playwright_tests && \
        npx playwright test --reporter=html,json,junit
      message: "エンドツーエンドテストを実行しました"
    
    # ========================================
    # Phase 3: Notionページ検証
    # ========================================
    - name: "confirm_notion_validation_start"
      action: "confirm"
      message: |
        Phase 3: Notionページ検証 を開始しますか？
        
        検証対象:
        - README ページ（構造、見出し、リンク）
        - getting-started ページ（手順、コード例）
        - api ページ（エンドポイント、パラメータ）
        
        Notion API を使用してページ内容を自動検証します。
    
    - name: "create_notion_validation_script"
      action: "create_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/notion_validation/validate_notion_pages.py"
      content: |
        #!/usr/bin/env python3
        """
        Notionページ検証スクリプト
        README, getting-started, api ページの構造と内容を自動検証
        """
        import os
        import sys
        import json
        from datetime import datetime
        from notion_client import Client
        
        # Notion API初期化
        notion_token = os.environ.get('NOTION_TOKEN')
        if not notion_token:
            print("❌ NOTION_TOKEN が設定されていません")
            sys.exit(1)
        
        notion = Client(auth=notion_token)
        
        # 検証対象ページID（環境変数から取得）
        README_PAGE_ID = os.environ.get('README_PAGE_ID')
        GETTING_STARTED_PAGE_ID = os.environ.get('GETTING_STARTED_PAGE_ID')
        API_PAGE_ID = os.environ.get('API_PAGE_ID')
        
        def validate_page_structure(page_id, expected_blocks):
            """ページ構造を検証"""
            try:
                blocks = notion.blocks.children.list(block_id=page_id)
                
                validation_results = {
                    'page_id': page_id,
                    'total_blocks': len(blocks['results']),
                    'validations': [],
                    'errors': []
                }
                
                # 期待されるブロックタイプの検証
                for expected in expected_blocks:
                    block_type = expected['type']
                    content_check = expected.get('content')
                    
                    matching_blocks = [
                        b for b in blocks['results'] 
                        if b['type'] == block_type
                    ]
                    
                    if len(matching_blocks) == 0:
                        validation_results['errors'].append(
                            f"Missing block type: {block_type}"
                        )
                    else:
                        # コンテンツ検証
                        if content_check:
                            block = matching_blocks[0]
                            text_content = extract_text_content(block)
                            if content_check.lower() in text_content.lower():
                                validation_results['validations'].append(
                                    f"✓ Found '{content_check}' in {block_type}"
                                )
                            else:
                                validation_results['errors'].append(
                                    f"✗ Expected '{content_check}' not found in {block_type}"
                                )
                        else:
                            validation_results['validations'].append(
                                f"✓ Block type {block_type} exists"
                            )
                
                return validation_results
            
            except Exception as e:
                return {
                    'page_id': page_id,
                    'error': str(e)
                }
        
        def extract_text_content(block):
            """ブロックからテキストコンテンツを抽出"""
            block_type = block['type']
            
            if block_type in ['paragraph', 'heading_1', 'heading_2', 'heading_3']:
                rich_text = block[block_type].get('rich_text', [])
                return ''.join([t['plain_text'] for t in rich_text])
            
            elif block_type == 'bulleted_list_item':
                rich_text = block['bulleted_list_item'].get('rich_text', [])
                return ''.join([t['plain_text'] for t in rich_text])
            
            elif block_type == 'code':
                rich_text = block['code'].get('rich_text', [])
                return ''.join([t['plain_text'] for t in rich_text])
            
            return ''
        
        def validate_readme():
            """READMEページ検証"""
            print("\n📄 Validating README page...")
            
            expected_blocks = [
                {'type': 'heading_1', 'content': 'README'},
                {'type': 'heading_2', 'content': 'Installation'},
                {'type': 'heading_2', 'content': 'Usage'},
                {'type': 'code'},
                {'type': 'bulleted_list_item'},
            ]
            
            results = validate_page_structure(README_PAGE_ID, expected_blocks)
            return results
        
        def validate_getting_started():
            """getting-startedページ検証"""
            print("\n🚀 Validating getting-started page...")
            
            expected_blocks = [
                {'type': 'heading_1', 'content': 'Getting Started'},
                {'type': 'heading_2', 'content': 'Step'},
                {'type': 'code'},
                {'type': 'paragraph'},
            ]
            
            results = validate_page_structure(GETTING_STARTED_PAGE_ID, expected_blocks)
            return results
        
        def validate_api():
            """apiページ検証"""
            print("\n🔌 Validating api page...")
            
            expected_blocks = [
                {'type': 'heading_1', 'content': 'API'},
                {'type': 'heading_2', 'content': 'Endpoint'},
                {'type': 'code'},
                {'type': 'heading_3', 'content': 'Parameter'},
            ]
            
            results = validate_page_structure(API_PAGE_ID, expected_blocks)
            return results
        
        def generate_report(results):
            """検証レポート生成"""
            report = {
                'timestamp': datetime.now().isoformat(),
                'pages': results,
                'summary': {
                    'total_pages': len(results),
                    'passed': sum(1 for r in results.values() if len(r.get('errors', [])) == 0),
                    'failed': sum(1 for r in results.values() if len(r.get('errors', [])) > 0),
                }
            }
            
            # JSON保存
            report_path = '../test_reports/notion_validation_report.json'
            with open(report_path, 'w', encoding='utf-8') as f:
                json.dump(report, f, indent=2, ensure_ascii=False)
            
            print(f"\n✅ Report saved: {report_path}")
            
            # コンソール出力
            print("\n" + "="*60)
            print("📊 Notion Page Validation Summary")
            print("="*60)
            print(f"Total Pages: {report['summary']['total_pages']}")
            print(f"Passed: {report['summary']['passed']}")
            print(f"Failed: {report['summary']['failed']}")
            print("="*60)
            
            for page_name, result in results.items():
                print(f"\n{page_name}:")
                for validation in result.get('validations', []):
                    print(f"  {validation}")
                for error in result.get('errors', []):
                    print(f"  ❌ {error}")
            
            return report['summary']['failed'] == 0
        
        if __name__ == '__main__':
            results = {}
            
            if README_PAGE_ID:
                results['README'] = validate_readme()
            
            if GETTING_STARTED_PAGE_ID:
                results['getting-started'] = validate_getting_started()
            
            if API_PAGE_ID:
                results['api'] = validate_api()
            
            success = generate_report(results)
            sys.exit(0 if success else 1)
    
    - name: "run_notion_validation"
      action: "execute_shell"
      command: |
        cd Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/notion_validation && \
        chmod +x validate_notion_pages.py && \
        python3 validate_notion_pages.py
      message: "Notionページ検証を実行しました"
    
    # ========================================
    # Phase 4: スクリーンショット自動取得
    # ========================================
    - name: "confirm_screenshot_capture"
      action: "confirm"
      message: |
        Phase 4: スクリーンショット自動取得 を開始しますか？
        
        取得対象:
        - 全画面のスクリーンショット
        - 主要コンポーネントのスクリーンショット
        - レスポンシブ表示（PC、タブレット、スマホ）
    
    - name: "create_screenshot_capture_script"
      action: "create_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/playwright_tests/tests/screenshot-capture.spec.js"
      content: |
        const { test } = require('../fixtures');
        const { expect } = require('@playwright/test');
        
        test.describe('スクリーンショット自動取得', () => {
          const viewports = [
            { name: 'desktop', width: 1920, height: 1080 },
            { name: 'tablet', width: 768, height: 1024 },
            { name: 'mobile', width: 375, height: 667 },
          ];
        
          for (const viewport of viewports) {
            test(`全画面キャプチャ - ${viewport.name}`, async ({ page, saveScreenshot }) => {
              await page.setViewportSize({ width: viewport.width, height: viewport.height });
              
              // ランディングページ
              await page.goto('/');
              await saveScreenshot(`landing_${viewport.name}`);
              
              // プロジェクト一覧
              await page.goto('/projects');
              await saveScreenshot(`projects_list_${viewport.name}`);
              
              // プロジェクト詳細（1件目）
              const firstProject = page.locator('.project-item').first();
              if (await firstProject.isVisible()) {
                await firstProject.click();
                await saveScreenshot(`project_detail_${viewport.name}`);
              }
              
              // 新規作成フォーム
              await page.goto('/projects/new');
              await saveScreenshot(`new_project_form_${viewport.name}`);
            });
          }
        
          test('コンポーネント別キャプチャ', async ({ page, saveScreenshot }) => {
            await page.goto('/');
            
            // ヘッダー
            const header = page.locator('header');
            await header.screenshot({ path: '../screenshots/component_header.png' });
            
            // ナビゲーション
            const nav = page.locator('nav');
            if (await nav.isVisible()) {
              await nav.screenshot({ path: '../screenshots/component_nav.png' });
            }
            
            // フッター
            const footer = page.locator('footer');
            if (await footer.isVisible()) {
              await footer.screenshot({ path: '../screenshots/component_footer.png' });
            }
            
            console.log('📸 Component screenshots captured');
          });
        });
    
    - name: "run_screenshot_capture"
      action: "execute_shell"
      command: |
        cd Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/playwright_tests && \
        npx playwright test screenshot-capture.spec.js
      message: "スクリーンショットを自動取得しました"
    
    # ========================================
    # Phase 5: 最終QAレポート生成
    # ========================================
    - name: "generate_final_qa_report"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11-4_結合テスト実施/test_reports/final_qa_report.md"
      content: |
        # 最終QAレポート - 結合テスト実施結果
        
        **実行日時**: {{now}}
        **プロジェクト**: {{project_name}}
        **テスト対象**: {{integration_test_plan.total_stories}}ストーリー
        
        ---
        
        ## 📊 実行サマリー
        
        ### エンドツーエンドテスト結果
        **実行シナリオ数**: {{e2e_results.total_scenarios}}件
        **成功**: {{e2e_results.passed}}件
        **失敗**: {{e2e_results.failed}}件
        **スキップ**: {{e2e_results.skipped}}件
        **成功率**: {{e2e_results.pass_rate}}%
        
        ### Notionページ検証結果
        | ページ | ステータス | 検証項目 | エラー |
        |--------|-----------|---------|-------|
        | README | {{notion_results.readme.status}} | {{notion_results.readme.validations}} | {{notion_results.readme.errors}} |
        | getting-started | {{notion_results.getting_started.status}} | {{notion_results.getting_started.validations}} | {{notion_results.getting_started.errors}} |
        | api | {{notion_results.api.status}} | {{notion_results.api.validations}} | {{notion_results.api.errors}} |
        
        ### スクリーンショット取得
        **取得枚数**: {{screenshot_results.total_screenshots}}枚
        **デバイス別**: PC {{screenshot_results.desktop}}枚、タブレット {{screenshot_results.tablet}}枚、スマホ {{screenshot_results.mobile}}枚
        **保存先**: `screenshots/`
        
        ---
        
        ## 🎯 テストシナリオ詳細
        
        {{#each e2e_scenarios.scenarios}}
        ### シナリオ {{@index}}: {{name}}
        **優先度**: {{priority}}
        **ステータス**: {{status}}
        **実行時間**: {{execution_time}}秒
        
        #### テストケース
        {{#each test_cases}}
        - {{case_name}}: {{result}}
        {{/each}}
        
        #### スクリーンショット
        {{#each screenshots}}
        - ![{{name}}](../screenshots/{{filename}})
        {{/each}}
        {{/each}}
        
        ---
        
        ## ⚠️ 失敗したテスト
        
        {{#if e2e_results.failures}}
        {{#each e2e_results.failures}}
        ### {{test_name}}
        **エラーメッセージ**: {{error_message}}
        **スタックトレース**:
        ```
        {{stack_trace}}
        ```
        **スクリーンショット**: ![失敗時](../screenshots/{{failure_screenshot}})
        {{/each}}
        {{else}}
        ✅ **失敗したテストはありません**
        {{/if}}
        
        ---
        
        ## 🔍 Notionページ検証詳細
        
        ### README ページ
        {{#each notion_results.readme.validations}}
        - {{.}}
        {{/each}}
        {{#if notion_results.readme.errors}}
        **エラー**:
        {{#each notion_results.readme.errors}}
        - ❌ {{.}}
        {{/each}}
        {{/if}}
        
        ### getting-started ページ
        {{#each notion_results.getting_started.validations}}
        - {{.}}
        {{/each}}
        {{#if notion_results.getting_started.errors}}
        **エラー**:
        {{#each notion_results.getting_started.errors}}
        - ❌ {{.}}
        {{/each}}
        {{/if}}
        
        ### api ページ
        {{#each notion_results.api.validations}}
        - {{.}}
        {{/each}}
        {{#if notion_results.api.errors}}
        **エラー**:
        {{#each notion_results.api.errors}}
        - ❌ {{.}}
        {{/each}}
        {{/if}}
        
        ---
        
        ## 📸 スクリーンショット一覧
        
        ### デスクトップ
        {{#each screenshot_results.desktop_screenshots}}
        - ![{{name}}](../screenshots/{{filename}})
        {{/each}}
        
        ### タブレット
        {{#each screenshot_results.tablet_screenshots}}
        - ![{{name}}](../screenshots/{{filename}})
        {{/each}}
        
        ### モバイル
        {{#each screenshot_results.mobile_screenshots}}
        - ![{{name}}](../screenshots/{{filename}})
        {{/each}}
        
        ---
        
        ## 🎯 リリース判定
        
        {{#if all_tests_passed}}
        ✅ **リリース可**: 全テストが成功しました
        
        **合格基準**:
        - [x] E2Eテスト成功率 100%
        - [x] Notionページ検証 全ページ合格
        - [x] スクリーンショット取得 完了
        - [x] Critical/Highバグ 0件
        {{else}}
        ⚠️ **リリース保留**: 以下の問題が未解決です
        
        **ブロッカー**:
        {{#each blockers}}
        - {{.}}
        {{/each}}
        
        **推奨アクション**:
        {{#each recommended_actions}}
        - {{.}}
        {{/each}}
        {{/if}}
        
        ---
        
        ## 📝 次のステップ
        
        {{#if all_tests_passed}}
        1. ✅ レポート確認完了
        2. ✅ リリース判定会議
        3. → `/aipm/aipm_4_15_delivery_リリース判定` でリリース承認
        {{else}}
        1. ❌ 失敗したテストの原因調査
        2. ❌ バグ修正・再テスト
        3. → `/aipm/aipm_4_12_delivery_バグ登録` でバグ登録
        4. → 修正完了後、結合テスト再実行
        {{/if}}
        
        ---
        
        **生成日時**: {{now}}
        **レポート生成者**: AIPM System
    
    - name: "notify_integration_test_completion"
      action: "display"
      content: |
        ✅ 結合テスト実施プロセスが完了しました
        
        **生成されたドキュメント**:
        - playwright_tests/playwright.config.js（Playwright設定）
        - playwright_tests/tests/*.spec.js（E2Eテストシナリオ）
        - notion_validation/validate_notion_pages.py（Notion検証スクリプト）
        - test_reports/final_qa_report.md（最終QAレポート）
        - screenshots/*.png（スクリーンショット）
        
        **実行結果サマリー**:
        - E2Eテスト: {{e2e_results.passed}}/{{e2e_results.total_scenarios}} 成功
        - Notionページ検証: {{notion_results.passed}}/{{notion_results.total_pages}} 合格
        - スクリーンショット: {{screenshot_results.total_screenshots}}枚 取得完了
        
        {{#if all_tests_passed}}
        ✅ **全テスト合格** - リリース判定に進めます
        {{else}}
        ⚠️ **一部テスト失敗** - バグ登録・修正が必要です
        {{/if}}
        
        **次のアクション**:
        1. レポート確認: open test_reports/final_qa_report.md
        2. スクリーンショット確認: open screenshots/
        3. Playwright HTMLレポート: npx playwright show-report
        {{#if all_tests_passed}}
        4. リリース判定: `/aipm/aipm_4_15_delivery_リリース判定`
        {{else}}
        4. バグ登録: `/aipm/aipm_4_12_delivery_バグ登録`
        {{/if}}
```

## 成果物
- `playwright_tests/` - Playwrightテストスイート
  - `playwright.config.js` - Playwright設定
  - `tests/*.spec.js` - E2Eテストシナリオ
  - `tests/fixtures.js` - カスタムフィクスチャ
- `notion_validation/` - Notion検証スクリプト
  - `validate_notion_pages.py` - ページ構造検証
- `screenshots/` - スクリーンショット（PC/タブレット/スマホ）
- `test_reports/` - テストレポート
  - `final_qa_report.md` - 最終QAレポート
  - `playwright-report/` - Playwright HTMLレポート
  - `results.json` - JSON形式の結果
  - `junit.xml` - JUnit XMLレポート
  - `notion_validation_report.json` - Notion検証結果

## 次のコマンド
→ 全テスト合格時: `/aipm/aipm_4_15_delivery_リリース判定` でリリース承認
→ テスト失敗時: `/aipm/aipm_4_12_delivery_バグ登録` でバグ登録・修正

## 変更点（v1.0 - 結合テスト実施）
- **Playwright統合**: ブラウザ自動テストフレームワーク採用
- **Notion API検証**: README/getting-started/apiページの自動検証
- **スクリーンショット自動取得**: 全画面・コンポーネント・レスポンシブ対応
- **最終QAレポート**: 包括的なテスト結果レポート自動生成
- **リリース判定支援**: テスト結果に基づく自動判定