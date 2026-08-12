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

- **並び順**: 評価順（強い順）／体力が少ない順（危ない店が上）／店ID順（固定配置で状態遷移を追う）。
- **色**: 体力レール（緑→黄→赤）、リーダー（rank1）は金、脱落はグレーアウト＋最終順位バッジ。
- 体力が減った瞬間は赤フラッシュ、脱落した瞬間はスタンプ演出で状態遷移が見える。
