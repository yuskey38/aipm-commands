# : リクルーティング計画

## 目的
対象者の定義・募集チャネル・スクリーニング条件・スケジュール・インセンティブ等を整理し、効率的な被験者募集を実行します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(sense_リクルーティング|ユーザー募集|被験者募集|参加者募集)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}", "{{read_files(find_files(patterns=['Flow/**/*.md','Flow/**/*.mdx','**/submit_*/README.md']))}}"]
      instructions: |
        スレッドの直近文脈と同スレッドの成果物から、以下キーの推定値を出力（なければ空）。
        keys: target_definition, channels, screening, slots, schedule, incentive
        出力はJSONのオブジェクト1件のみ。
      store_as: "prefill"
    - name: "collect_recruiting_info"
      action: "ask_questions"
      questions:
        - key: "target_definition"
          question: "対象者定義（属性/行動/経験）"
          required: true
          default: "{{prefill.target_definition}}"
        - key: "channels"
          question: "募集チャネル（例：社内/顧客/掲示板/パネル）"
          required: true
          default: "{{prefill.channels}}"
        - key: "screening"
          question: "スクリーニング条件（必須/望ましい）"
          required: true
          default: "{{prefill.screening}}"
        - key: "slots"
          question: "必要人数/枠（例：5-8）"
          required: true
          default: "{{prefill.slots}}"
        - key: "schedule"
          question: "想定スケジュール（期間/曜日/時間）"
          required: true
          default: "{{prefill.schedule}}"
        - key: "incentive"
          question: "インセンティブ（例：Amazon券/謝礼/社内ポイント）"
          required: false
          default: "{{prefill.incentive}}"
      store_as: "rc"

    - name: "confirm"
      action: "confirm"
      message: "入力に基づきリクルーティング計画を作成します。よろしいですか？"

    - name: "ensure_output_dir"
      action: "execute_shell"
      command: "mkdir -p {{patterns.flow_date}}/1_sense/04_リクルーティング計画"

    - name: "create_plan"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/1_sense/04_リクルーティング計画/draft_recruiting_plan.md"
      content: |
        # リクルーティング計画
        - 対象定義: {{rc.target_definition}}
        - 募集チャネル: {{rc.channels}}
        - スクリーニング: {{rc.screening}}
        - 必要枠: {{rc.slots}}
        - スケジュール: {{rc.schedule}}
        - インセンティブ: {{rc.incentive}}

        ## 連絡テンプレ（雛形）
        - 件名: ユーザーインタビューご協力のお願い
        - 本文: 目的/所要時間/謝礼/候補日程/同意事項

    - name: "notify"
      action: "notify"
      message: |
        ✅ リクルーティング計画を作成しました：
        {{patterns.flow_date}}/draft_recruiting_plan.md
        次は「06_インタビュー分析（個別）」へ。
```

## 次に実行
- `06_インタビュー分析（個別）`
