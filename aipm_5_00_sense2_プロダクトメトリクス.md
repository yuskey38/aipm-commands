# Strategy: プロダクトメトリクス設計（Delivery計測）

## 目的
Deliveryで公開したサービス/機能に対し、ビジネスとユーザー価値に直結する計測指標（North Star＋先行/遅行、ファネル/KPI/イベント）を短時間で設計し、実装に渡せる形で出力します。

## 実行手順（Rules Steps）
```yaml
- trigger: "(strategy_プロダクトメトリクス|メトリクス設計|計測設計|KPI設計)"
  priority: high
  steps:
    - name: "infer_defaults_from_thread"
      action: "analyze"
      data: ["{{thread_messages}}","{{read_files(find_files(patterns=['**/delivery2_development_plan.md','**/strategy_roadmap.yaml','**/sense2_research_summary.md','**/check_*.md']))}}"]
      instructions: |
        現在の対象機能/アウトカム/主要行動/ログ基盤の候補を抽出し、各1行で表示用にまとめてください。
      store_as: "auto_metrics_context"
    - name: "prefill_context_display"
      action: "display"
      content: |
        🔎 自動抽出（候補）
        {{auto_metrics_context}}
    - name: "collect_context"
      action: "ask_questions_with_template"
      template: |
        === 対象コンテキスト ===
        1) 対象プロダクト/機能（リリース名・日付）
        →
        2) 事業ゴール/学びたい仮説（1-3文）
        →
        3) 主要ユーザー行動（AARRRやHEART等、該当する段）
        →
        4) 技術的制約/ログ基盤（例：GA4/Amplitude/自前BE/Front-end埋め込み）
        →
        =======================

    - name: "wait"
      action: "wait_for_all_answers"

    - name: "confirm"
      action: "confirm"
      message: "入力に基づきメトリクス設計ドラフトを作成します。よろしいですか？"

    - name: "create_metrics_draft"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/strategy_product_metrics.md"
      content: |
        # プロダクトメトリクス設計
        - 対象: {{context.1}}
        - 事業ゴール/学びたい仮説: {{context.2}}
        - 主要ユーザー行動: {{context.3}}
        - ログ基盤/制約: {{context.4}}

        ## North Star Metric（NSM）
        - 候補: 
        - 根拠（価値と相関/将来価値の代理性）:

        ## 先行指標（Leading）と遅行指標（Lagging）
        - 先行指標（意思決定の早期検出に有効な1-3指標）:
        - 遅行指標（成果/継続価値の確認に有効な1-3指標）:

        ## ファネル（AARRR 例）
        - Acquisition: 
        - Activation: 
        - Retention: 
        - Revenue: 
        - Referral: 

        ## KPIとイベント定義（実装渡し用）
        | 名称 | 目的/意味 | 計算式/トリガ | パラメータ | 計測粒度 | 可視化先 |
        | --- | --- | --- | --- | --- | --- |
        |  |  |  |  |  |  |

        ```mermaid
        flowchart LR
          A[User Event] -->|track| B((Ingest)) --> C[Warehouse/Analytics]
          C --> D[Dashboard]
          C --> E[Alert/Experiment]
        ```

        ## アクション
        - 計測実装タスク（FE/BE/ETL/ダッシュボード）
        - 閾値/アラート設計
        - 実験設計（必要なら）

    - name: "create_instrumentation_snippets"
      action: "create_markdown_file"
      path: "{{patterns.flow_date}}/strategy_instrumentation_snippets.md"
      content: |
        # 計測スニペット（雛形）
        ```html
        <script>
          // Page View
          window.analytics?.track('page_view', {
            path: location.pathname,
            ts: Date.now()
          });

          // CTA Click
          document.querySelectorAll('[data-cta]').forEach(el => {
            el.addEventListener('click', () => {
              window.analytics?.track('cta_click', {
                id: el.getAttribute('data-cta'),
                path: location.pathname,
                ts: Date.now()
              });
            });
          });
        </script>
        ```

        ```javascript
        // Node/BE 例: event enqueue
        enqueue('signup_completed', { userId, plan, ts: Date.now() });
        ```

    - name: "notify"
      action: "notify"
      message: |
        ✅ メトリクス設計ドラフトを作成しました：
        - {{patterns.flow_date}}/strategy_product_metrics.md
        - {{patterns.flow_date}}/strategy_instrumentation_snippets.md
        Deliveryの実装タスクへ連携し、ダッシュボード整備まで進めましょう。
```

## 次に実行
- `4_delivery/07_開発タスク分解` に計測実装タスクを追加
- 実装後は `Experiment`/`Dashboard` 整備へ

