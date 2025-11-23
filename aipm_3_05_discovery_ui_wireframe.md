# 05 Discovery UI Wireframe

## 実行モード
### 🚀 **複数ペルソナモード**（4ペルソナのストーリーマップがある場合）
4ペルソナ（田中・佐藤・山田・高橋）のストーリーマップ（ST001-ST018）とMVP優先度からUI設計を自動生成し、ペルソナ別・統合UIワイヤーフレームを作成

### 📝 **単一ペルソナモード**（1ペルソナのみの場合）
従来の質問形式で単一ペルソナのUI設計を収集し、画面番号とSTを紐づけてワイヤーフレーム作成

## 複数ペルソナモード - 自動検出対象
- `P001_田中_雅人/story_map_P001.yaml` - AI推進リーダーストーリー
- `P003_山田_健太/story_map_P003.yaml` - 現場PdMストーリー
- `P004_高橋_美咲/story_map_P004.yaml` - コンテンツ運用ストーリー
- `integrated_story_map.yaml` - 統合ストーリー・MVP優先度・スプリント計画
- `integrated_solution_map.yaml` - ソリューション・仮説対応

## 単一ペルソナモード - 質問項目
- 画面数（最大2画面推奨）（必須）
- メイン画面の要素（上から順）（必須）
- 操作フロー（3-5手順）（必須）
- UI/UXの工夫点（必須）
- 参考デザイン（任意）

## 目的
複数ペルソナのストーリー（ST）に対するUI設計を統合的に実施し、MVP優先度に基づく画面設計・操作フロー・ワイヤーフレームを生成。ペルソナ別UI要件・共通プラットフォーム設計を明確化します。

