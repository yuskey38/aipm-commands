# : 仮想インタビューログ生成（アイディア発散）

## 目的
既存の仮説・インタビュー設計を踏まえて、発散用の仮想インタビューログを短時間で生成します。アイディエーションの刺激として、複数パターンの回答を出力します（賛同/中立/否定、想定外の視点など）。

## 実行手順（Rules Steps）
```yaml
- trigger: "(sense_仮想インタビュー|virtual_interview|SynthInterview)"
  priority: high
  steps:
    - name: "infer_context"
      action: "analyze"
      data: ["{{thread_messages}}", "{{find_files(patterns=['**/1_sense/03_インタビュー設計/draft_interview_guide.md','**/problem_*.md','**/solution_*.md','**/persona_*.md'])}}"]
      instructions: |
        スレッドと直近のインタビュー設計/課題/ソリューション/ペルソナに関するドキュメントから、以下キーの推定値をJSON1件で返してください（なければ空）。
        keys: goal, target_profile, key_topics
      store_as: "ctx"

    - name: "collect_seeds"
      action: "ask_questions_with_template"
      template: |
        === 仮想インタビュー生成シード ===
        1) 対象者プロフィール（例：共働きママ/学生/営業職 など）
        → {{ctx.target_profile}}
        2) 深掘りトピック（;区切り 3-6件）
        → {{ctx.key_topics}}
        3) 目的（この生成で得たい示唆）
        → {{ctx.goal}}
        4) 生成件数（例：3）
        → 3
        ==================================
      store_as: "seed"

    - name: "confirm"
      action: "confirm"
      message: "指定内容で仮想インタビューログを生成します。よろしいですか？"

    - name: "ensure_output_dir"
      action: "execute_shell"
      command: "mkdir -p {{patterns.flow_date}}/1_sense/90_仮想インタビューログ生成"

    - name: "generate_virtual_logs"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/1_sense/90_仮想インタビューログ生成/virtual_interview_logs.md"
      content: |
        # 仮想インタビューログ（発散用）
        - 目的: {{seed.3}}
        - 対象: {{seed.1}}
        - トピック: {{seed.2}}

        {{#repeat (seed.4 | int | default:3)}}
        ---
        ## セッション{{@index+1}}
        - トーン: {{pick_one '賛同' '中立' '否定' '想定外'}}

        ### Q1: 最近の状況（トピック関連）
        A: 

        ### Q2: 困りごと・頻度・影響
        A: 

        ### Q3: 既存の対処と限界
        A: 

        ### Q4: 期待する体験・価値
        A: 

        ### Q5: 提案した解に対する反応（良い/懸念）
        A: 

        #### インサイト候補（自動案）
        - 
        - 
        {{/repeat}}

    - name: "notify"
      action: "notify"
      message: |
        ✅ 仮想インタビューログを作成しました：
        {{patterns.flow_date}}/1_sense/90_仮想インタビューログ生成/virtual_interview_logs.md
```
