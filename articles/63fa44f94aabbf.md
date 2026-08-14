---
title: "CI の E2E 実行時間を X 倍改善した話"
emoji: "🙆‍♀️"
topics: []
published: false
---

こんにちは、Freelance Developer の天海です。

TODO: 
- [ ] 测试代码数量
- [ ] 难点：为什么 playwright worker 不行？
- [ ] api setup 问题
- [ ] frontend dev 和 build + start 对比

TAIAN で、CI 上の E2E テストが遅いという課題に取り組みました。

結論：壁時計時間を「3h 13m  → 24 分」に短縮。

**Before**: `3h 13m 18s` 🐢
**After**: `24m 58s` 🐇🐇🐇🐇🐇

![](https://storage.googleapis.com/zenn-user-upload/c820f4d0fded-20260120.png)

## なぜ遅かったのか（具体）

- CI の依存インストールに毎回時間がかかる。
- 大規模スイートを単一ジョブで直列実行していた。
- 待機条件が曖昧で、flaky が発生しやすい。
- 失敗時のアーティファクトが不足し、原因特定に時間がかかる。

## 2 並列化と分割（シャーディング）

動機：壁時計時間のボトルネックを解消するため。

- 機能単位でスイートを分割（例：認証／検索／購入フロー）。
- GitHub Actions のジョブマトリクスで並列化。
- テストのタグ付けと命名規則を統一。

効果：大スイートの直列実行を解消し、Y 分まで短縮。

3. キャッシュ戦略（依存・ブラウザ）

目的：毎回のセットアップ時間を削減。

- Node 依存（pnpm / npm）を CI キャッシュ化。
- ブラウザバイナリ（例：Playwright）をバージョンピン＋キャッシュ。
- コンテナ起動の軽量化（ベースイメージ見直し）。

効果：温状態での開始時間が短縮。冷起動も改善。

計測条件

- 環境：GitHub Actions（Linux）、Playwright、Node vXX
- シナリオ：全 E2E スイートを対象（n = 10）
- 指標：壁時計時間／flaky 率／失敗時の調査時間




