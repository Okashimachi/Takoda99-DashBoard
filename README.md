# Takoda99-DashBoard

タコダ99 の **ライブ観測ダッシュボード**（運営・開発者向け）。試合中の99店の
体力／評価／順位／生存を1画面で俯瞰する。

- フレームワーク無しの素の HTML / JS / CSS（依存ライブラリなし）。
- データ源は **ゲームサーバー（Takoda99-Server）の `/admin/ws`**（読み取り専用の観測ストリーム）。
- 本番は Server が `//go:embed` で `/admin` に**同梱配信**する（単一デプロイ）。このリポは配信物の**ソース**。
- 仕様の正典は Server 側の `docs/plan-honsen/plan-h01_観測ダッシュボードMVP.md` と `plan-h00_共有コントラクト.md`。

## ファイル

| ファイル | 役割 |
|---|---|
| `index.html` | ページ構造（トップバー＋99店グリッド＋設定パネル） |
| `styles.css` | ダークな「ライブオペレーション盤面」スタイル・レスポンシブ |
| `app.js` | `/admin/ws` の購読・再接続・盤面の差分描画・並び替え |

## ワイヤ契約（plan-h00 §4）

`/admin/ws` は `proto.Envelope{ type, payload }` を JSON で流す。

```jsonc
// h01
{ "type": "StoreListUpdate",
  "payload": {
    "stores": [
      { "storeId": "p-1", "displayName": "たこ焼き太郎",
        "evalNormalized": 0.83, "rank": 4, "creditLife": 3,
        "alive": true, "finalRank": null }
    ],
    "aliveCount": 42
  }
}
```

- `evalNormalized` は 0..1。`rank` は生存店内の評価順（1 が最上位）。
- `creditLife` は体力（信用）。0 で自滅脱落。分母（初期体力）は config 由来のため、
  観測した最大値を分母として体力バーを描く。
- `finalRank` は脱落済みの店のみ入る（生存店では欠落）。
- **未知の `type`（h02 の `AdminSnapshot` 等）は無視**する前方互換設計。

## 開発（ローカル）

このダッシュボードは静的ファイルなので、任意の静的サーバーで開ける。

```bash
# 例: 適当な静的サーバーで配信
python3 -m http.server 5173
# → http://localhost:5173/?token=<CONFIG_ADMIN_TOKEN>
```

別オリジンから Server の `/admin/ws` に繋ぐ場合（cross-origin）:

1. Server を起動（`CONFIG_ADMIN_TOKEN` を設定）。cross-origin を許可するなら
   `ALLOWED_ORIGINS=http://localhost:5173` を Server に渡す（`plan-h00 §5`）。
2. ダッシュボードの ⚙（設定）で **サーバー**（例 `http://localhost:8080`）と**トークン**を入力して接続。
   - URL に `?token=…` があれば自動で使う。空欄のサーバー入力は「このページと同じサーバー」を意味する。

本番同等（同梱配信）で見るなら、Server の `/admin?token=…` を開く（下記 sync 後）。

## Server への同梱（sync）

Server は自リポ内 `internal/admin/webdist/` を `//go:embed` する。ここへ最新の配信物を取り込む:

```bash
# Takoda99-Server 側で実行（既定のコピー元は ../Takoda99-DashBoard）
scripts/sync-dashboard.sh
# 別パスのときは引数で: scripts/sync-dashboard.sh /path/to/Takoda99-DashBoard
```

sync 後に Server をビルドすれば `/admin` から最新のダッシュボードが配信される。
将来バンドラを入れる場合は `dist/` を出力すれば sync スクリプトが自動でそちらを拾う。

## 操作

- **並び順**: スコア順（強い順）／切られる順（弱い順）／行列が長い順／店ID順（固定配置で状態遷移を追う）。
- **色 = 誰か**（人間 / 強Bot / 中Bot / 弱Bot の4色）。**状態は色を奪わずに**表す
  （次に切られる＝赤枠、脱落＝彩度を落とす、負スコア＝0 線より下）。
- リーダー（rank1）は金枠、**人間の店は常にハイライト**（脱落しても見失わないよう灰色化を弱める）。

## h34 で足した観測（tier と heat）

`plan-h34`（Server 側 `docs/plan-honsen/`）で、h30〜h33 の調整結果を見るための表示を足した。

| 表示 | 使うフィールド | 見たいこと |
|---|---|---|
| tier の4色（分布ビュー・盤面） | `stores[].tier`（`strong`/`normal`/`weak`・人間は無し） | 強 Bot が上位を独占していないか |
| tier 別の集計（分布ビュー上部） | 同上 | 各 tier の平均スコア・生存数。人間は順位も出る |
| スコア分布の 0 基準線 | `stores[].score` | **負スコア**（h31 以降、弱 tier は終盤に負へ回る）を潰さずに見る |
| 火力（heat）カーブ | `heatLevel` / **`heatMaxLevel`** | **上限に届いたか**（`maxLevel` の水平線） |
| 平均打鍵数 | `avgKeystrokes` | h30 で語を短くした効果 |

- **heat の履歴はサーバーに持たせない**（配信の状態は publisher が持つ・h23）。
  ダッシュボードがスナップショット受信のたびに積むので、**カーブは「このページを開いてから」のぶん**。
  試合開始前に開いておくこと。試合（`matchId`）が替わったら履歴はリセットされる。
- `heatMaxLevel` を配らないサーバー（h34 未反映）に繋いだ場合は、**上限線を引かずに警告を出す**。
  勝手な線を引くと「上端に届いた」の誤読を生むため（#75 → h26 → h32 と3度再発した問題）。
