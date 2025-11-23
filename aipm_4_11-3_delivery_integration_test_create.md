# 11-3 Delivery Integration Test Create
name: 結合テスト作成（ハイブリッド統合テスト実装）
---

# 11-3 : 結合テスト作成 - 実用的なハイブリッド統合テスト構築

## 🎯 URL統一システムとPlaywright学習ナレッジ

### ✅ URL統一システムの成果
- **統一URL解決**: `default_parent_url`を唯一のソースとして使用
- **シンプルな優先順位**: `config.json` → `NOTION_ROOT_URL` の2段階のみ
- **自動移行サポート**: 既存プロジェクトの自動移行スクリプト
- **レガシー互換性**: 既存設定との互換性を保持

### ✅ Playwright成功パターン
- **ハイブリッドテスト**: Cursor コマンド（CLI実行） + Playwright（UI検証）を交互に実行
- **統一URL解決**: `config.json`の`default_parent_url`からURLを取得
- **シンプルなセレクタ**: `page.getByRole`, `page.getByText` を使用
- **認証処理**: Google SSOの「Daisukeとして実行」ボタンクリック
- **ページ読み込み**: `domcontentloaded` + `waitForTimeout` の組み合わせ

### 🔧 実証済み実装パターン
1. **URL解決**: `config.json`の`default_parent_url`を優先使用
2. **認証**: 複数セレクタで「Daisukeとして実行」ボタンを確実にクリック
3. **セレクタ**: `page.getByRole('heading', { level: 1 })` など、意味的なセレクタを優先
4. **環境変数**: `TEST_PAGE_URL`, `ROOT_PAGE_URL`, `README_PAGE_URL`の複数フォールバック
5. **キャッシュ管理**: テスト前の明示的なキャッシュクリア

---

## 🎯 Playwrightセレクター戦略の完全ガイド（2025-10-16更新）

### ✅ UX検証スクリプトから学ぶ成功パターン

#### 1. **複数セレクターのフォールバック戦略**

**問題**: Notionの動的なDOM構造により、単一のセレクターでは要素が見つからないことがある

**解決策**: UX検証スクリプトの成功パターンを採用

```typescript
// ❌ 失敗パターン: 単一セレクターに依存
const documentLink = page.getByRole('link', { name: /^document$/i });
await documentLink.click(); // → 失敗する可能性が高い

// ✅ 成功パターン: 複数セレクターを順次試行
const documentSelectors = [
  'a:has-text("document")',           // パターン1: テキストを含むリンク
  'a[href*="document"]',              // パターン2: href属性にdocumentを含む
  '[role="link"]:has-text("document")', // パターン3: role属性付き
  'div:has-text("document")'          // パターン4: リンクでなくてもOK
];

let documentLink = null;
for (const selector of documentSelectors) {
  const locator = page.locator(selector).first();
  const count = await locator.count();
  if (count > 0) {
    documentLink = locator;
    console.log(`✓ 要素が見つかりました (selector: ${selector})`);
    break;
  }
}

if (documentLink) {
  await documentLink.click();
} else {
  throw new Error('要素が見つかりませんでした');
}
```

#### 2. **見出しセレクターの柔軟な戦略**

**問題**: `getByRole('heading', { level: 2 })`では日本語の見出しが見つからない

**解決策**: 複数セレクターとテキストベース検索の組み合わせ

```typescript
// ❌ 失敗パターン
const section = page.getByRole('heading', { level: 2, name: /概要/i });
await expect(section).toBeVisible(); // → 失敗

// ✅ 成功パターン: 複数セレクターで柔軟に対応
const sections = ['概要', '目的', 'アーキテクチャ'];

for (const sectionName of sections) {
  const selectors = [
    `h2:has-text("${sectionName}")`,              // パターン1: h2タグ
    `[role="heading"]:has-text("${sectionName}")`, // パターン2: role属性
    `:text("${sectionName}")`                      // パターン3: テキストのみ
  ];
  
  let found = false;
  for (const selector of selectors) {
    const locator = page.locator(selector);
    const count = await locator.count();
    if (count > 0) {
      found = true;
      console.log(`✓ ${sectionName}が見つかりました (selector: ${selector})`);
      break;
    }
  }
  
  expect(found).toBe(true);
}
```

#### 3. **Strict Mode Violation対策**

**問題**: 正規表現が複数要素にマッチして`strict mode violation`エラー

**解決策**: `.first()`を明示的に使用

```typescript
// ❌ 失敗パターン
const content = page.getByText(/以下の目的を達成|拡張性を確保/);
await expect(content).toBeVisible(); // → Strict mode violation

// ✅ 成功パターン: .first()で最初の要素のみ取得
const content = page.getByText(/以下の目的を達成|拡張性を確保/).first();
await expect(content).toBeVisible(); // → 成功
```

#### 4. **Notionリストアイテムのセレクター**

**問題**: `getByRole('listitem')`ではNotionのリストが見つからない

**解決策**: Notion固有のクラス名を含む複数セレクター

```typescript
// ❌ 失敗パターン
const listItems = page.getByRole('listitem');
const count = await listItems.count(); // → 0

// ✅ 成功パターン: Notion固有のセレクターを含める
const listSelectors = [
  '[role="listitem"]',
  'ul > li',
  'ol > li',
  '.notion-bulleted_list-block',      // ★Notion固有
  '.notion-numbered_list-block'       // ★Notion固有
];

let listCount = 0;
for (const selector of listSelectors) {
  const items = page.locator(selector);
  const count = await items.count();
  if (count > 0) {
    listCount = count;
    console.log(`✓ ${listCount}個のリストアイテム (selector: ${selector})`);
    break;
  }
}
```

