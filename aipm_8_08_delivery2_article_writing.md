# 08 Delivery2 Article Writing

## 目的
Delivery2に関連する技術解説・手順・ケーススタディ等のコンテンツを計画・執筆します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(delivery2_記事執筆|article_write2)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/delivery2_development_review.md','**/delivery2_story_*.md']))}}"]
      instructions: |
        記事タイトル/目的/種類/キーワードの候補を抽出し、display用に箇条書きで提示してください。
      store_as: "auto_article_seed"
    - name: "prefill_article_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_article_seed}}
    - name: "collect_article"
      action: "ask_questions"
      questions:
        - key: project_id
          question: "プロジェクトID"
          required: true
        - key: article_title
          question: "記事タイトル"
          required: true
        - key: article_purpose
          question: "記事の目的/対象読者"
          required: true
        - key: article_type
          question: "記事の種類"
          options: ["技術解説","チュートリアル","ケーススタディ","比較分析","トレンド解説","その他"]
          required: true
        - key: keywords
          question: "主要なキーワード/トピック"
          required: true
        - key: references
          question: "参考資料/ソース（任意）"
          required: false
        - key: outline
          question: "記事の構成案（任意）"
          required: false
        - key: attachments
          question: "添付資料（図表など）（任意）"
          required: false
      store_as: art

    - name: "confirm"
      action: "confirm"
      message: "記事ドラフトを作成します。よろしいですか？"

    - name: "create_draft"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/delivery2_article.md"
      content: |
        ---
        doc_type: article
        project_id: {{art.project_id}}
        created_at: {{today}}
        version: v1.0
        ---

        # {{art.article_title}}

        ## 記事計画
        - 種類: {{art.article_type}}
        - 目的/対象読者: {{art.article_purpose}}
        - キーワード: {{art.keywords}}

        {{#if art.references}}
        ## 参考資料
        {{art.references}}
        {{/if}}

        ## 構成案
        {{#if art.outline}}
        {{art.outline}}
        {{else}}
        1. はじめに（背景/目的）
        2. 主要セクション1
        3. 主要セクション2
        4. まとめ（学び/次のステップ）
        {{/if}}

        {{#if art.attachments}}
        ## 添付資料
        {{art.attachments}}
        {{/if}}

        ## 執筆プラン
        1. 調査 → 2. ドラフト → 3. レビュー → 4. 最終化

        ## 記事本文
        *ここに本文を執筆します*

        ## 公開先
        - {{patterns.dev_articles}}/[ファイル名]
        - その他

    - name: "notify"
      action: "notify"
      message: |
        ✅ 生成しました：{{patterns.flow_date}}/delivery2_article.md
```

## 次に実行
- `09_成果物確認`
