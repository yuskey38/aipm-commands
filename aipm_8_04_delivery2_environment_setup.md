# 04 Delivery2 Environment Setup

## 目的
Discovery2/Backlog2を前提に、二段階目Deliveryの開発環境を初期化します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(delivery2_開発初期化|開発環境セットアップ2)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/delivery2_development_plan.md','**/strategy_roadmap.yaml','**/discovery2_designdoc.md']))}}"]
      instructions: |
        プロジェクトID/開発種別/参照ドキュメント候補などの初期値を抽出して提示してください。
      store_as: "auto_setup_seed"
    - name: "prefill_setup_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_setup_seed}}
    - name: "collect_init"
      action: "ask_questions"
      questions:
        - key: project_id
          question: "プロジェクトID"
          required: true
        - key: dev_type
          question: "開発種別"
          options: ["ソフトウェア開発","記事/ドキュメント作成","複合タイプ"]
          required: true
        - key: technologies
          question: "開発言語・フレームワーク（任意）"
          required: false
        - key: article_themes
          question: "記事カテゴリ・テーマ（任意）"
          required: false
        - key: source_control
          question: "ソース管理方法"
          options: ["Git","ローカルのみ","その他"]
          default: "Git"
          required: true
        - key: reference_docs
          question: "参照するドキュメント（カンマ区切り）"
          required: true
        - key: setup_info
          question: "初期セットアップに必要な追加情報（任意）"
          required: false
      store_as: init

    - name: "confirm"
      action: "confirm"
      message: "Delivery2 開発環境セットアップドキュメントを作成します。よろしいですか？"

    - name: "create_draft"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/delivery2_development_setup.md"
      content: |
        ---
        doc_type: development_setup
        project_id: {{init.project_id}}
        created_at: {{today}}
        version: v1.0
        ---

        # 開発環境セットアップ：{{init.project_id}}

        ## 1. 概要
        - 開発種別: {{init.dev_type}}
        - 参照ドキュメント: {{init.reference_docs}}

        ## 2. 環境構成
        {{#if init.technologies}}
        ### 技術スタック
        - 言語/フレームワーク: {{init.technologies}}
        {{/if}}
        {{#if init.article_themes}}
        ### 記事/ドキュメントテーマ
        - {{init.article_themes}}
        {{/if}}
        ### ソース管理
        - {{init.source_control}}

        ## 3. ディレクトリ構成
        ```
        {{init.project_id}}/
        ├── development/
        │   ├── code/
        │   ├── articles/
        │   └── assets/
        └── documents/
        ```

        ## 4. セットアップ手順
        1. 開発環境の初期化
           ```bash
           mkdir -p {{patterns.dev_root}}/{code,articles,assets}
           ```
        2. 参照ドキュメントの確認
           - {{init.reference_docs}}
        3. 初期コード/テンプレート作成
           ```bash
           touch {{patterns.dev_code}}/README.md
           mkdir -p {{patterns.dev_code}}/src
           ```
        {{#if init.setup_info}}
        ## 5. 追加設定情報
        {{init.setup_info}}
        {{/if}}

        ## 次のステップ
        1. 開発計画の作成
        2. 実装順序計画
        3. ストーリー実装

    - name: "notify"
      action: "notify"
      message: |
        ✅ 生成しました：{{patterns.flow_date}}/delivery2_development_setup.md
```

## 次に実行
- `05_開発計画作成`