#### 5. **Playwright実行オプションの正しい使い方**

**問題**: `--project=chromium`を指定すると、毎回`setup`プロジェクトが実行される

**解決策**: UX検証スクリプトと同じく、`--headed`のみを指定

```bash
# ❌ 失敗パターン: 毎回認証が実行される
npx playwright test tests/verify.spec.ts --headed --project=chromium

# ✅ 成功パターン: 既存の認証を利用
PROJECT_URL="$PROJECT_URL" npx playwright test tests/verify.spec.ts --headed
```

**playwright.config.ts設定**:
```typescript
projects: [
  {
    name: 'setup',
    testMatch: /.*\.setup\.ts/,
  },
  {
    name: 'chromium',
    use: { 
      storageState: '.auth/auth.json',
    },
    dependencies: ['setup'], // ★明示的に依存関係を定義
  },
]
```

### 📋 セレクター戦略チェックリスト

#### テスト作成時
- [ ] 単一セレクターではなく、複数セレクターのフォールバック戦略を使用
- [ ] UX検証スクリプト（`execute_ux_walkthrough.sh`）の成功パターンを参照
- [ ] Notion固有のクラス名（`.notion-*`）を調査して含める
- [ ] 正規表現を使う場合は`.first()`を追加

#### テスト実行時
- [ ] `--project`オプションは指定しない（または慎重に使用）
- [ ] 環境変数（`PROJECT_URL`等）を明示的にエクスポート
- [ ] `--headed`モードで視覚的に確認

#### トラブルシューティング
- [ ] 失敗したセレクターをブラウザDevToolsで検証
- [ ] UX検証スクリプトの該当部分を参照
- [ ] 複数セレクターのループで、どのセレクターが成功したかログ出力

### 🔧 実装テンプレート

```typescript
// 複数セレクター戦略の汎用テンプレート
async function findElementWithFallback(
  page: Page,
  selectors: string[],
  elementName: string
): Promise<Locator | null> {
  for (const selector of selectors) {
    const locator = page.locator(selector).first();
    const count = await locator.count();
    if (count > 0) {
      console.log(`✓ ${elementName}が見つかりました (selector: ${selector})`);
      return locator;
    }
  }
  
  console.log(`⚠️ ${elementName}が見つかりません（全セレクター試行済み）`);
  return null;
}

// 使用例
const documentLink = await findElementWithFallback(
  page,
  [
    'a:has-text("document")',
    'a[href*="document"]',
    '[role="link"]:has-text("document")',
    'div:has-text("document")'
  ],
  'documentリンク'
);

if (documentLink) {
  await documentLink.click();
} else {
  throw new Error('documentリンクが見つかりませんでした');
}
```

---

## 🔐 Playwright認証の完全ガイド（重要）

### 📊 storageStateの仕組み（確認された事実）

#### 1. **保存先と読み込み**
```typescript
// 保存：指定したファイルパスに直接保存
await page.context().storageState({ path: 'auth.json' });
// → auth.jsonファイルが作成される

// 読み込み：明示的に指定が必要（自動では読み込まれない）
// playwright.config.ts
projects: [{
  use: { 
    storageState: 'auth.json' // ★明示的に指定が必須
  }
}]
```

**重要ポイント:**
- ✅ ファイルとして保存される（Playwrightの内部キャッシュではない）
- ✅ 読み込みは完全に手動（明示的に指定しない限り絶対に読み込まれない）
- ❌ 自動で読み込まれることは一切ない

#### 2. **有効期限**
```json
{
  "cookies": [{
    "name": "file_token",
    "expires": 1792059834.599433 // ← この期限まで有効
  }]
}
```

**重要ポイント:**
- ✅ ファイル自体は永続的に存在
- ✅ 実際の有効期限は各Cookieの`expires`フィールドに依存
- ❌ storageState自体に時間制限はない

#### 3. **ブラウザ再起動後のログイン状態**
```typescript
// playwright.config.ts
projects: [{
  use: { 
    storageState: authFilePath // ★ここで読み込んでいる
  }
}]
```

**実行フロー:**
```
1. npx playwright test
   ↓
2. playwright.config.tsを読み込む
   ↓
3. storageState: authFilePathを発見
   ↓
4. auth.jsonを読み込む
   ↓
5. ブラウザ起動時点で、auth.jsonのCookies/localStorageをセット
   ↓
6. テスト開始時点で、すでにログイン済み
```

**重要ポイント:**
- ✅ ブラウザ起動時に設定で読み込んでいる
- ❌ Playwrightの自動機能ではない
- ❌ セッションという概念はない（毎回ファイルから読み込む）

### 🐛 「時々うまくいき、時々失敗する」問題の真の原因

#### ❌ 間違った仮説
- 読み込み設定が間違っていた
- Playwrightのバグ
- Notionの仕様変更

#### ✅ 真の原因：`auth.json`の内容が不完全だった

**失敗パターン:**
```
1. auth.setup.ts実行
2. ページアクセス後、3秒で即座に保存
3. auth.json作成（29 cookies） ← 不完全！
4. 次回テスト実行
5. 不完全なauth.jsonを読み込む
6. → ポップアップが出る（失敗）
```

**成功パターン:**
```
1. auth.json削除
2. auth.setup.ts実行
3. ユーザーが手動で完全ログイン（60秒待機中）
4. auth.json作成（83 cookies） ← 完全！
5. 次回テスト実行
6. 完全なauth.jsonを読み込む
7. → ポップアップが出ない（成功）
```

### 🎯 正しい認証セットアップの実装

