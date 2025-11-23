# : 顧客調査（Web検索ベース）

## 最初に質問（実行前に回答してください）
- 調査対象のプロジェクト名（必須）
- 調査したいターゲットオーディエンス（必須）
- 顧客調査で特に知りたい内容やトピック（必須・複数可）
- 業界や市場の背景情報（任意）

## 目的
Senseは情報を収集し、発散的にオポチュニティ（機会）を見つけるフェーズです。本コマンドは顧客観点からの知見を集め、後続のオポチュニティ仮説抽出へつなげます。

## 実行手順（Rules Steps）
```yaml
- trigger: "(sense_顧客調査|Customer Research)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}", "{{read_files(find_files(patterns=['Flow/**/*.md','Flow/**/*.mdx','**/submit_*/README.md']))}}"]
      instructions: |
        スレッド内の直近の文脈と同スレッドで生成されたMarkdownから、以下キーの推定値を出力（なければ空）。
        keys: project_name, target_audience, research_topics, industry_context
        出力はJSONのオブジェクト1件のみ。
      store_as: "prefill"
    - name: "collect_research_requirements"
      action: "ask_questions"
      questions:
        - key: "project_name"
          question: "調査対象のプロジェクト名を入力してください"
          required: true
          default: "{{prefill.project_name}}"
        - key: "target_audience"
          question: "調査したいターゲットオーディエンスを具体的に教えてください"
          required: true
          default: "{{prefill.target_audience}}"
        - key: "research_topics"
          question: "顧客調査で特に知りたい内容やトピックを教えてください（複数可、カンマ区切り）"
          required: true
          default: "{{prefill.research_topics}}"
        - key: "industry_context"
          question: "業界や市場の背景情報があれば教えてください"
          required: false
          default: "{{prefill.industry_context}}"
      store_as: "research_params"

    - name: "confirm_research"
      action: "confirm"
      message: "以下の内容で顧客調査を実施します：\n\nプロジェクト: {{research_params.project_name}}\nターゲット: {{research_params.target_audience}}\n調査トピック: {{research_params.research_topics}}\n\nWeb検索を開始してよろしいですか？"

    - name: "ensure_output_dir"
      action: "execute_shell"
      command: "mkdir -p {{patterns.flow_date}}/1_sense/02_顧客調査"

    - name: "web_research_audience"
      action: "web_search"
      search_term: "{{research_params.target_audience}} 顧客特性 消費者行動 ニーズ 最新動向"
      explanation: "ターゲットオーディエンスについての最新情報を収集します"
      store_as: "audience_data"

    - name: "web_research_topics"
      action: "web_search"
      search_term: "{{research_params.target_audience}} {{research_params.research_topics}} 消費者調査 市場調査 最新"
      explanation: "指定されたトピックに関する顧客調査情報を収集します"
      store_as: "topic_data"

    - name: "web_research_behavior"
      action: "web_search"
      search_term: "{{research_params.target_audience}} 購買行動 意思決定プロセス 顧客体験 {{research_params.industry_context}}"
      explanation: "ターゲットの購買行動や意思決定プロセスに関する情報を収集します"
      store_as: "behavior_data"

    - name: "analyze_web_research"
      action: "analyze"
      data: ["{{audience_data}}", "{{topic_data}}", "{{behavior_data}}"]
      instructions: "収集データから顧客の課題・未充足ニーズ・期待される体験・阻害要因を抽出し、示唆（Implications）を付記してください。"
      store_as: "analyzed_results"

    - name: "create_draft"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/1_sense/02_顧客調査/sense_customer_research.md"
      template_reference: "basic/02_pmbok_research.mdc => customer_research_template"
```

## 次に実行
- `01_競合調査` と合わせてインサイトをプール
- `03_オポチュニティ仮説抽出` で機会仮説リストへ圧縮


