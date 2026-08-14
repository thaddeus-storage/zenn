---
title: "AI Agentの「あとで直す」を38件のIssueに変えた。Frog運用の実例"
emoji: "🐸"
type: "tech"
topics:
  - "aiagent"
  - "github"
  - "frog"
published: false
---

**AI Agent（以下、Agent）**が作業中に見つけた問題は、その場で直さなければチャットに埋もれがちです。
私はiOSアプリFoodPhotosに[Frog](https://github.com/wevm/frog)を導入し、主タスクの範囲外で見つかった問題をGitHub Issueへ移す運用を試しました。
2026年8月6日から14日までに、Frogが作成した`friction`ラベル付きIssueは38件になり、8月14日時点ですべて閉じています。

38件のIssueを閉じるための判断や修正をしたのはFrogではありません。
Frogが担当したのは、問題を発見時の情報とともに記録し、Issueとして公開し、Issueの状態をリポジトリへ同期する部分です。
修正内容と終了条件はAgentまたは開発者が決めます。

@[card](https://github.com/wevm/frog)

## 地図機能のPRで見つかった二つの問題

FoodPhotosのPR #69では、地図画面への遷移と表示範囲内の検索機能を追加しました。
このPRの検証中に、Dashboardの並行処理テストが二つ失敗しました。
二つのテストは単独で再実行すると通ったため、地図機能の回帰とは断定できず、実行順に依存する既存テストの可能性が残りました。

この時点でテスト基盤まで修正すると、地図機能のPRに別の変更が混ざります。
Agentは修正を始めず、`frog log`で次の二件を記録しました。

- Issue #79：古くなったUIのdraft PRをマージすると、生成物や画面仕様の競合を手作業で解消する必要がある
- Issue #80：Dashboardのactivation concurrency testが実行順に依存して失敗する

FrogはPR #69へ「2 frictions recorded」とコメントし、それぞれのIssueを作成しました。
PR #69は地図機能の変更とfriction logを含めてマージされ、不安定なテストの修正はIssue #80として別の作業になりました。

ここで残した`friction.md`には、期待する挙動、実際の挙動、再現手順、考えられる対策、そのとき進めていた作業が入っています。
次のAgentは元のチャットを探さず、失敗条件から調査を再開できました。

## 25ミリ秒の待機を決定的なテストへ変えた

Issue #80はPR #81で修正しました。
以前のテストは25ミリ秒の固定待機と途中の処理件数に依存していたため、通常のタスクスケジューリングでも処理が先へ進むと失敗しました。

PR #81では、モックの写真ライブラリへ分析リクエストを止める**リクエストゲート**を追加しました。
テストは最初のリクエストを確実に停止し、その状態でアプリを再びアクティブにする処理または複数のアクティブ化処理を実行します。
停止中のリクエストが一件だけであることを確認してからゲートを解放し、最後に三枚の写真が一度ずつ分析されたことを検証します。

修正後は対象の二テストを10回連続で実行し、すべて成功しました。
`DashboardViewModelTests`の27件と`FoodMapSearchPolicyTests`の4件をまとめた実行でも、31件すべてが成功しました。
PR #81は`Closes #80`を含み、デフォルトブランチへマージされた時点でGitHubがIssue #80を閉じました。

## 修正中に見つかったSwift 6とXCTestの制約

PR #81の実装中には、別の問題も見つかりました。
Swift 6では、actor-isolatedな値を`await`し、その式を`XCTAssertEqual`へ直接渡すとコンパイルできません。
XCTestのアサーションが同期的なautoclosureを使うためです。

次の形はコンパイルエラーになります。

```swift
XCTAssertEqual(await photoLibrary.analysisRequests(), ["mock-0"])
```

先に値を取得してからアサーションへ渡すとコンパイルできます。
PR #81では次の形にしました。

```swift
let requestsWhileBlocked = await photoLibrary.analysisRequests()
XCTAssertEqual(requestsWhileBlocked, ["mock-0"])
```

Agentはこの制約も`frog log`で記録し、その記録をPR #81へ含めました。
PRのマージ後、FrogはIssue #82を作成し、「1 friction recorded」というコメントからIssueへリンクしました。

![FrogがPR内のfrictionをIssue #82へ関連付けた画面](/images/frog-ai-agent-friction/recorded-friction.png)

Issue #82の対策自体は、すでにPR #81のローカル変数への変更に含まれていました。
そのため、解決箇所をIssueへコメントしたうえで、Issueを手動で閉じました。
すべてのfrictionを後日のPRへ送る必要はなく、同じPRで解決した場合も発生条件と対策を検索可能な形で残せます。

## 38件に含まれていた問題

38件は一種類の問題ではありません。
実際のIssueには、プロダクトコード、テスト、Simulator、ビルド手順、リリース手順、Agent用ツールの問題が含まれていました。

たとえばIssue #18には、XCTestを使ったスクリーンショット検証に30分から60分かかっていた事実を記録しました。
`simctl launch`と`simctl io screenshot`へ切り替えた実測では、25枚を約2分で取得できました。
Issue #33には、`String(localized:bundle:locale:)`へ明示的なlocaleを渡しても、`.strings`の検索結果がプロセスの言語になる問題を残しました。

通知欄にはFrogのアイコンとIssueの状態変更が並びました。
これは新しい問題が発生した数ではなく、それまで作業ログに散らばっていた問題に追跡先が付いた数です。

![Frogが作成したIssueと関連PRが並ぶGitHubの通知欄](/images/frog-ai-agent-friction/fixed-non-core-issues.png)

## 自動化される範囲

この運用での「自動記録」は、FrogがAgentの操作を監視して問題を推測する機能ではありません。
FoodPhotosでは`AGENTS.md`に、作業前に`frog list`を実行することと、ツール、文書、API、テスト、規約で問題に遭遇した時点で`frog log`を実行することを定めました。
Agentがこのルールを実行して初めて記録が作られます。

`frog log`は一件の問題を次のディレクトリへ保存します。
`artifacts/`には最小再現コードやログを追加できます。

```text
.agents/friction-log/20260814150726-xctest-assertion-autoclosures/
├── friction.md
└── artifacts/
```

FoodPhotosではGitHub Appモードを使いました。
Frogは記録をIssueとして公開し、元の`friction.md`へIssueの参照を書き戻し、Pull Requestへ対応表をコメントします。

Issueを修正するのはFrogではありません。
また、Issueを閉じただけでは、デフォルトブランチ上の記録は消えません。
Issueが閉じたあとにFrogが`frog/sync` Pull Requestを更新し、そのPull Requestをマージすると解決済みのディレクトリが削除されます。

## 最小限の導入手順

Agentへ導入を任せる場合は、公式READMEのプロンプトを使えます。

```text
Read https://frog.fm, and set up Frog in my project.
```

手動の場合はFrogを初期化し、生成された説明に従って`AGENTS.md`へルールを追加します。

```sh
npx frog init
```

GitHub AppモードとAction-onlyモードは同時に使わず、リポジトリごとに一方を選びます。
どちらのモードでも`frog/sync` Pull Requestを更新できるように、GitHub ActionsによるPull Requestの作成を許可する必要があります。
GitHub Appの権限追加やworkflowファイルの追加はリポジトリ設定を変えるため、Agentへセットアップを任せる場合も実行前に内容を確認します。

## チャットの外へ問題を残す

PR #69で見つかった不安定なテストはIssue #80になり、PR #81で決定的なテストへ置き換わりました。
その修正中に見つかったXCTestの制約もIssue #82として残り、同じPRで解決済みだと確認して閉じました。

Frog導入後の38件という数字は、Agentが作業中に見つけた問題をチャットの外へ移し、状態を確認できるようにした件数です。
主タスクと無関係な修正を混ぜず、問題の再現条件も失わないことが、この運用で得られた効果でした。
