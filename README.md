# Takoda99-DashBoard

タコダ99 の **ライブ観測ダッシュボード**（運営・開発者向け）。試合中の99店の状態を
ゲームサーバー（Takoda99-Server）の観測ストリームから購読して1画面で俯瞰する。

- フレームワーク無しの素の HTML / JS / CSS（依存ライブラリなし）。
- 本番は Server が `//go:embed` で `/admin` に**同梱配信**する（単一デプロイ）。このリポは配信物の**ソース**。
- 仕様の正典は Server 側 `docs/plan-honsen/`（`plan-h00_共有コントラクト.md` / `plan-h01_観測ダッシュボードMVP.md`）。
