---
title: "個人実証(続編)：誤検知の多い異常検知を、どう運用に乗せるか（サイレント運用・人間確認UI・LLM分析の設計）"
emoji: "🛠️"
type: "tech"
topics: ["python", "streamlit", "openai", "mypy", "security"]
published: true
---

再現性の確認や実装の詳細を追う場合は、以下のリポジトリを参照されたい。

リポジトリ: [log-anomaly-detection-poc](https://github.com/yuninaka/log-anomaly-detection-poc)

前半記事: [個人実証：機微データに触れないログ異常検知は成立するか（型で強制する設計と実測精度）](https://zenn.dev/yuninaka/articles/log-anomaly-detection-metadata-only)

## 前半のあらすじと今回のスコープ

前半記事では、AIが機微データ(生ログ・実データ)に一切触れない設計のまま異常検知が成立するかを検証した。ログレコードを「メタデータ層」(`MetadataRecord`)と「生データ層」(`RawDataRecord`)という2つのdataclassに分離し、異常検知アルゴリズムがメタデータ層のオブジェクトしか受け取れないことをmypyによる型レベルの強制で担保した[^step3]。一方、STL分解・IsolationForestという2つの検知手法で実際に精度を測定した結果、recallは高い(74〜100%)ものの、precisionは8〜13%程度に留まり、大半が誤検知だった[^readme-results]。

この結果は、固定閾値の検知器だけでは実運用に耐えないことを示しており、検知結果を即座に通知せず一定期間の精度を見極めるサイレント運用モードと、最終判断を人間の確認に委ねる人間確認フローという、当初から計画していた次段階の必要性を実測データが裏付ける形になった。今回はこの2つに加えて、機微データを含む判断が必要な場面で人間の確認後にのみLLMを使う根本原因分析まで、3段階の実装を行った。

## Part1: サイレント運用モード

検知結果をいきなり運用に流さず、一定期間「通知せずに蓄積するだけ」の状態で精度を見極める仕組みを実装した[^step5]。

核となるのは`accumulate_silent_mode_records`という純粋関数である。検知スコアと閾値を受け取り、`flagged`(閾値超えかどうか)を持つレコードを返すだけで、print出力やアラート送信のような副作用は一切持たない。「検知結果を即座に通知しない」という制約を、実行時のフラグやコメントではなく、関数の戻り値がデータだけであるという契約そのものによって保証している。これは前半記事で採用した「型で強制する」考え方の延長で、今回は「関数の設計で強制する」という形を取った。

精度が十分かどうかの判定は`decide_production_readiness`が行う。サイレントモード期間中に蓄積したデータからprecisionを算出し、閾値(デフォルト50%)以上なら本番移行可能(`reason="ready"`)、未満なら精度不足(`reason="insufficient_precision"`)と判定する。ここで重要なのは、precisionが算出不能(`NaN`、サイレント期間中に一件も陽性判定を出さなかった場合)のケースを`insufficient_samples`として明確に区別している点である。「精度が足りない」と「そもそも評価できていない」は、次に取るべき対応が違う(前者は検知ロジックの見直し、後者はデータを待つ)ため、同じ「移行不可」でも理由を潰さずに残した。

この設計を実際のデータで動かすと、7日・14日・30日いずれの学習データ量でも、STL・IsolationForestの両方で`reason="insufficient_precision"`(本番移行不可)という判定になった[^readme-results]。これは前半記事で示したprecision8〜13%という実測値をそのまま反映した結果であり、判定ロジックが正しく機能していることの裏付けでもある。閾値を50%から恣意的に下げて「移行可能」という結果を出すような調整は行っていない。

## Part2: 人間確認UI

サイレント運用モードが蓄積した検知結果を、人間が確認するためのUIをStreamlitで実装した[^step6]。ここでも前半記事の型分離の考え方を、UIの操作フローにまで一貫させることを狙った。

UIは2段階開示になっている。第1段階では、`flagged=True`のメタデータ層(`scenario_id`・エンドポイント名・時刻・スコア)だけを一覧表示する。生データ層(顧客IDを含む`RawDataRecord`)は、各行の「生データ層を確認する」ボタンを押した場合にのみ表示される。この開示を保証しているのが`raw_data_records_for_window`という専用関数で、指定した1バケットの観測ログだけを`RawDataRecord`に変換する。観測ログ全体を無条件で変換する既存関数(`to_raw_data_records`)とは別に用意することで、「人間が明示的に確認を選択した場合にのみ生データ層に触れる」ことをコードのフロー自体で保証している。

実装はpureなオーケストレーション層(`ui_logic.py`)とStreamlitのレンダリング層(`ui.py`)に分離した。前者はpytestで単体テストできるが、後者はボタン操作を伴うUIのためpytestでは検証しづらい。そこで`streamlit.testing.v1.AppTest`という公式のヘッドレステスト機構を使い、実際に日数選択→検知実行→一覧表示→ボタン押下、という操作を自動化して検証した。

この検証中に、実際にバグを1件発見した。7日分のデータで検知を実行すると1459件がflaggedになったが(この規模感自体が前半記事で示したprecisionの低さを裏付けている)、そのうち同じ`scenario_id`がSTL・IsolationForestの両方でflaggedになるケースがあり、その場合Streamlitのボタンの`key`が重複して`StreamlitDuplicateElementKey`という例外が発生した[^bug-key]。単体テストだけでは見つからない、実際にウィジェットを操作して初めて表面化する類のバグで、ボタンの`key`に`scenario_id`だけでなくアルゴリズム名も含めることで解消した。

## Part3: LLMによる根本原因分析

Step6のUI上で生データ層を確認した後、さらに別のボタンを押した場合にのみ、Azure OpenAIに根本原因分析を依頼する機能を追加した[^step7]。

ここで最初に決めたのは、「人間が画面上で確認してよい情報」と「外部LLM APIに送信してよい情報」は別の許可レベルとして扱う、という方針である。人間がすでに生データ層を確認したからといって、その全ての情報を外部のLLM APIに送ってよいことにはならない。根本原因分析(タイムスタンプ・ステータスコード・レイテンシのパターン分析)という目的に照らせば、顧客IDは不要な情報である。そこで`RawDataRecord`から顧客IDを除いた`RootCauseAnalysisInput`という専用の型を新設し、LLM呼び出し関数の型シグネチャがこの型だけを受け付けるようにした。`RawDataRecord`を直接渡すとmypyがコンパイル時にエラーとして検出する構成で、前半記事のStep3・今回のStep6と同じ、mypyのsubprocess実行によるテストでこの境界を検証している。

Azure OpenAIへの呼び出しは失敗時、例外の詳細を`logging`モジュールでサーバー側のログにのみ残し、UI側には固定文言だけを返すようにした。ここでもレビューで実装上の抜けが1件見つかった。当初のコードは、API呼び出し自体が例外を投げるケースはtry節で捕捉していたが、呼び出しは成功したものの応答の構造が期待と異なるケース(例えばコンテンツフィルタ等で`choices`が空リストになる場合)を想定しておらず、レスポンスの解析をtry節の外に置いていた[^bug-parse]。この場合`IndexError`が未捕捉のままUIまで伝播し、防ごうとしていたはずの「例外詳細の画面への漏洩」がまさに起きる構成になっていた。レスポンスの解析もtry節に含めることで修正し、空の`choices`リストや想定外の応答構造でも同じフォールバック文言を返すことを確認する回帰テストを追加した。

なお、実際のAzure OpenAIリソースへの接続確認はまだ行っていない。認証情報が用意されていないため、この時点ではモック(フェイククライアント)による検証に留めており、実接続の確認は利用者が自分の環境で行う前提としている。スコープを広げすぎないための意図的な線引きである。

## むすび

検知アルゴリズム(Step3〜4)・人間確認UI(Step6)・LLMによる根本原因分析(Step7)という3段階すべてで、「その相手に本当に必要な情報だけを渡す」という設計思想が一貫して貫かれた。検知アルゴリズムにはメタデータ層のみ、UIの画面には人間が明示的に求めた場合のみ生データ層、外部LLM APIには顧客IDを除いた情報のみ、という形で、前半記事で示した型分離の考え方が、機能を追加するたびに形を変えて延長されていった。これが今回のPoC全体を通じて一番の収穫だったと考えている。

一方でまだ検証できていないことも多い。実際のAzure OpenAIリソースへの接続確認、固定閾値ではない検知精度のチューニング、より大規模なデータでの検証はいずれも手つかずのままである。実装の詳細はリポジトリを参照されたい。

リポジトリ: https://github.com/yuninaka/log-anomaly-detection-poc

[^step3]: 型分離の実装はPull Request [#8](https://github.com/yuninaka/log-anomaly-detection-poc/pull/8)を参照。READMEの「メタデータ層と生データ層の型分離(Step3)」節。前半記事全体の実装範囲(Step0〜4)はリポジトリのコミット履歴を参照。
[^readme-results]: 数値はREADME「実測結果」節の表、およびPull Request [#14](https://github.com/yuninaka/log-anomaly-detection-poc/pull/14)の実行結果を参照。
[^step5]: 実装はPull Request [#14](https://github.com/yuninaka/log-anomaly-detection-poc/pull/14)、マージコミット[`58e49f2`](https://github.com/yuninaka/log-anomaly-detection-poc/commit/58e49f226a8a77b6d656650612762d7f3874dc67)。READMEの「サイレント運用モード(Step5)」節を参照。
[^step6]: 実装はPull Request [#16](https://github.com/yuninaka/log-anomaly-detection-poc/pull/16)、マージコミット[`bcf2cbd`](https://github.com/yuninaka/log-anomaly-detection-poc/commit/bcf2cbd38be1e61ba1a40e3f379123148f95243d)。READMEの「人間確認UI(Step6)」節を参照。
[^bug-key]: `StreamlitDuplicateElementKey`のバグ修正はコミット[`80e175b`](https://github.com/yuninaka/log-anomaly-detection-poc/commit/80e175b685c5491b9e9903b2f4f1d7aa189e7715)(Pull Request #16に同梱)。
[^step7]: 実装はPull Request [#18](https://github.com/yuninaka/log-anomaly-detection-poc/pull/18)、マージコミット[`fd428b7`](https://github.com/yuninaka/log-anomaly-detection-poc/commit/fd428b7e314bb01c16b9b1b7fe2e14d6e4591d6e)。READMEの「LLM根本原因分析(Step7)」節を参照。
[^bug-parse]: API応答解析の未捕捉バグの修正はコミット[`ce85b2f`](https://github.com/yuninaka/log-anomaly-detection-poc/commit/ce85b2f9021544b04bb677d1b4695d6e6e35c78d)(Pull Request #18)。