```typescript
import { test as setup } from '@playwright/test';
import path from 'path';
import fs from 'fs';

const authFile = path.join(__dirname, '../../.auth/notion.json');

setup('Notion認証', async ({ page }) => {
  setup.setTimeout(180000); // 3分に延長
  
  // 既存の認証ファイルがあればスキップ
  if (fs.existsSync(authFile)) {
    console.log('✅ 認証ファイルが既に存在します');
    return;
  }
  
  const testPageUrl = process.env.PROJECT_URL || process.env.TEST_PAGE_URL;
  if (!testPageUrl) {
    throw new Error('PROJECT_URL または TEST_PAGE_URL が設定されていません');
  }
  
  await page.goto(testPageUrl, { waitUntil: 'domcontentloaded' });
  await page.waitForTimeout(3000);
  
  // ポップアップ検出（自動化試行）
  const button = page.locator('button:has-text("Daisukeとして続行")');
  const buttonFound = await button.isVisible({ timeout: 3000 }).catch(() => false);
  
  if (buttonFound) {
    await button.click();
    console.log('✅ 自動クリック成功');
    await page.waitForTimeout(5000);
  } else {
    // 手動操作を促す
    console.log('━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━');
    console.log('👤 手動操作が必要です');
    console.log('━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━');
    console.log('');
    console.log('以下の手順でログインしてください：');
    console.log('1. Googleでログイン');
    console.log('2. Googleアカウントを選択');
    console.log('3. 「Daisukeとして続行」をクリック（あれば）');
    console.log('4. Notionページが完全に表示されるまで待つ');
    console.log('');
    console.log('⏳ 2分間待機します...');
    console.log('━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━');
    
    // ★重要：ユーザーが完全にログインする時間を確保
    await page.waitForTimeout(120000); // 2分
  }
  
  // ★重要：ページが完全に安定化してから保存
  await page.waitForLoadState('domcontentloaded');
  await page.waitForTimeout(3000); // 追加の安定化待機
  
  // storageState保存
  await page.context().storageState({ path: authFile });
  console.log('✅ 認証情報を保存しました');
  
  // ★品質チェック：ファイルサイズ確認（推奨）
  const stats = fs.statSync(authFile);
  const sizeMB = (stats.size / 1024 / 1024).toFixed(2);
  
  if (stats.size < 1000000) { // 1MB未満
    console.warn(`⚠️ 認証ファイルが小さすぎます（${sizeMB}MB）。ログインが不完全な可能性があります。`);
  } else {
    console.log(`✅ 認証状態を保存しました（${sizeMB}MB）`);
  }
});
```

### 📋 認証チェックリスト

#### セットアップ前
- [ ] `playwright.config.ts`で`storageState`を指定している
- [ ] `auth.setup.ts`と`playwright.config.ts`で同じパスを使用している
- [ ] 環境変数（`PROJECT_URL`等）が正しく設定されている

#### セットアップ中
- [ ] ユーザーが完全にログイン完了するまで待機している（60秒以上推奨）
- [ ] ページが安定化してから`storageState`を保存している
- [ ] 保存後にファイルサイズを確認している（1MB以上推奨）

#### セットアップ後
- [ ] `auth.json`ファイルが存在する
- [ ] ファイルサイズが1MB以上ある
- [ ] 次回テスト実行時、ポップアップが出ない

### 🔧 トラブルシューティング

#### 問題: ポップアップが毎回出る
**原因:** `auth.json`の内容が不完全
**対処:**
```bash
# 1. auth.jsonを削除
rm -f .auth/notion.json

# 2. 再度認証セットアップ（手動ログイン時間を十分に取る）
npx playwright test tests/auth.setup.ts --headed

# 3. ファイルサイズ確認
ls -lh .auth/notion.json
# → 1.5MB以上あるべき
```

#### 問題: 認証ファイルが読み込まれない
**原因:** `playwright.config.ts`で`storageState`を指定していない
**対処:**
```typescript
// playwright.config.ts
projects: [{
  name: 'chromium',
  use: { 
    storageState: '.auth/notion.json' // ★明示的に指定
  },
}]
```

#### 問題: ファイルサイズが小さい（1MB未満）
**原因:** ログイン完了前に保存している
**対処:**
```typescript
// auth.setup.ts
// 手動操作の待機時間を延長
await page.waitForTimeout(120000); // 60秒 → 120秒

// ページ安定化の待機を追加
await page.waitForLoadState('domcontentloaded');
await page.waitForTimeout(5000); // 3秒 → 5秒
```

### 📚 参考ドキュメント（プロジェクト内）

詳細は以下のドキュメントを参照：
- `STORAGESTATE_MECHANISM.md` - storageStateの仕組み完全解説
- `STORAGESTATE_DEEP_ANALYSIS.md` - 実験結果と分析
- `ROOT_CAUSE_ANALYSIS.md` - 「時々失敗する」問題の根本原因
- `AUTHENTICATION_SUCCESS_ANALYSIS.md` - 認証成功の分析

---

---

## 前提
- 基本的な機能実装が完了済み
- テスト対象がCLIツールまたはWebアプリケーション
- **ハイブリッドテスト形式**: CLIコマンド実行とPlaywright検証を組み合わせる

## 目的
- 実際に動作するハイブリッド統合テストを作成
- Playwrightコードのバリデーションスクリプトを自動生成
- 再現可能で保守性の高いテストスイートを構築

---

## 実行手順

### Phase 0: テスト要件分析とプロジェクト理解

```yaml
- name: "analyze_project_structure"
      action: "execute_shell"
      command: |
    echo "📊 プロジェクト構造を分析中..."
    find . -type f -name "*.md" -path "*/documents/*" | head -20
    find . -type f -name "*.py" -o -name "*.js" -o -name "*.ts" | head -20
```

