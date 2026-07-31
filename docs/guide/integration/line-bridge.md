# LINE Bridge 連携

Tealus は **LINE Official Account** からの投稿を Tealus ルームに取り込む **LINE Bridge** を Phase 4 後半（2026-06-02 〜 06-06）で実装し、**v0.4.0 で出荷**しました（Phase 1 〜 Phase 2.3 まで完成 + 凍結）。LINE グループ内の会話を **Tealus の context space** に流し込み、AI agent と業務知識辞書の対象として扱えるようにする外部 channel 拡張の第 1 例です。

本ページは **採用者向けの概念紹介**で、技術的な step-by-step 手順は本体 repo の [`setup-line-bridge.md`](https://github.com/gamasenninn/tealus/blob/main/docs/setup-line-bridge.md) を参照してください。

## 全体像 — Inbound 専用 bridge

```
[LINE Official Account]                 [Tealus]
    │                                      │
    │  ──── webhook (POST /api/line) ──►   │
    │   text / image / audio / video       │   既存の messages / voice / media
    │   file / sticker / location          │   pipeline に投影
    │                                      │
```

- **Inbound のみ（LINE → Tealus）**: Phase 2.x 完成、AI agent と業務辞書の対象として扱える
- **Outbound（Tealus → LINE post）は未実装**: LINE Messaging API が高額で需要 surface 待ち（Phase 2.4 候補）

## 対応する 7 つの type — Phase 1 〜 2.3 完成

| LINE 上の操作 | event type | Tealus への投影先 | 対応 Phase |
|---|---|---|---|
| テキスト送信 | `text` | text | **Phase 1** |
| 写真を選択 | `image` | image + サムネイル | **Phase 1** |
| 音声メッセージ | `audio` | voice + **自動文字起こし** + organon polyseme inject | **Phase 1** |
| 動画を選択 | `video` | video + ffmpeg thumbnail | **Phase 2.1** |
| ファイル添付 | `file` | file（**原名 + 原拡張子保持**）+ TextFilePreview（MD / JSON / CSV / TXT inline preview） | **Phase 2.1** |
| LINE スタンプ | `sticker` | image（LINE 公式 sticker CDN 経由） | **Phase 2.2** |
| 位置情報 | `location` | text + markdown（緯度経度 + Google Maps link） | **Phase 2.2** |

!!! info "複数画像の同時送信は 1 メッセージにまとまります（v0.7.0、[#353](https://github.com/gamasenninn/tealus/issues/353)）"
    LINE で複数枚の写真をまとめて送ると、**画像ごとに別々の webhook が順不同で届く**ため、以前は Tealus 側でバラバラのメッセージになっていました。

    v0.7.0 から、同じ送信に属する画像を待ち合わせて **1 メッセージ + 複数メディア**として再構成します。待ち時間は `LINE_IMAGESET_FLUSH_MS`（既定 15 秒）で、全枚が揃った時点で待たずに投稿されます。揃わなかった場合は待ち時間の経過後に届いた分だけで投稿されます。

## 設定方式 — 2 file 案（Phase 2.3）

LINE グループから Tealus ルームへの mapping は **2 ファイル** で管理します。**両ファイルとも書き換え後 next webhook で即反映** (server restart 不要)。

| file | 役割 | 編集者 |
|---|---|---|
| `server/config/line-groups.json` | **catalog file**。webhook 受信時に LINE API で group name を自動取得し atomic write | 自動（手動編集不要） |
| `server/config/line-group-mappings.json` | **mappings file**。LINE group → Tealus room の紐付け | 採用者が手動編集 |

mappings file は object form（D2 推奨）と pure string form（後方互換）の **両形式に対応**しています。

## LINE 公式 spec 準拠 — 「ダンマリ」回避設計

| 項目 | 設計 |
|---|---|
| HTTP 200 silent return | secret path mismatch / signature verify failed でも **200 + log warn** を返却。LINE 側の webhook auto-suspend trigger を構造的にゼロに |
| security | 攻撃者に URL / verify status 情報を leak しない（200 silent return の副次効果） |

「URL を間違えると LINE 側が webhook を勝手に suspend してしまう」採用者観点の trap を、**LINE 公式 spec に従った 200 silent return** で完全回避しています。

## Tealus PWA 内蔵 file preview

LINE 経由で添付された MD / JSON / CSV / TXT / source code 等のファイルは、Tealus PWA 上で **TextFilePreview component** により inline preview されます。

- UTF-8 decode（Chrome Android の文字化け回避）
- cache + truncate（256 KB 上限）
- `react-markdown` + `remarkGfm` + `remarkBreaks`（Tealus 内 markdown 統一感）

## scope 凍結（2026-06-06、Day 21）

Phase 2.3 完成時点で **structural foundation 完成** として scope 凍結が宣言されました。下記は **将来 user voice surface 待ち**:

| scope | 状況 | 補足 |
|---|---|---|
| Phase 2.3 UI（mapping CRUD 画面） | 凍結 | YAGNI、業務担当者の編集 use case surface 待ち。**2 file 案がそのまま UI data source として再利用可能** |
| Phase 2.4 outbound（Tealus → LINE post） | 保留 | LINE Messaging API 高額のため需要 surface 待ち |
| Phase 1.5 既存 LINE 業務グループ受信（Android 通知 forward 等） | 別 angle | user 業務必要度次第 |
| Phase 3 sticker 専用 type / location map / animated sticker / private sticker | enhancement 候補 | |

## セットアップ手順

step-by-step は本体 repo の **[`setup-line-bridge.md`](https://github.com/gamasenninn/tealus/blob/main/docs/setup-line-bridge.md)** に集約しています。主な手順:

1. LINE Official Account 作成（[LINE Official Account Manager](https://manager.line.biz/) 経由）
2. Messaging API 有効化 + Channel secret / Channel access token 取得
3. webhook URL（ngrok / cloudflare tunnel / 公開 HTTPS）を LINE Developers Console に登録
4. Tealus 側で `.env` に `LINE_CHANNEL_SECRET` / `LINE_CHANNEL_ACCESS_TOKEN` 設定
5. `server/config/line-group-mappings.json` で LINE group → Tealus room の mapping を書く
6. dogfood 投稿で 7 type すべての投影を確認

## 関連

- [tealus #288](https://github.com/gamasenninn/tealus/issues/288) — Phase 1（text + image + audio）
- [tealus #289](https://github.com/gamasenninn/tealus/issues/289) — Phase 2.1（file + video + TextFilePreview）
- [tealus #290](https://github.com/gamasenninn/tealus/issues/290) — Phase 2.2（sticker + location）
- [tealus #291](https://github.com/gamasenninn/tealus/issues/291) — Phase 2.3（2 file 案 + 凍結宣言）
- [リリースノート](../../releases.md): v0.4.0 の LINE Bridge cluster
- 本体 doc: [`setup-line-bridge.md`](https://github.com/gamasenninn/tealus/blob/main/docs/setup-line-bridge.md) — 技術的な step-by-step 手順
