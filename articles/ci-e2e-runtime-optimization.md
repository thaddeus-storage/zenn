---
title: "Playwright の E2E CI を4分割し、3時間13分から25分に短縮した"
emoji: "⚡"
type: "tech"
topics:
  - "playwright"
  - "githubactions"
  - "e2e"
  - "ci"
published: true
published_at: "2026-08-21 09:00"
---

結婚式場向けSaaSの開発で運用していた Playwright の E2E テストは、テストケースの増加に伴って GitHub Actions の実行時間が3時間を超えていました。
そこで、最も大きなテストスイートを Playwright の sharding と GitHub Actions の matrix で4分割しました。
当時の workflow 表示では、実行時間が `3h 13m 18s` から `24m 58s` になり、壁時計時間は約87.1%減少しました。

![GitHub ActionsのE2E workflowを4分割した前後の実行時間](/images/ci-e2e-runtime-optimization/before-after.png)

## 遅さを四つに分ける

E2E テストの実行時間を調べるときは、最初から並列数を増やすのではなく、時間の使われ方を分けて確認しました。
今回の workflow には、次の四種類のコストがありました。

- **テスト実行**：Playwright がブラウザを操作して各テストを実行する時間
- **フロントエンド準備**：Vite によるビルドとサーバー起動にかかる時間
- **API 準備**：Docker コンテナ、データベース、Rails アプリを起動する時間
- **CI 環境準備**：Node.js の依存関係や Playwright のブラウザを用意する時間

3時間を超えていた主因は、大きなテストスイートを一つの job で実行していたことでした。
ただし、job を分けると API とフロントエンドの準備も各 job で繰り返されます。
テスト実行の短縮だけを見ると、別の場所で増えたコストを見落とします。

## workers ではなく sharding を使った理由

Playwright の `workers` と `shard` は、どちらもテストを並列に実行しますが、並列化する範囲が異なります。
`workers` は一台の runner 内で複数の worker process を動かします。
一方、sharding はテストスイートを分割し、それぞれを別の runner で実行できます。

一台の GitHub Actions runner で workers を増やしても、CPU とメモリの上限は変わりません。
ブラウザ、API、データベース、フロントエンドを同じ runner で動かす構成では、workers を増やすほどリソース競合が起きやすくなります。
今回は一台の中の並列度を上げるより、最も大きなスイートを複数の runner に分散するほうが、壁時計時間の短縮に適していました。

なお、workers と sharding は排他的な機能ではありません。
各 shard の中でも workers は利用できますが、まず runner 間で分割し、その後に各 runner のリソースに合わせて workers を調整する順序にしました。

## 最大のスイートを4つの runner に分ける

今回は planner 向けのスイートを4 shard に分け、GitHub Actions の matrix から別々の runner で実行しました。
セットアップ処理を省いて実行構造だけを示すと、次のようになります。

```yaml:.github/workflows/e2e-testing.yml
jobs:
  planner-e2e:
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]

    steps:
      - name: Run Playwright tests
        run: >-
          yarn playwright test
          --project=planner
          --shard=${{ matrix.shard }}/4
          --reporter=blob

      - name: Upload blob report
        if: ${{ !cancelled() }}
        uses: actions/upload-artifact@v4
        with:
          name: blob-report-${{ matrix.shard }}
          path: blob-report

  merge-reports:
    if: ${{ !cancelled() }}
    needs: planner-e2e

    steps:
      - name: Download blob reports
        uses: actions/download-artifact@v4
        with:
          pattern: blob-report-*
          path: all-blob-reports
          merge-multiple: true

      - name: Merge reports
        run: >-
          yarn playwright merge-reports
          --reporter=html
          ./all-blob-reports
```

`fail-fast: false` にしたのは、一つの shard が失敗しても残りの shard を実行し、失敗箇所をまとめて確認するためです。
各 shard は blob report を artifact として保存し、最後の job が一つの HTML report に統合します。
これにより、実行する runner が分かれても調査の入口を一つに保てます。

## Vite の dev server を preview server に替える