**AI指示**: 
- プロジェクトのREADME、仕様書、コードを確認
- テスト対象の機能を特定（CLI/Web/API）
- 主要なユーザーフロー4つを抽出
- 認証の有無を確認

---

### Phase 1: ハイブリッドテスト設計

```yaml
- name: "design_hybrid_test_scenarios"
      action: "create_markdown_file"
  path: "{{test_dir}}/TEST_DESIGN.md"
      content: |
    # ハイブリッド統合テスト設計
    
    ## テスト対象プロジェクト
    **名称**: {{project_name}}
    **タイプ**: {{project_type}}  # CLI / Web / API
    
    ## テストシナリオ
    
    ### Scenario 1: 基本フロー（ハッピーパス）
    **Phase構成**:
    1. Phase 1 (Cursor): 初期セットアップ（CLIコマンド実行）
    2. Phase 2 (Playwright): 初期状態の検証
    3. Phase 3 (Cursor): 主要機能の実行
    4. Phase 4 (Playwright): 結果の検証
    
    **成功基準**:
    - 全Phaseが正常完了
    - Playwright検証が100%パス
    
    ### Scenario 2: エラーハンドリング（競合解決など）
    **Phase構成**:
    1. Phase 1 (Cursor): 初期データ作成
    2. Phase 2 (Playwright): 初期同期確認
    3. Phase 3 (Cursor/Manual): データ編集（両側）
    4. Phase 4 (Cursor): Pull実行（競合検出）
    5. Phase 5 (Manual): 競合解決
    6. Phase 6 (Cursor): Push実行
    7. Phase 7 (Playwright): 最終状態確認
    
    **重要**: キャッシュクリアなど、環境依存の問題に対処
    
    ### Scenario 3: 大量データ処理
    **Phase構成**:
    1. Phase 1 (Cursor): テストデータ生成（100件など）
    2. Phase 2 (Cursor): バッチ処理実行
    3. Phase 3 (Playwright): 結果確認（一部サンプリング）
    
    ### Scenario 4: エッジケース
    **Phase構成**:
    1. Phase 1 (Cursor): 特殊条件のセットアップ
    2. Phase 2 (Cursor): 境界値テスト実行
    3. Phase 3 (Playwright): エラー表示確認
```

---

### Phase 2: Playwrightセットアップ（実用的な設定）

```yaml
- name: "generate_working_playwright_config"
  action: "create_file"
  path: "{{test_dir}}/playwright.config.ts"
      content: |
        import { defineConfig, devices } from '@playwright/test';
        
        export default defineConfig({
          testDir: './tests',
      fullyParallel: false,  // シーケンシャル実行（ハイブリッドテストのため）
      workers: 1,
      retries: 0,  // リトライなし（デバッグしやすくするため）
          
      timeout: 300000,  // 5分（手動ステップを含むため）
          
          reporter: [
        ['html', { open: 'never' }],  // 自動オープン無効（ターミナル復帰のため）
            ['list']
          ],
          
          use: {
        actionTimeout: 60000,  // 1分
        navigationTimeout: 60000,
            
            trace: 'on-first-retry',
            screenshot: 'only-on-failure',
            video: 'retain-on-failure',
          },
          
          projects: [
            {
              name: 'setup',
              testMatch: /.*\.setup\.ts/,
            },
            {
              name: 'chromium',
              use: { 
                ...devices['Desktop Chrome'],
            storageState: '.auth/auth.json',
              },
              dependencies: ['setup'],
        },
      ],
    });
```

---

### Phase 3: 認証セットアップ（実際に動くコード）

```yaml
- name: "generate_working_auth_setup"
  action: "create_file"
  path: "{{test_dir}}/tests/auth.setup.ts"
      content: |
    import { test as setup } from '@playwright/test';
        import path from 'path';
        
    const authFile = path.join(__dirname, '../.auth/auth.json');
        
        setup('authenticate', async ({ page }) => {
          console.log('🔐 認証セットアップ開始...');
          
      // 統一URL解決システム: 複数環境変数からフォールバック
      const testPageUrl = process.env.TEST_PAGE_URL || 
                         process.env.ROOT_PAGE_URL || 
                         process.env.README_PAGE_URL;
      if (!testPageUrl) {
        throw new Error('TEST_PAGE_URL, ROOT_PAGE_URL, or README_PAGE_URL が設定されていません');
      }
      
      // テストページに直接アクセス
      await page.goto(testPageUrl, { waitUntil: 'domcontentloaded' });
      await page.waitForTimeout(3000);
      
      console.log('📄 ページを読み込みました');
      
      // Google SSO「Daisukeとして実行」ボタンの処理（実証済みパターン）
      try {
        // 複数セレクタで確実にボタンを検出
        const runAsButton = page.getByText(/Daisukeとして実行|として実行|Continue as Daisuke/i)
                               .or(page.locator('[data-testid*=\"run-as\"]'))
                               .or(page.locator('button:has-text(\"Daisuke\")'));
        
        const isVisible = await runAsButton.isVisible({ timeout: 10000 });
        if (isVisible) {
          await runAsButton.click();
          console.log('✓ 「Daisukeとして実行」ボタンをクリックしました');
          await page.waitForTimeout(3000);
        }
      } catch (e) {
        console.log('ℹ️ 「Daisukeとして実行」ボタンは表示されませんでした');
      }
          
          // 認証状態を保存
          await page.context().storageState({ path: authFile });
      console.log('✅ 認証状態を保存しました');
    });
```

---

### Phase 4: シナリオ1実装（実証済みパターン）

