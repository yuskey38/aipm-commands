# 06 設計: Drawioスクリーン生成（AIPMハッカソン）

## 目的
`design/SCx_design.yaml`/`design/SCx_wire_aa.md` を参照しつつ、実際のDraw.io形式（.drawio, XML）のスクリーンファイルを生成します。

## 実行手順
```yaml
- trigger: "(設計_Drawioスクリーン生成|GenerateDrawioScreens)"
  priority: high
  steps:
    - name: "ask_screen_id"
      action: "ask_question"
      question: "生成するスクリーンIDは？（SC1 または SC2）"
      required: true
      store_as: "screen_id"

    - name: "confirm_generate"
      action: "confirm"
      message: "{{screen_id}} の .drawio 図面を生成します。よろしいですか？"

    # --- SC1.drawio を生成 ---
    - name: "generate_sc1_drawio"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/06_Drawioスクリーン生成/drawio/SC1.drawio"
      content: |
        <mxfile host="app.diagrams.net" type="device">
          <diagram name="SC1" id="sc1-diagram">
            <mxGraphModel dx="1280" dy="720" grid="1" gridSize="10" guides="1" tooltips="1" connect="1" arrows="1" fold="1" page="1" pageScale="1" pageWidth="1240" pageHeight="1754" background="#ffffff">
              <root>
                <mxCell id="0"/>
                <mxCell id="1" parent="0"/>
                <!-- Screen canvas -->
                <mxCell id="c1" value="SC1 メイン" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#FFFFFF;strokeColor=#000000;" vertex="1" parent="1">
                  <mxGeometry x="40" y="40" width="600" height="400" as="geometry"/>
                </mxCell>
                <!-- Header -->
                <mxCell id="h1" value="今日の一番" style="text;whiteSpace=wrap;html=1;align=left;verticalAlign=middle;fontSize=20;" vertex="1" parent="c1">
                  <mxGeometry x="20" y="16" width="560" height="30" as="geometry"/>
                </mxCell>
                <!-- Input -->
                <mxCell id="inp" value="タスクを入力" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#F8FAFC;strokeColor=#CBD5E1;" vertex="1" parent="c1">
                  <mxGeometry x="20" y="60" width="360" height="36" as="geometry"/>
                </mxCell>
                <!-- Add Button -->
                <mxCell id="add" value="追加" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#3B82F6;fontColor=#FFFFFF;strokeColor=#3B82F6;" vertex="1" parent="c1">
                  <mxGeometry x="390" y="60" width="90" height="36" as="geometry"/>
                </mxCell>
                <!-- Mode Button -->
                <mxCell id="mode" value="仕事モード" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#E2E8F0;strokeColor=#94A3B8;" vertex="1" parent="c1">
                  <mxGeometry x="20" y="110" width="160" height="34" as="geometry"/>
                </mxCell>
                <!-- List -->
                <mxCell id="list" value="リスト" style="shape=swimlane;startSize=24;rounded=1;whiteSpace=wrap;html=1;fillColor=#FFFFFF;strokeColor=#CBD5E1;" vertex="1" parent="c1">
                  <mxGeometry x="20" y="156" width="560" height="260" as="geometry"/>
                </mxCell>
              </root>
            </mxGraphModel>
          </diagram>
        </mxfile>

    # --- SC2.drawio を生成 ---
    - name: "generate_sc2_drawio"
      action: "create_markdown_file"
      path: "Flow/{{today}}/{{flow_dir}}/06_Drawioスクリーン生成/drawio/SC2.drawio"
      content: |
        <mxfile host="app.diagrams.net" type="device">
          <diagram name="SC2" id="sc2-diagram">
            <mxGraphModel dx="1280" dy="720" grid="1" gridSize="10" guides="1" tooltips="1" connect="1" arrows="1" fold="1" page="1" pageScale="1" pageWidth="1240" pageHeight="1754" background="#ffffff">
              <root>
                <mxCell id="0"/>
                <mxCell id="1" parent="0"/>
                <!-- Screen canvas -->
                <mxCell id="c2" value="SC2 詳細/完了" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#FFFFFF;strokeColor=#000000;" vertex="1" parent="1">
                  <mxGeometry x="40" y="40" width="600" height="400" as="geometry"/>
                </mxCell>
                <!-- Title -->
                <mxCell id="t2" value="タスク詳細" style="text;whiteSpace=wrap;html=1;align=left;verticalAlign=middle;fontSize=18;" vertex="1" parent="c2">
                  <mxGeometry x="20" y="16" width="560" height="28" as="geometry"/>
                </mxCell>
                <!-- Detail Area -->
                <mxCell id="d2" value="内容" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#FFFFFF;strokeColor=#CBD5E1;" vertex="1" parent="c2">
                  <mxGeometry x="20" y="56" width="560" height="200" as="geometry"/>
                </mxCell>
                <!-- Done Button -->
                <mxCell id="done" value="完了" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#3B82F6;fontColor=#FFFFFF;strokeColor=#3B82F6;" vertex="1" parent="c2">
                  <mxGeometry x="20" y="266" width="120" height="36" as="geometry"/>
                </mxCell>
                <!-- Back Button -->
                <mxCell id="back" value="戻る" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#E2E8F0;strokeColor=#94A3B8;" vertex="1" parent="c2">
                  <mxGeometry x="160" y="266" width="120" height="36" as="geometry"/>
                </mxCell>
              </root>
            </mxGraphModel>
          </diagram>
        </mxfile>

    - name: "display_result"
      action: "display"
      content: |
        ✅ 生成しました：
        - `06_Drawioスクリーン生成/drawio/SC1.drawio`
        - `06_Drawioスクリーン生成/drawio/SC2.drawio`
        
        片方のみが必要な場合は、対象 `screen_id` を指定して再実行してください（両方生成されていても問題ありません）。
```