フロントエンドは当初、E2E テストでも Vite の dev server を使っていました。
dev server は必要になったモジュールを実行時に変換するため、CI のように CPU が限られた環境では、初めて開く画面の表示に時間がかかります。
実際に、ログイン画面は開けても、ログイン後の画面が60秒以内に表示されず、Playwright が timeout することがありました。

そこで、CI では先にフロントエンドをビルドし、Vite の preview server で配信する構成へ変更しました。

```bash
# planner、customer、partnerを事前にビルドする
yarn build:e2e

# preview serverとproxy serverを起動する
yarn start:e2e

# Playwrightを実行する
yarn e2e
```

この変更によって、画面を開くたびに発生していた初回変換を、明示的なビルド工程へ移せました。
ビルド時間そのものは必要ですが、テスト中の応答時間を予測しやすくなり、timeout を増やすだけの対処を避けられます。

## 固定待機を readiness check に替える

API の起動では、当初 `sleep 120` で一律に待っていました。
この方法では、API が30秒で起動しても残り90秒を使えず、120秒後も起動していなければテスト開始後に失敗します。

変更後は、Docker コンテナの状態と Rails アプリの応答を一定間隔で確認し、利用可能になった時点で次へ進むようにしました。
上限時間までに起動しなかった場合は、コンテナの状態とログを出して setup を失敗させます。
これにより、待ち時間を短縮するだけでなく、テスト本体の失敗と環境起動の失敗を区別できるようになりました。

## 依存関係とブラウザをキャッシュする

matrix で job を増やすと、各 runner で同じ依存関係を準備する回数も増えます。
そこで、Node.js の依存関係、Playwright のブラウザ、API のセットアップに必要なツールをキャッシュしました。

キャッシュキーは固定値にせず、lockfile のハッシュを含めます。
依存関係が変わったときに古いキャッシュを完全一致として扱わないためです。

例えば、API のセットアップに使うツールのキャッシュキーには、`Gemfile.lock` のハッシュを含めました。

```yaml
- name: Cache API dependencies
  uses: actions/cache@v4
  with:
    path: |
      /home/linuxbrew/.linuxbrew/Cellar
      /home/linuxbrew/.linuxbrew/Homebrew
      ~/.local/share/mkcert
      ~/.pki/nssdb
    key: e2e-deps-${{ hashFiles('api/Gemfile.lock') }}
    restore-keys: e2e-deps-
```

キャッシュは sharding の代わりにはなりません。
キャッシュが減らすのは主に環境準備の時間であり、長いテストスイートを一つの runner で実行する構造は変わらないからです。

## sharding が増やしたコスト

4 shard の `planner-e2e` は、それぞれ独立した GitHub Actions runner で動きます。
そのため、各 shard が API とフロントエンドをセットアップし、同じ準備を合計4回実行することになりました。

setup 専用の job を一度だけ実行しても、そこで起動したプロセスやポートを別の runner から共有することはできません。
ビルド結果は artifact として渡せますが、実行中の API、データベース、フロントエンドを共有するには、外部のテスト環境を用意する必要があります。
外部環境を共有すると、今度は shard 間のデータ競合とテストの独立性を管理しなければなりません。

今回の変更は、総 runner 時間の最小化ではなく、開発者が結果を待つ壁時計時間の短縮を優先しています。
約25分まで短縮した段階では共有環境を追加せず、各 shard が独立した環境を持つ構成を維持しました。

## 観測結果

|  | workflow の所要時間 |
| --- | ---: |
| 変更前 | 3h 13m 18s |
| 変更後 | 24m 58s |
| 変更前 ÷ 変更後 | 約7.74 |

この結果を支えたのは、単一の調整ではありません。
最大のスイートを4台の runner に分散し、Vite の実行方法、readiness check、キャッシュ、report の統合を同時に整えた結果です。

## 参考資料

- [Sharding | Playwright](https://playwright.dev/docs/test-sharding)
- [Parallelism | Playwright](https://playwright.dev/docs/test-parallel)
- [Running variations of jobs in a workflow | GitHub Docs](https://docs.github.com/actions/using-jobs/using-a-matrix-for-your-jobs)