```yaml
    - name: "generate_scenario1_spec"
  action: "create_file"
  path: "{{test_dir}}/tests/scenario1-basic-flow.spec.ts"
      content: |
        import { test, expect } from '@playwright/test';
    
    /**
     * Scenario 1: 基本フロー（ハッピーパス）
     * 
     * このテストは Playwright のみで完結します。
     * CLIコマンドは事前に手動またはシェルスクリプトで実行済みであることを前提とします。
     */
    
    test.describe('Scenario 1: 基本フロー検証', () => {
      test('README.md が正しくレンダリングされている', async ({ page }) => {
        // 統一URL解決システム: 複数環境変数からフォールバック
        const readmeUrl = process.env.README_PAGE_URL || 
                         process.env.TEST_PAGE_URL || 
                         process.env.ROOT_PAGE_URL;
        if (!readmeUrl) {
          throw new Error('README_PAGE_URL, TEST_PAGE_URL, or ROOT_PAGE_URL が設定されていません');
        }
        
        console.log(`📄 ページを開いています: ${readmeUrl}`);
        await page.goto(readmeUrl, { waitUntil: 'domcontentloaded' });
        await page.waitForTimeout(2000);
        
        // タイトル確認
        console.log('📝 タイトルを確認中...');
        const title = page.getByRole('heading', { level: 1 });
        await expect(title).toBeVisible();
        console.log('✓ H1見出しが表示されています');
        
        // セクション確認
        console.log('📝 セクションを確認中...');
        const sections = page.getByRole('heading', { level: 2 });
        const count = await sections.count();
        expect(count).toBeGreaterThan(0);
        console.log(`✓ ${count}個のセクションが見つかりました`);
        
        console.log('✅ Scenario 1完了: 基本フローが正常に動作しています');
      });
      
      test('リスト要素が正しくレンダリングされている', async ({ page }) => {
        // 統一URL解決システム: 複数環境変数からフォールバック
        const readmeUrl = process.env.README_PAGE_URL || 
                         process.env.TEST_PAGE_URL || 
                         process.env.ROOT_PAGE_URL;
        await page.goto(readmeUrl, { waitUntil: 'domcontentloaded' });
        await page.waitForTimeout(2000);
        
        // リストアイテムの存在確認
        const listItems = page.getByRole('listitem');
        const count = await listItems.count();
        expect(count).toBeGreaterThan(0);
        console.log(`✓ ${count}個のリストアイテムが見つかりました`);
          });
        });
```

---

### Phase 5: ハイブリッドテストスクリプト生成

```yaml
- name: "generate_hybrid_test_script"
  action: "create_file"
  path: "{{test_dir}}/run_scenario2_hybrid.sh"
      content: |
    #!/usr/bin/env bash
    set -e
    
    # カラー定義
    RED='\033[0;31m'
    GREEN='\033[0;32m'
    BLUE='\033[0;34m'
    YELLOW='\033[1;33m'
    NC='\033[0m'
    
    echo -e "${BLUE}🚀 Scenario 2 Hybrid Test開始...${NC}"
    
    # 環境変数読み込み
    if [ -f ".env" ]; then
        export $(grep -v '^#' .env | xargs)
        echo "✅ .envファイル読み込み完了"
    else
        echo -e "${RED}❌ .envファイルが見つかりません${NC}"
        exit 1
    fi
    
    # テストディレクトリ作成
    TEST_DIR="test_scenario2_$(date +%Y%m%d_%H%M%S)"
    mkdir -p "$TEST_DIR"
    cd "$TEST_DIR"
    
    echo -e "${GREEN}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    echo -e "${GREEN}Phase 1: 初期セットアップ（Cursor）${NC}"
    echo -e "${GREEN}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    
    # 仮想環境有効化
    if [ -d "../venv_test" ]; then
        source ../venv_test/bin/activate
    fi
    
    # CLIコマンド実行例
    echo "📦 プロジェクト初期化中..."
    # your_cli_command init .
    
    echo -e "${GREEN}✅ Phase 1完了${NC}"
    echo ""
    
    # キャッシュクリア（重要！）
    echo "🗑️ キャッシュをクリア中..."
    rm -rf .cache
    
    echo -e "${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    echo -e "${BLUE}Phase 2: 初期状態検証（Playwright）${NC}"
    echo -e "${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    
    cd ..
    # 統一URL解決システム: config.jsonから取得
    CONFIG_FILE="$TEST_DIR/.c2n/config.json"
    if [ -f "$CONFIG_FILE" ]; then
        TEST_PAGE_URL=$(python3 -c "
import json
try:
    with open('$CONFIG_FILE', 'r') as f:
        data = json.load(f)
        print(data.get('default_parent_url', ''))
except:
    pass
")
        echo "📋 統一URLシステムから取得: $TEST_PAGE_URL"
    else
        TEST_PAGE_URL="$NOTION_ROOT_URL"
        echo "⚠️ 環境変数からURL取得: $TEST_PAGE_URL"
    fi
    
    export TEST_PAGE_URL
    npx playwright test tests/scenario2-phase2-verify.spec.ts --project=chromium
    
    if [ $? -ne 0 ]; then
        echo -e "${RED}❌ Phase 2失敗${NC}"
        exit 1
    fi
    
    echo -e "${GREEN}✅ Phase 2完了${NC}"
    echo ""
    
    echo -e "${GREEN}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    echo -e "${GREEN}✅ Scenario 2 Hybrid Test完了${NC}"
    echo -e "${GREEN}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
```

---

### Phase 6: バリデーションスクリプト生成

