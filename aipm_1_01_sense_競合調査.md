# : 競合調査（Web検索ベース）

## 最初に質問（実行前に回答してください）
- 調査対象のプロジェクト名（必須）
- 自社の製品・サービスの概要（必須）
- 主な競合企業・サービス（任意・カンマ区切り）
- 競合調査で特に知りたい側面（必須・例：価格戦略、マーケティング手法）

## 目的
Senseは情報を収集し、発散的にオポチュニティ（機会）を見つけるフェーズです。本コマンドは競合観点からの知見を集め、比較マトリクス雛形も作成します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(sense_競合調査|Competitor Research)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}", "{{read_files(find_files(patterns=['Flow/**/*.md','Flow/**/*.mdx','**/submit_*/README.md']))}}"]
      instructions: |
        直近スレッドの会話と同スレッドで生成されたMarkdownから、以下キーの推定値を出力してください（なければ空文字）。
        keys: project_name, product_service, main_competitors, research_focus
        出力はJSONのオブジェクト1件のみ。
      store_as: "prefill"
    - name: "collect_research_requirements"
      action: "ask_questions"
      questions:
        - key: "project_name"
          question: "調査対象のプロジェクト名を入力してください"
          required: true
          default: "{{prefill.project_name}}"
        - key: "product_service"
          question: "自社の製品・サービスの概要を簡潔に教えてください"
          required: true
          default: "{{prefill.product_service}}"
        - key: "main_competitors"
          question: "主な競合企業・サービスがあれば教えてください（カンマ区切りで複数可）"
          required: false
          default: "{{prefill.main_competitors}}"
        - key: "research_focus"
          question: "競合調査で特に知りたい側面を教えてください（例：価格戦略、マーケティング手法、製品機能など）"
          required: true
          default: "{{prefill.research_focus}}"
      store_as: "research_params"

    - name: "confirm_research"
      action: "confirm"
      message: "以下の内容で競合調査を実施します：\n\nプロジェクト: {{research_params.project_name}}\n自社製品/サービス: {{research_params.product_service}}\n主な競合: {{research_params.main_competitors}}\n調査フォーカス: {{research_params.research_focus}}\n\nWeb検索を開始してよろしいですか？"

    - name: "ensure_output_dir"
      action: "execute_shell"
      command: "mkdir -p {{patterns.flow_date}}/1_sense/01_競合調査"

    - name: "web_research_competitors"
      action: "web_search"
      search_term: "{{research_params.product_service}} 業界 主要企業 競合 マーケットシェア 最新"
      explanation: "業界の主要競合企業に関する情報を収集します"
      store_as: "competitors_data"

    - name: "web_research_specific_competitors"
      action: "web_search"
      search_term: "{{research_params.main_competitors}} {{research_params.research_focus}} 戦略 特徴 最新動向"
      condition: "research_params.main_competitors"
      explanation: "指定された競合企業についての詳細情報を収集します"
      store_as: "specific_competitors_data"

    - name: "web_research_market_trends"
      action: "web_search"
      search_term: "{{research_params.product_service}} 市場動向 業界トレンド 最新技術 将来予測"
      explanation: "業界の市場動向と最新トレンドに関する情報を収集します"
      store_as: "market_trends_data"

    - name: "web_research_competitive_strategies"
      action: "web_search"
      search_term: "{{research_params.product_service}} 業界 {{research_params.research_focus}} 競争戦略 差別化要因"
      explanation: "競合の戦略と差別化要因に関する情報を収集します"
      store_as: "strategies_data"

    - name: "analyze_web_research"
      action: "analyze"
      data: ["{{competitors_data}}", "{{specific_competitors_data}}", "{{market_trends_data}}", "{{strategies_data}}"]
      instructions: "収集したWeb検索データを分析し、競合の主要特性・市場ポジション・戦略・強み/弱み・示唆（Implications）を抽出してください。"
      store_as: "analyzed_results"

    - name: "create_draft"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/1_sense/01_競合調査/sense_competitor_research.md"
      template_reference: "basic/02_pmbok_research.mdc => competitor_research_template"

    - name: "create_competitor_classification"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/1_sense/01_競合調査/sense_competitor_summary_and_matrix.md"
      message: "競合/代用品/代替品の分類リストと比較マトリクスのドラフトを作成します（後続で上書き可能）。"

    - name: "append_comparison_matrix"
      action: "append_to_file"
      file: "{{patterns.flow_date}}/1_sense/01_競合調査/sense_competitor_summary_and_matrix.md"
      block: |
        \n---\n\n## 比較マトリクス（雛形）\n\n| 項目 | 競合A | 競合B | 競合C | 代用品 | 代替品 |\n| --- | --- | --- | --- | --- | --- |\n| 主な用途 |  |  |  |  |  |\n| 導入/UX |  |  |  |  |  |\n| ガバナンス |  |  |  |  |  |\n| テンプレ/ワンクリック |  |  |  |  |  |\n| 計測/監査 |  |  |  |  |  |\n| 価格/契約 |  |  |  |  |  |\n| 強み |  |  |  |  |  |\n| 弱み |  |  |  |  |  |
      message: "比較マトリクスの雛形を追記しました。"
```

## 次に実行
- `02_顧客調査` と合わせてインサイトをプール
- `03_オポチュニティ仮説抽出` で機会仮説リストへ圧縮


