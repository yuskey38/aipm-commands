# 01 Hackathon Marp Slide Generation

## 目的
仮説駆動開発の一連の流れ（ペルソナ→課題→ソリューション→MVP→検証）を、ストーリー性のある魅力的なプレゼンテーションとして自動生成します。実際の成果物から内容を抽出し、エモーショナルなアウトカムを強調します。

## 実行手順
```yaml
- trigger: "(発表_Marpスライド生成|CreateMarpSlides)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns:['**/persona_*.md','**/problem_*.md','**/solution_map*.md','**/story_map*.md','**/dev_tasks_order*.md','**/progress_report.md']))}}"]
      instructions: |
        タイトル/発表者/最重要メッセージ/デモ形式/想定時間の候補を抽出して提示してください。
      store_as: "auto_marp_meta"
    - name: "prefill_marp_meta"
      action: "display"
      content: |
        🔎 自動抽出
        {{auto_marp_meta}}
    # 成果物の読み込み（現在の構造に対応）
    - name: "read_artifacts"
      action: "read_multiple_files"
      files:
        - "Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/total_development_spec.md"
        - "Flow/{{today}}/{{flow_dir}}/01_ペルソナ作成/persona_todo.md"
        - "Flow/{{today}}/{{flow_dir}}/01_ペルソナ作成/experience_map.yaml"
        - "Flow/{{today}}/{{flow_dir}}/02_課題定義/problem_map.yaml"
        - "Flow/{{today}}/{{flow_dir}}/02_課題定義/customer_problem_map.yaml"
        - "Flow/{{today}}/{{flow_dir}}/03_ソリューションマップ/solution_map.yaml"
        - "Flow/{{today}}/{{flow_dir}}/04_ストーリーマップ/story_map.yaml"
        - "Flow/{{today}}/{{flow_dir}}/07_開発タスク分解/dev_tasks.yaml"
        - "Flow/{{today}}/{{flow_dir}}/10_タスクリファイン/progress_report.md"
        - "Flow/{{today}}/{{flow_dir}}/dev/src/index.html"
        - "Flow/{{today}}/{{flow_dir}}/dev/src/styles.css"
        - "Flow/{{today}}/{{flow_dir}}/dev/src/app.js"
    
    - name: "collect_meta"
      action: "ask_questions_with_template"
      template: |
        === スライド作成メタ ===
        1. 発表タイトル（例：5分で"次の一手"が出るTODOで、ママの毎日に余白を）
        → 【あなたの回答】：
        
        2. 発表者（GitHub ID / 氏名）
        → 【あなたの回答】：
        
        3. 想定時間（分）例: 5
        → 【あなたの回答】：
        
        4. デモの見せ方（ライブ実演/録画/スクショ）
        → 【あなたの回答】：
        
        5. 最も伝えたいメッセージ（例: AIが判断を代行→ママに時間の余白を）
        → 【あなたの回答】：
        =====================================
    
    - name: "wait_for_all_answers"
      action: "wait_for_all_answers"
    
    - name: "generate_marp"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/11_Marpスライド生成/slides_mvp.marp.md"
      template_reference: "aipm_hackathon/11_Marpスライド生成.md => marp_template"
    
    - name: "notify"
      action: "display"
      content: |
        ✅ Marpスライドを生成しました：
        `Flow/{{today}}/{{flow_dir}}/11_Marpスライド生成/slides_mvp.marp.md`
        
        Marp for VS Code 拡張機能でプレビュー、またはPDF/PPTXエクスポートできます。
        
        📁 提出方法（ZIP不要）
        - 下記フォルダをそのままGoogle Driveにドラッグ＆ドロップでアップロードしてください：
          `Flow/{{today}}/{{flow_dir}}/11_Marpスライド生成/`
        - 画像や追加素材がある場合は、このフォルダ内に配置してください
        
        📱 デモ用ファイル:
        - 実装コード: `Flow/{{today}}/{{flow_dir}}/dev/src/index.html`
        - ライブデモ: `dev/src/index.html` をブラウザで開いて実演
        
        必要に応じて画像やスクリーンショットを追加してください。

    # フォルダ提出（ZIP不要）を支援：Finder/Explorerを開く
    - name: "open_slide_folder"
      action: "execute_shell"
      command: "open Flow/{{today}}/{{flow_dir}}/11_Marpスライド生成"
      message: "スライド生成フォルダを開きます（ZIPせずフォルダのままアップロード）。"

    - name: "display_folder_upload_tips"
      action: "display"
      content: |
        📤 フォルダごと提出ガイド
        1. 開いたフォルダをGoogle Driveの提出先にドラッグ＆ドロップ
        2. フォルダ名にチーム名・氏名を含めると運営が識別しやすくなります
        3. 共同編集する場合はDrive上で権限を「編集可」に設定してください
```