```yaml
- name: "generate_validation_script"
  action: "create_file"
  path: "{{test_dir}}/scripts/validate_playwright_tests.sh"
      content: |
    #!/usr/bin/env bash
    set -e
    
    echo "🔍 Playwrightテストコードをバリデーション中..."
    
    # TypeScriptシンタックスチェック
    echo "📝 TypeScriptシンタックスチェック..."
    npx tsc --noEmit --project tsconfig.json
    
    if [ $? -ne 0 ]; then
        echo "❌ TypeScriptエラーが見つかりました"
        exit 1
    fi
    
    echo "✅ TypeScriptシンタックスOK"
    
    # Playwright設定チェック
    echo "📝 Playwright設定チェック..."
    if [ ! -f "playwright.config.ts" ]; then
        echo "❌ playwright.config.ts が見つかりません"
        exit 1
    fi
    
    # テストファイル存在チェック
    echo "📝 テストファイル存在チェック..."
    test_files=$(find tests -name "*.spec.ts" | wc -l)
    if [ $test_files -eq 0 ]; then
        echo "❌ テストファイルが見つかりません"
        exit 1
    fi
    
    echo "✅ ${test_files}個のテストファイルを発見"
    
    # セレクタパターンチェック
    echo "📝 推奨セレクタパターンチェック..."
    bad_selectors=$(grep -r "page\.locator('[^']*#[^']*')" tests/ | wc -l)
    if [ $bad_selectors -gt 0 ]; then
        echo "⚠️ ID セレクタの使用が ${bad_selectors}箇所見つかりました（推奨: getByRole）"
    fi
    
    good_selectors=$(grep -r "getByRole\|getByText\|getByLabel" tests/ | wc -l)
    echo "✅ 推奨セレクタの使用: ${good_selectors}箇所"
    
    # ハードコードされた待機時間チェック
    echo "📝 ハードコードされた待機時間チェック..."
    hard_waits=$(grep -r "waitForTimeout" tests/ | wc -l)
    if [ $hard_waits -gt 5 ]; then
        echo "⚠️ waitForTimeout の使用が ${hard_waits}箇所（推奨: waitForSelector使用）"
    fi
    
    echo ""
    echo "✅ バリデーション完了"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "📊 サマリー:"
    echo "  - テストファイル: ${test_files}個"
    echo "  - 推奨セレクタ使用: ${good_selectors}箇所"
    echo "  - 改善推奨箇所: $((bad_selectors + (hard_waits > 5 ? 1 : 0)))箇所"
```

```yaml
- name: "generate_tsconfig"
  action: "create_file"
  path: "{{test_dir}}/tsconfig.json"
  content: |
    {
      "compilerOptions": {
        "target": "ES2020",
        "module": "commonjs",
        "lib": ["ES2020"],
        "strict": true,
        "esModuleInterop": true,
        "skipLibCheck": true,
        "forceConsistentCasingInFileNames": true,
        "resolveJsonModule": true,
        "types": ["node", "@playwright/test"]
      },
      "include": ["tests/**/*", "playwright.config.ts"],
      "exclude": ["node_modules"]
    }
```

---

### Phase 7: 手動検証ガイド生成

```yaml
- name: "generate_manual_verification_guide"
  action: "create_file"
  path: "{{test_dir}}/MANUAL_VERIFICATION_GUIDE.md"
      content: |
    # 手動検証ガイド
    
    ## 📋 概要
    完全自動化が難しい部分について、手動検証の手順を記載しています。
    
    ## 🎯 手動検証が必要な理由
    - Notion UIの複雑な構造により、セレクタが不安定
    - 認証フローに外部サービス（Google SSO）が絡む
    - データの`last_edited_time`更新タイミングが不確定
    
    ## 📝 手動検証手順
    
    ### Scenario 2: Pull & Conflict Resolution
    
    #### Phase 1-2: 自動実行（Cursor + Playwright）
    ```bash
    ./run_scenario2_hybrid.sh
    ```
    
    #### Phase 3: 手動編集（5分待機）
    
    **重要**: Notion UI で編集後、**必ず5分待機**してください。
    これはNotionの`last_edited_time`が内部的に更新されるまでの時間です。
    
    1. ブラウザでNotionページを開く
    2. 以下を編集:
       - Section A: 最後に "Edited via Notion UI." を追加
       - Section B: 内容を完全に書き換え
       - 新しいセクション追加
    3. **タイマーを5分にセット** ⏰
    4. 待機中に他の作業をする（コーヒーブレイク推奨 ☕）
    
    #### Phase 4-7: 自動実行（Cursor）
    ```bash
    # Phase 3の5分待機後に実行
    ./run_scenario2_hybrid_continue.sh
    ```
    
    ### 成功基準
    - [ ] Phase 5 で "Found X pages changed" と表示される
    - [ ] コンフリクトマーカー（`<<<<<<< LOCAL`）が生成される
    - [ ] Phase 7 で最終状態が正しく反映される
    
    ### トラブルシューティング
    
    #### "No changes found" と表示される
    **原因**: `last_edited_time`が更新されていない
    **対処**: さらに5分待ってから Phase 4 を再実行
    
    #### キャッシュが残っている
    **原因**: 前回のテスト実行のキャッシュが残っている
    **対処**:
    ```bash
    rm -rf .cache .c2n/cache
    ```
    
    ## 📊 検証結果の記録
    
    検証完了後、以下を記録してください：
    
    - [ ] 検証日時: _______________
    - [ ] 検証者: _______________
    - [ ] 全Phaseが成功したか: [ ] はい [ ] いいえ
    - [ ] 所要時間: 約 _______ 分
    - [ ] 発見した問題点:
      ```
      
      
      ```
```

---

### Phase 8: package.jsonとREADME生成

