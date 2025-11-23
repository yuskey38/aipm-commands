# : プロダクト定義

## 目的
Senseのオポチュニティ仮説を踏まえ、関係者のビジョンと言語化を行い、プロダクト（プログラム）を定義し、あわせてポジションステートメントも作成します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(focus_プロダクト定義|Product Definition)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}", "{{read_files(find_files(patterns=['Flow/**/*.md','Flow/**/*.mdx','**/submit_*/README.md']))}}"]
      instructions: |
        スレッドの直近文脈と同スレッドの成果物から、以下キーの推定値を出力（なければ空）。
        keys: program_name, theme_strategy, mission, vision, value, product_vision, organization,
              when_context, what_unique, how_features, where_origin, why_need, who_target
        出力はJSONのオブジェクト1件のみ。
      store_as: "prefill"
    - name: "load_opportunities"
      action: "ask_question"
      question: "sense_opportunities.yaml のパス（任意。未指定ならスキップ）"
      required: false
      store_as: "op_path"

    - name: "collect_program_definition"
      action: "ask_questions"
      questions:
        - key: "program_name"
          question: "プロダクト/プログラム名は？"
          required: true
          default: "{{prefill.program_name}}"
        - key: "theme_strategy"
          question: "関連するテーマ/戦略は？"
          required: true
          default: "{{prefill.theme_strategy}}"
        - key: "mission"
          question: "Mission（何のために存在するか）"
          required: true
          default: "{{prefill.mission}}"
        - key: "vision"
          question: "Vision（どんな状態を目指すか）"
          required: true
          default: "{{prefill.vision}}"
        - key: "value"
          question: "Value（大切にする価値観）"
          required: true
          default: "{{prefill.value}}"
        - key: "product_vision"
          question: "プロダクトビジョン（数年後の理想）"
          required: true
          default: "{{prefill.product_vision}}"
        - key: "organization"
          question: "体制（PO/PM/主要ステークホルダー）"
          required: true
          default: "{{prefill.organization}}"
      store_as: "pd"

    - name: "collect_positioning"
      action: "ask_questions"
      questions:
        - key: "when_context"
          question: "いつ（時代/現状の文脈）"
          required: true
          default: "{{prefill.when_context}}"
        - key: "what_unique"
          question: "なにを（唯一性：世界/日本で唯一の何か）"
          required: true
          default: "{{prefill.what_unique}}"
        - key: "how_features"
          question: "どのように（特徴的機能・差別化 3-5点）"
          required: true
          default: "{{prefill.how_features}}"
        - key: "where_origin"
          question: "どこの（出自/対象市場/地理）"
          required: true
          default: "{{prefill.where_origin}}"
        - key: "why_need"
          question: "なぜ（解決するニーズ/問題 1文）"
          required: true
          default: "{{prefill.why_need}}"
        - key: "who_target"
          question: "だれに（明確な対象/セグメント）"
          required: true
          default: "{{prefill.who_target}}"
      store_as: "pos"

    - name: "wait"
      action: "wait_for_all_answers"
      message: "必要な回答が揃うまで先に進みません。"

    - name: "confirm"
      action: "confirm"
      message: "プロダクト定義とポジションステートメントのドラフトを作成します。よろしいですか？"

    - name: "create_product_definition"
      action: "edit_file"
      path: "{{patterns.flow_date}}/focus_product_definition.md"
      content: |
        # プロダクト（プログラム）定義 - {{pd.program_name}}

        ## テーマ/戦略
        {{pd.theme_strategy}}

        ## Mission / Vision / Value
        - Mission: {{pd.mission}}
        - Vision: {{pd.vision}}
        - Value : {{pd.value}}

        ## プロダクトビジョン
        {{pd.product_vision}}

        ## 体制
        {{pd.organization}}

        ## 参考: Senseのオポチュニティ
        {{#if op_path}}（参照: {{op_path}}）{{/if}}

    - name: "create_positioning_statement"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/focus_positioning_statement.md"
      content: |
        # ポジションステートメント - {{pd.program_name}}

        - いつ：{{pos.when_context}} において
        - なにを：私たちのプロダクトは世界（日本）で唯一の「{{pos.what_unique}}」である
        - どのように：特徴的機能は {{pos.how_features}}
        - どこの：{{pos.where_origin}}
        - なぜ：{{pos.why_need}}
        - だれに：{{pos.who_target}} のために提供する

    - name: "notify_next"
      action: "notify"
      message: |
        ✅ ドラフトを作成しました。
        - {{patterns.flow_date}}/focus_product_definition.md
        - {{patterns.flow_date}}/focus_positioning_statement.md
        次は「02_市場規模推定」→「03_ラフロードマップ作成」→「04_OKR作成」へ。
```

## 次に実行
- `02_市場規模推定`