## Marpテンプレート
```marp_template
---
marp: true
theme: default
paginate: true
backgroundColor: #fafafa
style: |
  section.lead {
    text-align: center;
  }
  section.impact {
    background-color: #1e293b;
    color: white;
  }
  blockquote {
    border-left: 4px solid #3b82f6;
    padding-left: 1em;
    font-style: italic;
  }
  .emoji-large {
    font-size: 3em;
  }
  .value-flow {
    font-size: 1.8em;
    line-height: 2;
  }
---

<!-- _class: lead -->

# {{title}}

### {{key_message}}

<br>

**{{presenter}}**
{{event_date}} {{event_name}}

---

<!-- _class: impact -->

# {{pain_emoji}} {{pain_moment}}

> {{pain_quote}}

**{{pain_impact}}**

---

# {{persona_emoji}} ペルソナ: {{persona.name}}

{{#persona_bullets}}
- {{.}}
{{/persona_bullets}}

<div class="emoji-large" style="text-align: center; margin-top: 1em;">
{{persona_icon_set}}
</div>

---

# 📊 発見した{{problems.length}}つの課題

{{#problems}}
### {{emoji}} **{{id}}: {{title}}** 
- {{description}}（{{frequency}}）
{{/problems}}

---

# 💡 ソリューション仮説

> **{{solution_hypothesis}}**

<br>

### 実装した{{core_features.length}}つのコア機能
{{#core_features}}
{{index}}. **{{emoji}} {{name}}**（{{description}}）
{{/core_features}}

---

<!-- _class: lead -->

# 🚀 デモ

### **{{demo_title}}**

{{demo_type}}

---

# 📱 実際の画面

{{#has_screenshot}}
![bg right:50% fit]({{screenshot_path}})
{{/has_screenshot}}

### {{main_feature_title}}
{{#main_feature_points}}
- {{.}}
{{/main_feature_points}}
- **{{main_feature_impact}}**

---

# 🎯 エモーショナルなアウトカム

## Before {{before_emoji}}
{{#before_states}}
- {{.}}
{{/before_states}}

## After {{after_emoji}}
{{#after_states}}
- **{{.}}**
{{/after_states}}

---

# 📈 初期検証の結果

### 定量効果
{{#quantitative_results}}
- {{check}} {{metric}}: **{{result}}**（{{status}}）
{{/quantitative_results}}

### 定性フィードバック
{{#qualitative_feedback}}
> {{.}}
{{/qualitative_feedback}}

---

# 🔬 仮説検証として学んだこと

### ✅ 検証できた仮説
{{#validated_hypothesis}}
- **{{title}}**{{impact}}
{{/validated_hypothesis}}

### 🤔 新たな問い
{{#new_questions}}
- {{.}}
{{/new_questions}}

---

# 🎯 MVPで最も大切にしたこと

<div class="value-flow" style="text-align: center; margin: 1em 0;">

{{#value_flow}}
**{{emoji}} {{step}}**
{{#not_last}}↓{{/not_last}}
{{/value_flow}}

</div>

---

# 🚀 Next Step

### MVP2で検証したいこと
{{#mvp2_items}}
{{index}}. **{{title}}** - {{description}}
{{/mvp2_items}}

### 技術的チャレンジ
{{#tech_challenges}}
- {{.}}
{{/tech_challenges}}

---

<!-- _class: lead -->

# 🙏 Thank you!

### {{key_message}}

<br>

**成果物**: `dev/src/` フォルダ
**GitHub**: {{github_url}}
**ライブデモ**: `dev/src/index.html`

<br>

<div class="emoji-large">
{{outcome_flow_emojis}}
</div>

---

# 📎 Appendix: 開発プロセス

{{#dev_process}}
{{index}}. **{{phase}}** → {{outcome}}
{{/dev_process}}

詳細: `Flow/{{today}}/{{flow_dir}}/`
実装: `Flow/{{today}}/{{flow_dir}}/dev/src/`

```

## スライドカスタマイズのヒント

### 画像の追加
```markdown
![bg right:40%](path/to/image.png)  # 右側40%に背景画像
![w:300](path/to/image.png)         # 幅300pxで挿入
```

### スタイルクラス
- `lead`: 中央寄せスライド
- `impact`: ダークテーマスライド

### エクスポート
1. VS Code: Marp for VS Code拡張機能
2. Export: PDF / PPTX / HTML

## 備考
- ペルソナ、課題、ソリューションの具体的な内容は自動的に成果物から抽出されます
- 実装コードは `dev/src/` フォルダから自動参照されます
- スクリーンショットは `dev/assets/` フォルダに配置を推奨
- デモは実演/録画/静止画から選択可能
- ライブデモ時は `dev/src/index.html` をブラウザで開いて実演