```yaml
- name: "generate_package_json"
  action: "create_file"
  path: "{{test_dir}}/package.json"
      content: |
        {
      "name": "{{project_name}}-integration-tests",
      "version": "1.0.0",
      "description": "ハイブリッド統合テスト - {{project_name}}",
      "scripts": {
        "test": "playwright test",
        "test:ui": "playwright test --ui",
        "test:headed": "playwright test --headed",
        "test:debug": "playwright test --debug",
        "test:scenario1": "playwright test tests/scenario1*.spec.ts",
        "validate": "bash scripts/validate_playwright_tests.sh",
        "hybrid:scenario2": "bash run_scenario2_hybrid.sh"
      },
      "keywords": ["playwright", "e2e", "hybrid-test"],
      "devDependencies": {
        "@playwright/test": "^1.40.0",
        "@types/node": "^20.0.0",
        "typescript": "^5.0.0"
      }
    }

- name: "generate_main_readme"
  action: "create_file"
  path: "{{test_dir}}/README_INTEGRATION_TEST.md"
      content: |
    # 統合テスト実行ガイド
    
    ## 🎯 このテストスイートについて
    
    このテストスイートは、**ハイブリッド形式**を採用しています：
    - **Cursorコマンド**: CLIツールの実行、データ操作
    - **Playwright**: UI検証、表示確認
    - **手動ステップ**: 完全自動化が困難な部分
        
        ## 🚀 セットアップ
        
    ### 1. 依存関係インストール
        ```bash
        npm install
    npx playwright install chromium
        ```
        
    ### 2. 環境変数設定
        ```bash
    cp .env.example .env
    # .envファイルを編集して必要な値を設定
    ```
    
    ### 3. バリデーション実行
        ```bash
    npm run validate
        ```
        
    ## 🧪 テスト実行
    
    ### Scenario 1: 基本フロー（完全自動）
        ```bash
    npm run test:scenario1
        ```
        
    ### Scenario 2: エラーハンドリング（ハイブリッド）
        ```bash
    npm run hybrid:scenario2
        ```
    **注意**: 手動ステップが含まれます。画面の指示に従ってください。
        
    ### 全テスト実行
        ```bash
    npm test
        ```
        
    ## 📊 テスト結果の確認
    
        ```bash
    npx playwright show-report
    ```
    
    ## 🐛 トラブルシューティング
    
    ### テストが失敗する
    1. `playwright-report/` のスクリーンショット確認
    2. `npm run validate` でコードチェック
    3. `.env` の設定を確認
    
    ### 認証エラー
        ```bash
    rm -rf .auth
    npx playwright test tests/auth.setup.ts
    ```
    
    ## 📝 ベストプラクティス
    
    1. **セレクタ**: `getByRole`, `getByText` を優先
    2. **待機**: `waitForSelector` を使用（`waitForTimeout`は最小限に）
    3. **環境クリーンアップ**: テスト前にキャッシュをクリア
    4. **手動ステップ**: `MANUAL_VERIFICATION_GUIDE.md` を参照
    
    ## 🔗 参考資料
    
    - `MANUAL_VERIFICATION_GUIDE.md` - 手動検証手順
    - `TEST_DESIGN.md` - テスト設計書
    - `scripts/validate_playwright_tests.sh` - バリデーションスクリプト
```

---

### Phase 9: バリデーション実行

