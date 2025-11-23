# : 成果物確認（開発レビュー）

## 目的
Delivery2の成果物（コード/記事/設計/テスト）をレビューし、品質チェックと確定反映までの段取りを整えます。

## 実行手順（Rules Steps）
```yaml
- trigger: "(delivery2_成果物確認|dev_review2)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/delivery2_story_*.md','**/delivery2_implementation_order.md','**/dev_tasks.yaml']))}}"]
      instructions: |
        確認対象/主要変更点/品質チェックの候補を抽出し、display用に1-2行ずつ提示してください。
      store_as: "auto_review_seed"
    - name: "prefill_review_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_review_seed}}
    - name: "collect_review"
      action: "ask_questions"
      questions:
        - key: project_id
          question: "プロジェクトID"
          required: true
        - key: review_target
          question: "確認対象（ファイルパスまたはストーリーID）"
          required: true
        - key: output_type
          question: "開発成果物の種類"
          options: ["ソースコード","記事/ドキュメント","設計資料","テスト結果","その他"]
          required: true
        - key: key_changes
          question: "主要な変更点/機能"
          required: true
        - key: verification_results
          question: "動作確認結果（コードの場合は任意）"
          required: false
        - key: quality_checks
          question: "品質チェック内容"
          required: true
        - key: pending_items
          question: "未解決の問題/TODO項目（任意）"
          required: false
        - key: final_path
          question: "確定先のパス（Stock内）"
          required: true
      store_as: rv

    - name: "confirm"
      action: "confirm"
      message: "開発成果物レビューを作成します。よろしいですか？"

    - name: "create_draft"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/delivery2_development_review.md"
      content: |
        ---
        doc_type: development_review
        project_id: {{rv.project_id}}
        created_at: {{today}}
        version: v1.0
        ---

        # 開発成果物レビュー：{{rv.project_id}}

        ## 1. レビュー概要
        - 対象: {{rv.review_target}}
        - 種類: {{rv.output_type}}
        - レビュー日: {{today}}

        ## 2. 主要な変更点/機能
        {{rv.key_changes}}

        {{#if (eq rv.output_type "ソースコード")}}
        ## 3. 動作確認結果
        {{rv.verification_results}}
        {{/if}}

        ## 4. 品質チェック
        {{rv.quality_checks}}

        {{#if rv.pending_items}}
        ## 5. 未解決の問題/TODO
        {{rv.pending_items}}
        {{/if}}

        ## 6. 確定情報
        - 確定先パス: {{rv.final_path}}
        - 確定方法: Flow → Stock への移動

        ## 7. 確定手順
        1. コードレビュー完了の確認
        2. テストの実行・確認
        3. 「確定反映して」コマンドの実行
        4. {{rv.final_path}} への反映確認

        ## 8. 次のステップ
        1. 残存する問題の対応計画
        2. 次の開発項目への移行

    - name: "notify"
      action: "notify"
      message: |
        ✅ 生成しました：{{patterns.flow_date}}/delivery2_development_review.md
        必要に応じて「確定反映して」でStockへ反映してください。
```

## 次に実行
- `03_スプリントレビュー` → ロードマップ/Backlog2調整へ
