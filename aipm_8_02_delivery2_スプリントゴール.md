# : スプリントゴール

## 目的
二段階目Delivery用のスプリントゴールを定義します。Discovery2/Backlog2に基づき、測定可能な目標と成功基準を明確にします。

## 実行手順（Rules Steps）
```yaml
- trigger: "(delivery2_スプリントゴール|Sprint Goal 2)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/backlog/backlog2.yaml','**/strategy_roadmap.yaml','**/focus2_wbs.md']))}}"]
      instructions: |
        今スプリントの候補目標/成功基準/測定指標を抽出し、質問のデフォルト想定として箇条書き出力してください。
      store_as: "auto_sprint_goal_seed"
    - name: "prefill_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_sprint_goal_seed}}
    - name: "start_info_collection"
      action: "call basic/pmbok_executing.mdc => sprint_goal_questions"
      message: "Delivery2の【スプリントゴール】作成に必要な情報を収集します。"
    - name: "wait_user_responses"
      action: "wait_for_all_answers"
      message: "必要な回答が揃うまで先に進みません。"
    - name: "confirm_document_creation"
      action: "confirm"
      message: "入力内容でスプリントゴールを作成します。よろしいですか？"
    - name: "create_draft"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/delivery2_sprint_goal_{{sprint_number}}.md"
      template_reference: "basic/pmbok_executing.mdc => sprint_goal_template"
    - name: notify
      action: notify
      message: |
        Delivery2 スプリントゴールのドラフトを Flow に生成しました：
        - {{patterns.flow_date}}/delivery2_sprint_goal_{{sprint_number}}.md
```

## 次に実行
- `03_スプリントレビュー`
