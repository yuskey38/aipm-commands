# 07 Delivery2 Story Implementation

## 目的
個別ストーリーの実装計画を整備し、成果物・テスト・次アクションまで明確化します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(delivery2_ストーリー実装|story_impl2)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/backlog/backlog2.yaml','**/delivery2_implementation_order.md','**/strategy_roadmap.yaml']))}}"]
      instructions: |
        実装対象のストーリー候補（ID/タイトル/受入条件/関連情報）を抽出し、1-3件を箇条書きで提示してください。
      store_as: "auto_story_seed"
    - name: "prefill_story_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_story_seed}}
    - name: "collect_story"
      action: "ask_questions"
      questions:
        - key: project_id
          question: "プロジェクトID"
          required: true
        - key: story_id
          question: "実装するストーリーID"
          required: true
        - key: story_title
          question: "ストーリータイトル"
          required: true
        - key: story_description
          question: "ストーリー内容"
          required: true
        - key: acceptance_criteria
          question: "受入条件"
          required: true
        - key: technologies
          question: "実装言語/フレームワーク"
          required: true
        - key: related_docs
          question: "関連するドキュメント/コード（任意）"
          required: false
        - key: constraints
          question: "注意点/制約（任意）"
          required: false
      store_as: st

    - name: "confirm"
      action: "confirm"
      message: "ストーリー実装計画を作成します。よろしいですか？"

    - name: "create_draft"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/delivery2_story_{{st.story_id}}.md"
      content: |
        ---
        doc_type: story_implementation
        project_id: {{st.project_id}}
        story_id: {{st.story_id}}
        created_at: {{today}}
        version: v1.0
        ---

        # {{st.story_id}}: {{st.story_title}} 実装計画

        ## 1. ストーリー概要
        {{st.story_description}}

        ## 2. 受入条件
        {{st.acceptance_criteria}}

        ## 3. 技術要素
        - 実装言語/フレームワーク: {{st.technologies}}
        {{#if st.related_docs}}- 関連ドキュメント/コード: {{st.related_docs}}{{/if}}
        {{#if st.constraints}}- 制約事項: {{st.constraints}}{{/if}}

        ## 4. 実装計画
        ### 実装ステップ
        1. 環境準備
        2. テスト計画
        3. コア機能実装
        4. UI/UX実装（該当時）
        5. 統合・テスト

        ### 実装コード
        ```
        // 主要なコード構造やアルゴリズム案
        ```

        ## 5. 実装結果（実装後に記入）
        - 成果物の場所: code/assets のパス
        - 動作確認結果: 

        ## 6. 次のステップ
        1. コードレビュー依頼
        2. リファクタリング検討
        3. 次のストーリーへ

    - name: "notify"
      action: "notify"
      message: |
        ✅ 生成しました：{{patterns.flow_date}}/delivery2_story_{{st.story_id}}.md
```

## 次に実行
- `08_記事執筆`（文書の場合）
- `09_成果物確認`
