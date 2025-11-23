# : スプリントレビュー

## 目的
Delivery2 スプリントの結果をレビューし、完了項目/インペディメント/改善点を整理します。Flow内の素材も活用します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(delivery2_スプリントレビュー|Sprint Review 2)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/daily_tasks.md','**/delivery2_development_review.md','**/delivery2_development_plan.md']))}}"]
      instructions: |
        直近2週間の完了項目とインペディメントの候補を抽出し、display用に簡潔にまとめてください。
      store_as: "auto_review_seed"
    - name: "prefill_seed"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_review_seed}}
    - collect_flow_materials:
        query:
          glob: "{{patterns.daily_tasks_glob}}"
          lookback_days: 14
        store_as: daily_pool
    - transform:
        source: "{{daily_pool}}"
        script: |
          import re, json
          done = set(); imped = []
          for path,txt in source:
              for line in txt.splitlines():
                  m = re.match(r"- \[x\] (US-[0-9]+)", line, re.I)
                  if m: done.add(m.group(1))
                  if "⚠" in line or "impediment" in line.lower():
                      imped.append(line.lstrip("- "))
          print(json.dumps({
            "demo_items": "\n".join(f"- {d}" for d in sorted(done)),
            "impediments": "\n".join(f"- {i}" for i in imped)
          }))
        store_as: auto_data
    - prefill_question_answers:
        target_questions: "pmbok_executing.mdc => sprint_review_questions"
        source: "{{auto_data}}"
    - ask_unfilled_questions:
        message: "Delivery2 Sprint Review に不足している情報を入力してください。"
    - confirm: "入力＋自動集約で Delivery2 Sprint Review を Flow に作成します。よろしいですか？"
    - create_markdown_file:
        path: "{{patterns.flow_date}}/delivery2_sprint_review_{{sprint_id}}.md"
        template_reference: "basic/pmbok_executing.mdc => sprint_review_template"
    - notify: |
        ✅ 生成しました：{{patterns.flow_date}}/delivery2_sprint_review_{{sprint_id}}.md
```

## 次に実行
- 次スプリント計画 or Backlog2調整