## 実行手順
```yaml
- trigger: "(UIワイヤーフレーム|仮説駆動_UIワイヤーフレーム|HypothesisUIWireframe|UI Wireframe)"
  priority: high
  steps:
    # Step 1: モード判定（複数ペルソナストーリーマップの存在確認）
    - name: "check_story_maps"
      action: "analyze"
      data: [
        "{{find_files(patterns=['**/P001_*/story_map_*.yaml'])}}", 
        "{{find_files(patterns=['**/P003_*/story_map_*.yaml'])}}", 
        "{{find_files(patterns=['**/P004_*/story_map_*.yaml'])}}", 
        "{{find_files(patterns=['**/integrated_story_map.yaml'])}}"
      ]
      instructions: |
        ストーリーマップファイル存在確認結果から実行モードを判定してください：
        - 3つ以上のペルソナストーリーマップ + 統合ファイルが存在: "multi_persona"
        - それ以下の場合: "single_persona"
        
        結果をJSONで返してください：
        {"mode": "multi_persona|single_persona", "found_story_maps": ["P001", "P003", "P004"], "has_integrated": true/false}
      store_as: "execution_mode"

    # === 複数ペルソナモード ===
    - name: "load_story_data"
      condition: "{{execution_mode.mode == 'multi_persona'}}"
      action: "analyze"
      data: [
        "{{read_files(find_files(patterns=['**/P001_*/story_map_*.yaml']))}}",
        "{{read_files(find_files(patterns=['**/P003_*/story_map_*.yaml']))}}",
        "{{read_files(find_files(patterns=['**/P004_*/story_map_*.yaml']))}}",
        "{{read_files(find_files(patterns=['**/integrated_story_map.yaml']))}}",
        "{{read_files(find_files(patterns=['**/integrated_solution_map.yaml']))}}"
      ]
      instructions: |
        4ペルソナのストーリーマップと統合ストーリーからUI設計を生成し、以下の構造で返してください：
        {
          "personas": [
            {
              "id": "P001",
              "name": "田中 雅人",
              "role": "AI推進リーダー",
              "ui_requirements": {
                "primary_screens": [{"screen_id": "SC001", "title": "統合ダッシュボード", "linked_stories": ["ST001", "ST003"]}],
                "key_features": ["ROI可視化", "部下PdM監視", "統合UI/UX"],
                "ux_priorities": ["効率性", "一覧性", "承認フロー"]
              }
            }
          ],
          "shared_ui_components": [
            {"component_id": "COMP001", "name": "統合UI/UXプラットフォーム", "target_stories": ["ST001", "ST006", "ST010"]}
          ],
          "mvp_screen_priorities": [
            {"screen_id": "SC001", "priority": 1, "target_personas": ["P001", "P003", "P004"]}
          ]
        }
      store_as: "multi_persona_ui"

    # === 単一ペルソナモード ===
    - name: "prefill_from_yaml"
      condition: "{{execution_mode.mode == 'single_persona'}}"
      action: "display"
      content: |
        📝 単一ペルソナモードで実行します
        
        🔎 参照
        - story_map.yaml / story_map_mermaid.md
        - spec_mvp.yaml（任意）
    
    - name: "show_ui_patterns"
      action: "display"
      content: |
        🎨 UIパターン例
        - ミニマル型（1タスクだけ）
        - リスト型（チェックボックス）
        - カード型（小さなまとまり）
    
    - name: "collect_ui_info"
      action: "ask_questions_with_template"
      template: |
        === UI設計テンプレート（候補は上に表示） ===
        1. 画面数と役割（既定: 1画面）
        → 【あなたの回答】：
        
        2. メイン画面の要素（上から順に）
        既定候補: タブ(朝/昼/夜) / 主要1アクション / 候補3件 / 3枠表示
        → 【あなたの回答】：
        
        3. 操作フロー（3-5手順）
        既定候補: 表示→今やる→中断3択→完了→3枠更新
        → 【あなたの回答】：
        
        4. UI/UXの工夫
        既定候補: 親指リーチ/静音/遅延<300ms/VoiceOver
        → 【あなたの回答】：
        
        5. 参考デザイン（任意）
        → 【あなたの回答】：
        =====================================
    
    - name: "wait_inputs"
      action: "wait_for_all_answers"
    
    # 自動提案（ストーリー連動の画面番号割当とフロー）
    - name: "auto_propose_screen_map_yaml"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/screen_map_proposed.yaml"
      content: |
        screens:
          - id: SC1
            title: メイン
            links_stories: [ST1, ST2]
          - id: SC2
            title: 詳細/完了
            links_stories: [ST3]
        mapping:
          ST1: SC1
          ST2: SC1
          ST3: SC2
        flows:
          MVP:
            - from: SC1
              action: 今やる
              to: SC2
            - from: SC2
              action: 完了
              to: SC1
          Release1:
            - from: SC1
              action: 時間選択
              to: SC2
        numbering_policy:
          - prefix: SC=Screen
    
    - name: "auto_propose_screen_flow_mermaid"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/screen_flow_proposed_mermaid.md"
      content: |
        ```mermaid
        flowchart LR
          %% MVP
          SC1[SC1 メイン] -->|今やる| SC2[SC2 詳細/完了]
          SC2 -->|完了| SC1
          
          %% Release1
          SC1 -->|時間選択| SC2
        ```
    
    - name: "display_proposal"
      action: "display"
      content: |
        ✍️ 画面番号とストーリーの対応、およびMVP/Releaseごとのスクリーンフロー提案を出力しました。修正点を入力してください（空欄は採用）。
    
    - name: "collect_screen_corrections"
      action: "ask_questions_with_template"
      template: |
        === 修正テンプレ（空欄は採用） ===
        1) スクリーン定義（SCxの追加/名称/links_stories）
        →
        2) ST→SCの対応（例: ST2: SC2）
        →
        3) MVPフロー（from/action/toの追加・変更）
        →
        4) Release1フロー（from/action/toの追加・変更）
        →
        =====================================
    
    - name: "confirm_generate"
      action: "confirm"
      message: "提案＋修正内容で確定成果物（screen_map.yaml / screen_flow.yaml / screen_flow_mermaid.md / design/*.yaml / design/*_wire_aa.md / drawio/*.drawio プレース）を生成します。よろしいですか？"
    
    - name: "export_screen_map_yaml"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/screen_map.yaml"
      content: |
        screens:
          {{screens_final_yaml}}
        mapping:
          {{mapping_final_yaml}}
        numbering_policy:
          - prefix: SC=Screen
    
    - name: "export_screen_flow_yaml"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/screen_flow.yaml"
      content: |
        flows:
          MVP:
            {{flow_mvp_final_yaml}}
          Release1:
            {{flow_release1_final_yaml}}
    
    - name: "export_screen_flow_mermaid"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/screen_flow_mermaid.md"
      content: |
        ```mermaid
        flowchart LR
          %% MVP
          SC1[SC1 {{sc1_title | default:"メイン"}}] -->|今やる| SC2[SC2 {{sc2_title | default:"詳細/完了"}}]
          SC2 -->|完了| SC1
          
          %% Release1
          SC1 -->|時間選択| SC2
        ```
    
    - name: "export_screen_design_specs_SC1"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/design/SC1_design.yaml"
      content: |
        screen: SC1
        title: メイン
        tokens:
          color_primary: "#3b82f6"
          color_bg: "#f8fafc"
          spacing: 8
          radius: 12
          font_size_title: 20
        layout:
          header: {x: 24, y: 16, w: 512, h: 28, text: "今日の一番"}
          input:  {x: 24, y: 56, w: 360, h: 36, placeholder: "タスクを入力"}
          addBtn: {x: 392, y: 56, w: 96,  h: 36, text: "追加"}
          list:   {x: 24, y: 104, w: 464, h: 320}
          modeBtn:{x: 24, y: 440, w: 160, h: 36, text: "仕事モード"}
        links_stories: [ST1, ST2]
        accessibility:
          focus_order: [input, addBtn, list, modeBtn]
          aria:
            - element: modeBtn
              role: button
              aria-pressed: true|false
    
    - name: "export_screen_design_specs_SC2"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/design/SC2_design.yaml"
      content: |
        screen: SC2
        title: 詳細/完了
        tokens:
          color_primary: "#3b82f6"
          color_bg: "#ffffff"
          spacing: 8
          radius: 12
          font_size_title: 18
        layout:
          title:   {x: 24, y: 16, w: 512, h: 24, text: "タスク詳細"}
          detail:  {x: 24, y: 56, w: 464, h: 200}
          doneBtn: {x: 24, y: 264, w: 120, h: 36, text: "完了"}
          backBtn: {x: 160, y: 264, w: 120, h: 36, text: "戻る"}
        links_stories: [ST3]
        accessibility:
          focus_order: [doneBtn, backBtn]
          aria: []
    
    - name: "export_screen_design_aa_SC1"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/design/SC1_wire_aa.md"
      content: |
        ```
        +--------------------------------------------------+
        | 今日の一番                                       |
        +------------------------+-----------+------------+
        | [ タスクを入力........ ] | [追加]    | (仕事モード) |
        +------------------------+-----------+------------+
        | • 候補タスク1                                      |
        | • 候補タスク2                                      |
        | • 候補タスク3                                      |
        +--------------------------------------------------+
        ```
    
    - name: "export_screen_design_aa_SC2"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/design/SC2_wire_aa.md"
      content: |
        ```
        +----------------------------------------------+
        | タスク詳細                                   |
        +----------------------------------------------+
        | [内容.....................................]  |
        |                                              |
        | [ 完了 ]    [ 戻る ]                         |
        +----------------------------------------------+
        ```
    
    - name: "generate_drawio_placeholder"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/drawio/SCREENS_PLACEHOLDER.txt"
      content: |
        各スクリーンのDraw.ioファイル（例: SC1.drawio, SC2.drawio）をこのディレクトリに配置してください。
        別コマンド「設計_Drawioスクリーン生成」で自動生成できます。
    
    - name: "generate_wireframe_md"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/05_UIワイヤーフレーム/ui_wireframe_todo.md"
      content: |
        # アプリ UIワイヤーフレーム
        
        ## 画面一覧とストーリー対応（screen_map.yaml）
        - SC1: ST1, ST2
        - SC2: ST3
        
        ## スクリーンフロー（MVP/Release）
        - MVP/Releaseの画面遷移図は `screen_flow_mermaid.md` を参照（Mermaidコードフェンス付きで出力済み）
        
        ## Draw.io
        - `drawio/SC1.drawio`, `drawio/SC2.drawio` は別コマンドで生成可
    
    - name: "notify_completion"
      action: "display"
      content: |
        ✅ 生成しました：
        - 05_UIワイヤーフレーム/screen_map.yaml / screen_flow.yaml / screen_flow_mermaid.md
        - 05_UIワイヤーフレーム/design/SC1_design.yaml / design/SC2_design.yaml
        - 05_UIワイヤーフレーム/design/SC1_wire_aa.md / design/SC2_wire_aa.md
        - 05_UIワイヤーフレーム/drawio/ 配下のプレースホルダ
        - 05_UIワイヤーフレーム/ui_wireframe_todo.md
```

## 次のコマンド
→ `設計_Drawioスクリーン生成` で `SC1.drawio`/`SC2.drawio` を自動生成
→ その後 `仮説駆動_アプリ骨子生成` でスターターコード生成
