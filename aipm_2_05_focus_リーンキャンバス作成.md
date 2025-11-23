# : リーンキャンバス作成（Sense/Focus仮説の統合）

## 目的
Senseでの調査・UXリサーチ、Focusでの定義/市場規模/ロードマップ/OKRを踏まえ、仮説をリーンキャンバスに統合します。YAMLとMermaidで出力し、後続のDelivery計画の基礎にします。

## 実行手順（Rules Steps）
```yaml
- trigger: "(focus_リーンキャンバス|Lean Canvas|LC作成)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}", "{{read_files(find_files(patterns=['Flow/**/*.md','Flow/**/*.mdx','**/submit_*/README.md']))}}"]
      instructions: |
        スレッド文脈と成果物から、以下キーの推定値を返してください（なければ空）。
        keys: opportunities_path, product_def_path, market_size_path, research_summary_path,
              problem_1, problem_2, problem_3, customersegments, uniquevalueproposition,
              solution_1, solution_2, solution_3, channels, revenuestreams, coststructure,
              keymetrics, unfairadvantage
        出力はJSONのオブジェクト1件のみ。
      store_as: "prefill"
    - name: "collect_sources"
      action: "ask_questions"
      questions:
        - key: "opportunities_path"
          question: "sense_opportunities.yaml のパス（任意。未指定ならスキップ）"
          required: false
          default: "{{prefill.opportunities_path}}"
        - key: "product_def_path"
          question: "focus_product_definition.md のパス（任意）"
          required: false
          default: "{{prefill.product_def_path}}"
        - key: "market_size_path"
          question: "focus_market_size_estimation.md のパス（任意）"
          required: false
          default: "{{prefill.market_size_path}}"
        - key: "research_summary_path"
          question: "draft_research_summary.md のパス（任意）"
          required: false
          default: "{{prefill.research_summary_path}}"
      store_as: "src"

    - name: "ask_canvas_items"
      action: "ask_questions_with_template"
      template: |
        === リーンキャンバス入力（必要箇所のみ、空欄は後で補完） ===
        Problem（顧客の主要課題。上位3つまで）
        → {{prefill.problem_1}}; {{prefill.problem_2}}; {{prefill.problem_3}}
        CustomerSegments（主要セグメント/早期採用者）
        → {{prefill.customersegments}}
        UniqueValueProposition（UVP・一言サマリ/差別化ポイント）
        → {{prefill.uniquevalueproposition}}
        Solution（主要課題に対する上位3つの解決策）
        → {{prefill.solution_1}}; {{prefill.solution_2}}; {{prefill.solution_3}}
        Channels（獲得チャネル）
        → {{prefill.channels}}
        RevenueStreams（収益の流れ）
        → {{prefill.revenuestreams}}
        CostStructure（主要コスト）
        → {{prefill.coststructure}}
        KeyMetrics（成功を測る1-3の重要指標）
        → {{prefill.keymetrics}}
        UnfairAdvantage（模倣困難な優位性）
        → {{prefill.unfairadvantage}}
        =====================================================
      store_as: "lc"

    - name: "confirm"
      action: "confirm"
      message: "リーンキャンバス（YAML/Mermaid）を生成します。よろしいですか？"

    - name: "create_lean_canvas_yaml"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/focus_lean_canvas.yaml"
      content: |
        lean_canvas:
          sources:
            opportunities: {{src.opportunities_path}}
            product_definition: {{src.product_def_path}}
            market_size: {{src.market_size_path}}
            research_summary: {{src.research_summary_path}}
          problem:
            - {{lc.problem_1}}
            - {{lc.problem_2}}
            - {{lc.problem_3}}
          customer_segments:
            - {{lc.customersegments}}
          unique_value_proposition: {{lc.uniquevalueproposition}}
          solution:
            - {{lc.solution_1}}
            - {{lc.solution_2}}
            - {{lc.solution_3}}
          channels:
            - {{lc.channels}}
          revenue_streams:
            - {{lc.revenuestreams}}
          cost_structure:
            - {{lc.coststructure}}
          key_metrics:
            - {{lc.keymetrics}}
          unfair_advantage: {{lc.unfairadvantage}}

    - name: "create_lean_canvas_mermaid"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/focus_lean_canvas_mermaid.md"
      content: |
        ```mermaid
        flowchart LR
          subgraph P[Problem]
            PR1[{{lc.problem_1}}]
            PR2[{{lc.problem_2}}]
            PR3[{{lc.problem_3}}]
          end
          subgraph CS[Customer Segments]
            SEG[{{lc.customersegments}}]
          end
          subgraph UVP[Unique Value Proposition]
            U[{{lc.uniquevalueproposition}}]
          end
          subgraph SOL[Solution]
            S1[{{lc.solution_1}}]
            S2[{{lc.solution_2}}]
            S3[{{lc.solution_3}}]
          end
          subgraph CH[Channels]
            C[{{lc.channels}}]
          end
          subgraph REV[Revenue Streams]
            R[{{lc.revenuestreams}}]
          end
          subgraph COST[Cost Structure]
            CO[{{lc.coststructure}}]
          end
          subgraph KM[Key Metrics]
            K[{{lc.keymetrics}}]
          end
          subgraph UA[Unfair Advantage]
            UA1[{{lc.unfairadvantage}}]
          end

          PR1 & PR2 & PR3 --> U --> S1 & S2 & S3
          SEG --> U
          U --> K
          S1 & S2 & S3 --> R
          S1 & S2 & S3 --> C
          U -.影響.-> CO
        ```

    - name: "notify"
      action: "notify"
      message: |
        ✅ 生成しました：
        - {{patterns.flow_date}}/focus_lean_canvas.yaml
        - {{patterns.flow_date}}/focus_lean_canvas_mermaid.md
        Delivery計画（開発タスク分解/スコープ定義）に活用してください。
```

## 次に実行
- `4_delivery/07_開発タスク分解`（リーンキャンバスのProblem→Solution→Storyへ展開）