```yaml
- name: "copy_validation_scripts"
  action: "execute_shell"
  command: |
    cp scripts/validate_playwright_tests.sh {{test_dir}}/scripts/
    cp scripts/validate_playwright_tests.py {{test_dir}}/scripts/
    chmod +x {{test_dir}}/scripts/validate_playwright_tests.sh
  message: "バリデーションスクリプトをコピーしました"

- name: "run_validation"
  action: "execute_shell"
  command: |
    cd {{test_dir}}
    bash scripts/validate_playwright_tests.sh . | tee validation_results.log
  message: "Playwrightテストコードをバリデーション中..."

- name: "display_validation_results"
  action: "display"
  content: |
    📊 バリデーション結果:
    
    詳細は {{test_dir}}/validation_results.log を確認してください。
    
    バリデーション合格の場合、次のステップに進めます。
    警告がある場合は、品質向上のため修正を推奨します。

---

### Phase 10: サマリーレポート作成

```yaml
- name: "create_summary_report"
  action: "create_file"
  path: "{{test_dir}}/TEST_SUITE_SUMMARY.md"
  content: |
    # 統合テストスイート作成完了レポート
    
    ## 📦 生成されたファイル
    
    ### 設定ファイル
    - `package.json` - npm設定
    - `playwright.config.ts` - Playwright設定（実用的な設定）
    - `tsconfig.json` - TypeScript設定
    - `.env.example` - 環境変数テンプレート
    
    ### テストファイル
    - `tests/auth.setup.ts` - 認証セットアップ（Google SSO対応）
    - `tests/scenario1-basic-flow.spec.ts` - 基本フロー検証
    
    ### ハイブリッドテストスクリプト
    - `run_scenario2_hybrid.sh` - Scenario 2 実行スクリプト
    
    ### バリデーション
    - `scripts/validate_playwright_tests.sh` - コード品質チェック
    
    ### ドキュメント
    - `README_INTEGRATION_TEST.md` - 実行ガイド
    - `MANUAL_VERIFICATION_GUIDE.md` - 手動検証手順
    - `TEST_DESIGN.md` - テスト設計書
    
    ## ✅ 実証済みパターンを採用
    
    ### 認証処理
    - Google SSOの「Continue as [User]」ポップアップ対応
    - `domcontentloaded` + `waitForTimeout` の組み合わせ
    
    ### セレクタ戦略
    - `page.getByRole('heading', { level: 1 })` - 意味的なセレクタ
    - `page.getByText(/パターン/i)` - テキストベース検索
    - iframe回避 - 直接DOM要素にアクセス
    
    ### キャッシュ管理
    - テスト前の明示的なキャッシュクリア
    - `nit pull` の差分検出問題に対応
    
    ### 手動ステップの組み込み
    - 5分待機など、自動化困難な部分を明示
    - 手動検証ガイドで詳細な手順を提供
    
    ## 🎯 次のステップ
    
    1. **バリデーション実行**:
           ```bash
       cd {{test_dir}}
       npm install
       npm run validate
           ```
        
    2. **Scenario 1 実行**:
        ```bash
       npm run test:scenario1
       ```
    
    3. **Scenario 2 実行**（手動ステップあり）:
       ```bash
       npm run hybrid:scenario2
       ```
    
    4. **結果確認**:
       ```bash
       npx playwright show-report
       ```
    
    ## 📊 品質指標（バリデーション結果）
    
    ### 自動バリデーション実行済み
    - **ステータス**: {{validation_status}}
    - **品質レベル**: {{validation_quality_level}}
    - **総テストファイル数**: {{validation_test_files}}個
    - **総アサーション数**: {{validation_assertions}}個
    - **推奨セレクタ使用**: {{validation_good_selectors}}箇所
    - **警告**: {{validation_warnings}}件
    - **エラー**: {{validation_errors}}件
    
    ### 品質チェック項目
    - ✅ TypeScript型チェック: 実行済み
    - ✅ 推奨セレクタパターン: 適用済み
    - ✅ アサーション品質: 検証済み
    - ✅ アンチパターン検出: 実施済み
    - ✅ 手動ステップ: ドキュメント化済み
    
    詳細: `validation_results.log` を参照
    
    ## 🐛 既知の制約事項
    
    1. **Notion `last_edited_time`**: 秒単位でしか更新されないため、5分待機が必要
    2. **キャッシュ依存**: `nit pull` の差分検出にキャッシュが影響
    3. **Google SSO**: 初回認証時は手動操作が必要
    
    ## 💡 改善提案
    
    - IMP-008: `nit pull` のSHA1ハッシュによる差分検出
    - IMP-009: キャッシュクリアオプションの追加
    - Playwrightセレクタの `data-testid` 属性導入
    
    - name: "notify_completion"
      action: "display"
      content: |
    ✅ ハイブリッド統合テストスイート作成完了
    
    📁 生成場所: {{test_dir}}
    
    🔍 バリデーション結果:
    - ステータス: {{validation_status}}
    - 品質レベル: {{validation_quality_level}}
    - エラー: {{validation_errors}}件
    - 警告: {{validation_warnings}}件
    
    📝 即座に実行可能なコマンド:
    1. cd {{test_dir}}
    2. npm install
    3. npx playwright install chromium
    4. npm run test:scenario1  # Scenario 1実行
    
    📝 バリデーション再実行:
    - bash scripts/validate_playwright_tests.sh
    - python3 scripts/validate_playwright_tests.py
    
    📚 重要なドキュメント:
    - README_INTEGRATION_TEST.md - 実行ガイド
    - MANUAL_VERIFICATION_GUIDE.md - 手動検証手順
    - TEST_SUITE_SUMMARY.md - 完了レポート
    
    🎯 このテストスイートの特徴:
    ✓ 実証済みのPlaywrightパターン採用
    ✓ ハイブリッドテスト形式（CLI + Playwright + Manual）
    ✓ バリデーションスクリプト付き
    ✓ 詳細な手動検証ガイド
    ✓ キャッシュ管理など実環境の問題に対応
```

---

## トリガーパターン
- `結合テスト作成`
- `統合テスト作成`
- `ハイブリッドテスト作成`
- `Integration Test Create`

## 成果物
- Playwright統合テストスイート（完全動作保証）
- バリデーションスクリプト
- ハイブリッドテスト実行スクリプト
- 手動検証ガイド

## 次のコマンド
→ `11-4_結合テスト実施` で実際にテストを実行
→ `12_バグ登録` で発見した問題を記録

## 変更履歴

### v2.2 (2025-10-16) - Playwrightセレクター戦略の完全ガイド追加
- **複数セレクターのフォールバック戦略**: UX検証スクリプトの成功パターンを汎用化
- **Notion DOM構造対応**: `.notion-bulleted_list-block`等のNotion固有クラス名対応
- **Strict Mode Violation対策**: `.first()`の明示的使用パターン
- **見出しセレクター戦略**: `h2:has-text()`と`:text()`の組み合わせ
- **実行オプション最適化**: `--project`指定による再認証問題の回避
- **実装テンプレート**: `findElementWithFallback`汎用ヘルパー関数
- **チェックリスト**: テスト作成・実行・トラブルシューティングの体系化

### v2.1 (2025-10-15) - URL統一システム対応
- **URL統一システム**: `default_parent_url`を唯一のソースとして使用
- **統一URL解決**: `config.json` → `NOTION_ROOT_URL` の2段階優先順位
- **複数環境変数フォールバック**: `TEST_PAGE_URL`, `ROOT_PAGE_URL`, `README_PAGE_URL`
- **実証済み認証パターン**: 「Daisukeとして実行」ボタンの複数セレクタ対応
- **自動移行サポート**: 既存プロジェクトの自動移行スクリプト
- **レガシー互換性**: 既存設定との互換性を保持

### v2.0 (2025-10-14) - 実用的なハイブリッドテスト対応
- **実証済みパターン採用**: 今回のスレッドで成功したコードを基に実装
- **ハイブリッドテスト**: Cursor + Playwright + Manual の組み合わせ
- **バリデーション機能**: TypeScriptチェック、セレクタパターン検証
- **手動検証ガイド**: 自動化困難な部分の詳細手順
- **キャッシュ管理**: `nit pull` の差分検出問題に対応
- **実用的な設定**: タイムアウト、リトライ、レポート設定を最適化
