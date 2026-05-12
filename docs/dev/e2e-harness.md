# E2E verification harness

`agent-server/tools/e2e/` は **Light / Light v2 / Deep agent の調整 phase verification run** 用の scenario runner です（[tealus #262](https://github.com/gamasenninn/tealus/issues/262)、v0.2.3〜）。CI gate ではなく、fix の前後で「scenario set を 1 回流して挙動を verify する」用途です。

## 設計思想

- 既存の log + chat 履歴を観察 surface として活用（新 infra を増やさない）
- 実環境 path を全通す（CLI が Tealus API 経由で test room に投下）
- 専用 test bot user + 専用 sandbox room で隔離（本番 DB / 履歴を汚さない）
- TTS は test room 設定で disable（Aivis 課金回避）
- sequential 実行（TTS / mcp_cache 干渉回避）

## 3 layer judgment 構造

LLM 系 E2E は **決定論前提では機能しない** という観察に基づき、判定を 3 layer に分けています。

| layer | 判定対象 | 結果 | 例 |
|-------|---------|------|-----|
| **決定論層** | tool chain shape、log line の存在、HTTP status 等 | **fail** で stop | 「`list_rooms` → `search_messages` の順で呼ばれたか」「`extraction_method=vision_gemini` log があるか」 |
| **観察層** | token 使用量、latency、**LLM-as-judge による semantic correctness 採点** | **warn-only**（fail にはしない） | 「応答 token が baseline ±50% 内か」「judge score が `min_score` 以上か」 |
| **人 review 層** | manual_check field の検査項目（最終応答の妥当性、tone、画像 quality 等） | report に列挙して人が読む | 「応答に hallucinated link が含まれていないか」「画像が prompt と整合するか」 |

!!! info "なぜ 3 layer か"
    LLM 応答は同じ prompt でも **quality variance**（1 件 / 8+ 件 / 13 件など）を持ちます。決定論層だけで fail を判定すると false positive で運用が止まる、観察層だけだと regression を見落とす、人 review 層だけだと scale しない — の trade-off を 3 layer で吸収します。これは **将来の MCP server / 別 protocol 実装にも一般化** する構造的教訓です。

## LLM-as-judge layer（Phase 2.b、v0.2.3）

観察層の中核として、bot 応答を **LLM（default `gpt-4o-mini`）に採点させる** 仕組みです（[tealus #262](https://github.com/gamasenninn/tealus/issues/262) Phase 2.b）。決定論層では捉えられない semantic correctness を可視化します。

### scenarios.json の `llm_judge` field

```json
{
  "id": "S1",
  "description": "cross-room の tag 整理を依頼",
  "input": "業務メモ ルームの TODO tag を整理して",
  "deterministic_checks": [...],
  "llm_judge": {
    "criteria": "応答が tag を正しく分類して、漏れなく列挙しているか",
    "min_score": 7
  }
}
```

- `criteria`: 採点基準を自然文で記述（採点 LLM に渡る prompt 内容）
- `min_score`: 0〜10 の閾値、これ未満で **warn**（fail にはしない）
- 1 scenario あたり **1 judge call**、JSON mode (`response_format`) で score + reasoning を取得

### env override

| env | 役割 | default |
|-----|------|---------|
| `E2E_JUDGE_API_KEY` | judge 専用 API key（cost separation 用） | `OPENAI_API_KEY` fallback |
| `E2E_JUDGE_MODEL` | judge 用 model 切替 | `gpt-4o-mini` |

## 初期 6 scenarios

| ID | 内容 | tier | judgment |
|----|------|------|----------|
| S1 | cross-room の tag 整理 | Light v2 | LLM-as-judge あり（変動大） |
| S2 | mention strip（router の `@bot /light ...` 検出） | Light v1 | 決定論層のみ |
| S3 | PDF scan の要約（vision fallback path） | Light v1 | LLM-as-judge あり、preconditions で sample PDF 投下 |
| S4 | 画像生成（`generate_and_send_image`） | Light v2 | 決定論層 + 観察層 |
| S5 | 挨拶（router 直接応答） | Router | 決定論層 |
| S6 | Deep キーワード判定 | Deep | 決定論層 |

## 使い方の概観

```bash
# 全 scenario run
node agent-server/tools/e2e/run.js

# 特定 scenario のみ
node agent-server/tools/e2e/run.js --filter S1,S2

# dry-run（投下せず schema validation のみ）
node agent-server/tools/e2e/run.js --dry-run
```

出力は markdown report として `report/e2e-runs/YYYY-MM-DD-NNNN.md` に保存（非公開）。

詳細手順 / Tealus 側 setup（test bot user + test room 作成）は本体 repo の [`agent-server/tools/e2e/README.md`](https://github.com/gamasenninn/tealus/blob/main/agent-server/tools/e2e/README.md) を参照してください。

## 想定 use case

| trigger | 期待動作 |
|---|---|
| #260 (Light v2 機能 parity) fix 完了時 | S4 が pass する事を verify |
| #261 (vision fallback) fix 完了時 | S3 が pass する事を verify |
| router 周辺の修正時 | S2 / S5 / S6 で regression check |
| Light agent 改善時 | S1 で cross-room 探索の質的変化を観察（LLM-as-judge score 推移） |

## 設計判断のポイント

- **CI gate にしない理由**: LLM variance で false fail が頻発、運用が止まる
- **本番 DB 隔離**: scenarios.json 編集時、`test_room_env` 経由で test room を指す事を再 verify
- **API cost**: 1 run ~$0.5-1（gpt-5.4-mini）、Light v2 は subscription path で 0
- **regression suite 化**: 後続 fix で scenario 1 件追加して累積、初期 6 件は出発点

## 関連

- 上流: [tealus #262](https://github.com/gamasenninn/tealus/issues/262) E2E verification harness (Phase 1 + Phase 2.b LLM-as-judge)
- 関連 fix: [#258](https://github.com/gamasenninn/tealus/issues/258) / [#260](https://github.com/gamasenninn/tealus/issues/260) / [#261](https://github.com/gamasenninn/tealus/issues/261)
- 本体 repo: [`agent-server/tools/e2e/README.md`](https://github.com/gamasenninn/tealus/blob/main/agent-server/tools/e2e/README.md)
- 関連 doc: [Agent ガイド](agents.md) / [MCP Server](mcp.md)
