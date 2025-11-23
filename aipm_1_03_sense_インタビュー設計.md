# : インタビュー設計

## 目的
探索的UXリサーチのためのインタビュー設計（目的・対象・質問・運用）を短時間で整えます。

## 実行手順（Rules Steps）
```yaml
- trigger: "(sense_インタビュー設計|インタビュー質問|質問表作成|インタビューガイド)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}", "{{read_files(find_files(patterns=['Flow/**/*.md','Flow/**/*.mdx','**/submit_*/README.md']))}}"]
      instructions: |
        スレッドの直近文脈と同スレッドの成果物から、以下キーの推定値を出力（なければ空）。
        keys: research_goal, target_profile, key_topics, constraints
        出力はJSONのオブジェクト1件のみ。
      store_as: "prefill"
    - name: "collect_design_inputs"
      action: "ask_questions"
      questions:
        - key: "research_goal"
          question: "今回のリサーチ目的（学びたいこと）は？"
          required: true
          default: "{{prefill.research_goal}}"
        - key: "target_profile"
          question: "対象者プロフィール（例：属性/利用状況/経験年数）"
          required: true
          default: "{{prefill.target_profile}}"
        - key: "key_topics"
          question: "深掘りしたいトピック（; 区切りで3-6件）"
          required: true
          default: "{{prefill.key_topics}}"
        - key: "constraints"
          question: "制約（時間/環境/録音可否/倫理配慮など）"
          required: false
          default: "{{prefill.constraints}}"
      store_as: "iv"

    - name: "confirm_create"
      action: "confirm"
      message: "入力に基づきインタビューガイドを作成します。よろしいですか？"

    - name: "ensure_output_dir"
      action: "execute_shell"
      command: "mkdir -p {{patterns.flow_date}}/1_sense/03_インタビュー設計"

    - name: "create_interview_guide"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/1_sense/03_インタビュー設計/draft_interview_guide.md"
      content: |
        # インタビューガイド
        - 目的: {{iv.research_goal}}
        - 対象: {{iv.target_profile}}
        - 制約: {{iv.constraints}}

        ## セクション/質問（雛形）
        1) 導入・同意取得
        2) 現状把握（行動/ツール/頻度）
        3) 課題・痛点（具体事例/頻度/影響）
        4) 既存解決策の評価（良い点/不満/代替手段）
        5) 期待する体験・価値
        6) クロージング（追加事項/謝辞）

        ### 深掘りトピック
        {{iv.key_topics}}

    - name: "notify"
      action: "notify"
      message: |
        ✅ インタビューガイドを作成しました：
        {{patterns.flow_date}}/draft_interview_guide.md
        次は「05_リクルーティング計画」へ。
```

## 次に実行
- `05_リクルーティング計画`
