# : OKR作成

## 目的
優先度付けされたフォーカスに基づき、最初の一歩としてのOKRを定義します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(focus_OKR作成|OKR Draft)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}", "{{read_files(find_files(patterns=['Flow/**/*.md','Flow/**/*.mdx','**/submit_*/README.md']))}}"]
      instructions: |
        スレッド文脈から、以下の推定値を返してください（なければ空）。
        keys: program, period, owner, phase, objectives, krs
        出力はJSONのオブジェクト1件のみ。
      store_as: "prefill"
    - name: "collect_okr_inputs"
      action: "ask_questions"
      questions:
        - key: "program"
          question: "プログラム/プロジェクト名は？"
          required: true
          default: "{{prefill.program}}"
        - key: "period"
          question: "期間は？（例：2025-Q4）"
          required: true
          default: "{{prefill.period}}"
        - key: "owner"
          question: "OKRオーナーは？"
          required: true
          default: "{{prefill.owner}}"
        - key: "phase"
          question: "フェーズは？（立ち上げ/改善/拡張）"
          required: true
          default: "{{prefill.phase}}"
        - key: "objectives"
          question: "Objectivesを1〜3件で入力してください（; 区切り）"
          required: true
          default: "{{prefill.objectives}}"
        - key: "krs"
          question: "各ObjectiveのKey Resultsを入力してください（Objective順に | 区切り、各KRは ; 区切りで）"
          required: true
          default: "{{prefill.krs}}"
      store_as: "okr_params"

    - name: "confirm_creation"
      action: "confirm"
      message: "以下の内容でOKRドラフトを作成します：\n\nプログラム: {{okr_params.program}}\n期間: {{okr_params.period}}\nオーナー: {{okr_params.owner}}\nフェーズ: {{okr_params.phase}}\n\n実行してよろしいですか？"

    - name: "create_draft"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/focus_okr_{{okr_params.period | slugify}}.md"
      template_reference: "basic/03_pmbok_planning.mdc => okr_template"
```

## 次に実行
- 提出・実装フェーズへ移行


