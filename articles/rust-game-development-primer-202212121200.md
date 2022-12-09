---
title: "ゲームを作りたくなったからRustで作ってみる！ 入門編"
emoji: "🦀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust", "amethyst", "gfxrs", "ゲーム開発"]
published: false
---

<!-- textlint-disable -->

:::message
こちらは [Sun\* Advent Calendar 2022](https://adventar.org/calendars/8211) 12日目として公開した記事です。
:::

<!-- textlint-enable -->

# TL;DR

- ポーカーのチップ管理アプリを2Dゲーム形式で作りたい筆者
- Rustはゲーム開発にも利用できるのか？について調査
- C++に代わるパフォーマンスと信頼性、高い生産性を備えていることもあり、ゲーム開発に最適かつ世代交代への期待があることを把握
- 豊富なライブラリーの中から`Bevy`と`Fyrox`をピックアップし、特徴も比較した上で`Bevy`を使う方針に決定
- この記事では調査結果までを記載し、今後は続編として実装結果の記事化を目指す

# 参考になりそうな人

- 株式会社Sun Asteriskの社内文化やエンジニア職の雰囲気を知りたい方
- Rustを使った開発に興味がある方
- これからゲーム開発をしようと思っている方
- 定番のUnityやUnrealEngineといったゲーム開発IDEは実装経験があっても、言語+ライブラリーでのゲーム開発経験がない方

# はじめに

## 自己紹介

株式会社Sun Asteriskに所属する半田侑宜です。社内ではWebアプリエンジニアとして働いていて、某Webアプリのバックエンド開発に携わっています。なので、普段の業務ではゲーム開発には関わりがないですが、大学生・院生時代にUnityを使ったゲームを開発したり研究した経験があります。

余談ですが、プライベートでもよくゲームをプレイします。特にゼルダの伝説が好きです。任天堂Switchの[ゼルダの伝説 Breath of the Wild](https://www.nintendo.co.jp/zelda/botw/index.html)は有名ですが、その次回作[Tears of the Kingdom](https://www.nintendo.co.jp/zelda/totk/index.html)が来年前半に発売予定なので楽しみです。夜しか眠れません。

## 背景

最近、社内メンバーと部活動としてポーカー（[Texas Hold 'em Sit & Go]()）を楽しむ機会があり、（現金は賭けずに）チップを導入したこともあり大人げなく盛り上がりました。ただ、仕事終わりかつゲーム展開に夢中なハイテンションの大人たちにはチップを管理する余裕がなく、チップ管理をスマートフォンアプリに任せていました。賭ける時のルールや金額計算、手持ち数値化、ブラインド（強制ベット金額）の時間増額管理…といった具合に、機能が豊富でなかなか完成度は高いのですが、いくつか気になる点もありました。

- Digital Poker Chips
  - メリット
    - プレイヤー数に依らず1アプリで全体のチップを管理でき、とても手軽
  - デメリット
    - [Android版](https://play.google.com/store/apps/details?id=rubewijk.DigitalPokerChips&hl=en&gl=US&pli=1)のみのため、使えるスマートフォンに限りがある
- Rocket Poker Chips
  - メリット
    - [iOS](https://apps.apple.com/jp/app/rocket-poker-chips/id1579858247)、[Android](https://play.google.com/store/apps/details?id=rocket.chips&hl=ja&gl=US)の両対応
    - プレーヤー1人につき1アプリを使用し同一ネットワークでもって相互通信するシステムなので、各自自分のスマホを操作・閲覧すればいい（昨今のコロナ情勢に沿ったスタイルともいえる）
    - 「待機中の人の画面が暗くなる」「チップのアニメーションが凝っている」などの工夫もあり完成度が高い
  - デメリット
    - 同一ネットワークが用意できないなど、場合によっては「プレーヤー1人につき1アプリ」の敷居が高い
    - 一定期間の無料利用後、課金の必要あり

<!-- prettier-ignore -->
|     |     |
| --- | --- |
| ![](/images/rust-game-development-primer-202212121200/digital_poker_chips_preview.webp =350x) *Digital Poker Chipsのスクリーンショット[^digital_poker_chips_screenshot]* | ![](/images/rust-game-development-primer-202212121200/rocket_poker_chips_preview.webp =320x)<br /> *Rocket Poker Chipsのスクリーンショット[^rocket_poker_chips_screenshot]* |

この2つのアプリは特に完成度が高いので列挙していますが、それでも互いにメリット・デメリットがあります。そして、「これらのメリットを融合したちょうどいいアプリがあればなぁ…」と考えていたのですが、エンジニアらしく「だったら自分で開発すればいいじゃん」と思い至りました。まさしくこれが「ぼくのかんがえたさいきょうのポーカーチップ管理アプリ」ですね。（まだ出来上ってないですが）
とりあえずは社内向けに開発して、さらに活動を盛り上げられたらいいなって思っています。

[^digital_poker_chips_screenshot]: [Google Play](https://play.google.com/store/apps/details?id=rubewijk.DigitalPokerChips&hl=en&gl=US&pli=1)に掲載の画像を引用。
[^rocket_poker_chips_screenshot]: [App Store](https://apps.apple.com/jp/app/rocket-poker-chips/id1579858247?platform=iphone)に掲載の画像を引用。

## 技術選定

ざっくりとした基本方針として、下記の3点があります。

- "Rocket Poker Chips"のようにグラフィカルで表現豊かにしたい
- WebGLなどで動作させて、クロスプラットフォーム対応のアプリにしたい
- 1アプリか、複数アプリ同時接続かをユーザーが選択できる

そのため、2Dゲーム開発環境が必要です。ゲーム開発というと、UnityやUnrealEngineについ手が伸びてしまうイメージが個人的にあります。おそらく、有名・手軽・業界標準という要素が大きく影響していますが、ふと「言語から選定してもいいのでは？」と気づきました。加えて、ちょうどRustに手を出してみたかったところだったので、ここぞとばかりにRustを使用言語として選定してみました。「動作が速い」「保守性が高い」と言われているので、結構ゲーム開発に向いているんじゃないかなと踏んでいます。

# 調査

## Rustとは

言語の選定ができたところで、ここからは利用技術を事前に調査します。そもそもRustとは一体どんな言語なのか、今回改めて調査してみました。

<!-- prettier-ignore -->
![](/images/rust-game-development-primer-202212121200/rust-logo-blk.jpg)
*Rustのロゴ[^rust-logo]*

Rustは、2015年5月15日にバージョン1.0.0がリリースされた[^rust-released-version-1.0.0]比較的新しい言語です。2022年12月12日時点では、バージョン1.65.0[^rust-released-version-1.65.0]に達しています。開発体制はRust Foundationという非営利団体が行っています。[Firefoxブラウザ](https://www.mozilla.org/ja/firefox/)で有名なMozillaが開発支援を行っており、あのMicrosoftやGoogleも一目置く存在です[^microsoft-and-google-use-rust]。MicrosoftはWindows、GoogleはAndroidという形で、それぞれのOS開発にてRustが活躍しています。

Rustの[公式サイト](https://www.rust-lang.org/ja)によると、下記が売りの言語のようです。

- 高いメモリー効率による高いパフォーマンス
- 静的型付けやメモリー安全性、スレッド安全性による高い信頼性
- エンジニアにとって優しいツール群を備えたことによる高い生産性

また、Mozillaの開発者によれば、C言語やC++に代わるシステムプログラミング言語を目指していた[^mozilla_developer_interviewed_for_rust]ようです。OS開発で採用される流れにも、Rustの特性にも納得できる方針ですね。そして、C++はゲーム開発においても活躍する言語のうちの1つなので、その役目がRustに替わることを期待してみるのも良さそうです。

[^rust-logo]: [Rust Foundation - Logo Policy and Media Guide](https://foundation.rust-lang.org/policies/logo-policy-and-media-guide/)より引用。
[^rust-released-version-1.0.0]: [Announcing Rust 1.0 | Rust Blog](https://blog.rust-lang.org/2015/05/15/Rust-1.0.html)を参照。
[^rust-released-version-1.65.0]: GitHubの[rust/RELEASES.md at master · rust-lang/rust](https://github.com/rust-lang/rust/blob/master/RELEASES.md#version-1650-2022-11-03)を参照。
[^microsoft-and-google-use-rust]: [グーグルやMSが「Rust」言語でOS開発、背景に国家による諜報活動の影 | 日経クロステック（xTECH）](https://xtech.nikkei.com/atcl/nxt/column/18/00692/042700054/)を参照。
[^mozilla_developer_interviewed_for_rust]: [Rust - Mozilla の開発したシステムプログラミング言語 - に関するインタビュー](https://www.infoq.com/jp/news/2012/08/Interview-Rust/?itm_source=infoq_en&itm_medium=link_on_en_item&itm_campaign=item_in_other_langs)を参照。

## Rustでゲームを作るには？

さて、Rustの特性をある程度知れたところで、今度はゲーム開発を要点に調べてみました。

まずはゲーム開発の手助けとなるライブラリーの存在を調査しました。その過程で、[Are we game yet?](https://arewegameyet.rs/)というRustのゲーム開発周辺情報について整理しているサイトを発見しました。サンプルゲームやドキュメントが記載されていて、この時点でRustゲーム開発コミュニティーの盛り上がりを感じます。

このサイトの[Ecosystemセクション](https://arewegameyet.rs/#ecosystem)を参考にしつつ、日本語記事でも盛り上がりをみせているライブラリーがないか探してみたところ、BevyとFyroxという2つのライブラリーに出会いました。2つとも機能やドキュメントが豊富でライブラリー自体の開発頻度も活発のようだったので、これらについてさらに調査を進めました。その結果を下記にまとめます。

### Bevy

<!-- prettier-ignore -->
![](/images/rust-game-development-primer-202212121200/bevy_logo_light.png =300x)
*Bevyのロゴ [^bevy-logo]*

- 公式サイト： https://bevyengine.org
- GitHub: https://github.com/bevyengine/bevy
- バージョン： v0.9.1（2022年12月12日時点）
- 次元： 2D/3Dどちらも作成可能
- プラットフォーム： Windows、Linux、macOS、WebAssembly（AndroidやiOSのネイティブ版は開発中）
- 特徴：
  - データ指向
  - 処理の並行性が高く、高速
  - Entity Component System(ECS)を採用
  - ホットリロード対応

### Fyrox

<!-- prettier-ignore -->
![](/images/rust-game-development-primer-202212121200/fyrox_logo.png =100x)
*Fyroxのロゴ [^fyrox-logo]*

- 公式サイト： https://fyrox.rs
- GitHub: https://github.com/FyroxEngine/Fyrox
- バージョン： v0.28.0（2022年12月12日時点）
- 次元： 2D/3Dどちらも作成可能
- プラットフォーム： Windows、Linux、macOS、WebAssembly
- 特徴：
  - オブジェクト指向
  - 安全性、信頼性があり、高速
  - GUIエディターがある
  - 機能が豊富（アニメーション、アセット管理、AI、物理計算…）

[^bevy-logo]: GitHubリポジトリの[bevy/assets/branding/bevy_logo_light.png](https://github.com/bevyengine/bevy/blob/main/assets/branding/bevy_logo_light.png)を引用。
[^fyrox-logo]: GitHubリポジトリの[Fyrox/pics/logo.png](https://github.com/FyroxEngine/Fyrox/blob/master/pics/logo.png)を引用。

# 調査後記

「Rustで開発してみたい」という気持ちで進み始めたため、調査前は「Rust言語でゲーム開発ができるのか」という疑問がありました。ただ、いざ調べてみると、「ゲーム開発に対してもRustのポテンシャルが発揮できそう」という感触が得られ、「ゲーム開発界隈も盛り上がっている」ということが分かりました。ライブラリーも豊富に存在していた中でBevyとFyroxの2択に絞り込みましたが、とりあえずは**Bevy**を使う方針で進めてみようと考えています。

Bevyのようにデータ志向やECSといった特徴を持つ技術を使った開発経験がなく、そこに興味を持ったところが一番の理由です。また、FyroxはBevyに比べて機能が豊富で、レンダリング面での優位性やIDEと同等の操作といった要素がありそうです。しかし、今回のゲームについてはそこまでのグラフィック精度や開発サポート性は必要ないだろうと判断しました。

# まとめ

Rustでのゲーム開発について調査しました。その結果として、Rustが持つ特徴はゲーム開発に最適であり、C++に代わる活躍を見せ始めていることが分かりました。また、開発の手助けとなるライブラリーも豊富に存在しており、その中から2Dゲームの開発とクロスプラットフォームビルドが可能であるBevyを選定しました。

今回は、調査に時間が必要だったことと記事の執筆期間が短かったことから、今回は調査結果までを記事にしました。今後は、今回の内容を元に開発を開始して、その経過や結果を再度記事にできればと考えています。

次回 [Sun\* Advent Calendar 2022](https://adventar.org/calendars/8211) 13日目は、PMとして活躍する今年入社の松本陶矢くんによる記事です。「新卒PMがいきなりシニアのベトナムメンバーとタッグを組んだ話」をお送りします！
