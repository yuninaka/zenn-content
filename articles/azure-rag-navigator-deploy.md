---
title: "個人実証：AzureネイティブRAGでERP導入ナビゲーターを構築した(第3回:デプロイ・CI/CD・IaC編)"
emoji: "🧭"
type: "tech"
topics: ["azure", "appservice", "githubactions", "bicep", "rag"]
published: true
---

## 1. 概要

### 1.1 やったこと(3行サマリ)

- [前回記事](https://zenn.dev/yuninaka/articles/azure-rag-navigator-eval)までで完成したRAGアプリを、Azure App Serviceに実デプロイし、GitHub Actionsで自動デプロイ・品質ゲートのCI/CDパイプラインを組んだ
- 「[レビューをやめた話](https://zenn.dev/singularity/articles/stopped-reviewing-my-code)」の考え方を参考に、Ruff strictルール・mypy strict・vultureによる「読まなくても壊れないコードベース」化に取り組み、導入直後に見つかったRuff 92件・mypy 9件の指摘をすべて型を緩めずに解消した
- App Serviceのリソース作成でリージョン単位のVMクォータ不足に遭遇し、Japan Westへの変更で解決。この制約をBicep IaCのパラメータ既定値にもそのまま反映し、実際に遭遇した制約を隠さずコード化した(実デプロイは未実施、構文チェックのみ)

リポジトリ: [azure-rag-erp-navigator](https://github.com/yuninaka/azure-rag-erp-navigator)

### 1.2 結果だけ先に

| 項目 | 実測値 |
|---|---|
| Ruff strictルール導入直後の指摘件数 | 92件(型注釈追加が大半、すべて解消) |
| mypy strict導入直後のエラー件数 | 9件(すべて型を緩めずに解消、`# type: ignore`は2箇所のみ) |
| GitHub Actions CI実行時間(pytest+ruff+mypy+vulture) | 39秒 |
| GitHub Actions CD実行時間(App Serviceへのデプロイ) | 3分34秒 |
| 本番環境での質問応答レイテンシ(埋め込み+検索+生成の合計) | 約16.5秒(App ServiceはJapan West、他リソースはJapan East) |

本番稼働直後に、環境変数の登録漏れが原因で生のトレースバックが画面に露出するバグを実際に発見・修正した。詳細は3章で扱う。

## 2. Step7前半:「読まなくても壊れないコードベース」への品質ゲート整備

### 2.1 なぜ今、品質ゲートを強化したか

Step1〜6は、pytestによる単体テストと実リソースに対する動作確認を都度行いながら進めてきたが、Ruff・mypyの設定は初期状態のまま緩かった。デプロイフェーズに入る前に、[「レビューをやめた話」](https://zenn.dev/singularity/articles/stopped-reviewing-my-code)で紹介されている考え方(型・lintのゲートを厳しくすることで、人間のレビューが「ロジックの妥当性」に集中できるようにする)をこのプロジェクトにも適用することにした。

導入したのは4つのツールで、それぞれ役割と「ビルドを失敗させるか」を分けている。

| ツール | 役割 | ゲートにするか |
|---|---|---|
| pytest | 単体テスト | する |
| ruff | lint(複雑度・型注釈必須化・命名・bugbear等) | する |
| mypy(strict) | 静的型チェック | する |
| vulture | デッドコード検出 | **しない**(レポートのみ) |

vultureだけ非ゲートにしたのは、Streamlitのコールバック登録やdataclassのフィールドなど、動的な呼び出しパターンを誤検知しやすいため。導入初日からビルドを失敗させる運用はリスクが高いと判断し、まずは警告表示に留めた。

### 2.2 Ruff strict化:92件の大半は型注釈の欠落

`select`に`C90`(mccabe複雑度)・`PLR`(pylint refactor)・`N`(naming)・`B`(bugbear)・`ANN`(型注釈必須化)を追加したところ、導入直後に92件の違反が出た。内訳は以下の通り。

- **型注釈の追加(約80件)**: 関数の引数・戻り値への型注釈漏れ。テストファイルも含めて全て手作業で注釈した。`per-file-ignores`でテストディレクトリを対象外にする選択肢もあったが、今回は採用しなかった
- **マジックナンバーの定数化(約9件)**: `assert x == 3`のような比較を、値の意味を名前で示す形に変更した
- **設計変更(1件)**: `generate_answer`が引数過多(PLR0913、10引数)だった。Azure接続一式(5引数)を`RagDependencies`というdataclassにまとめる`src/rag/dependencies.py`を新設し、`eval/run_eval.py`・`src/app/streamlit_app.py`・`scripts/ask_question.py`の3箇所で重複していたクライアント組み立てロジックも同時に共通化した

```python
@dataclass(frozen=True)
class RagDependencies:
    openai_client: AzureOpenAI
    embedding_deployment: str
    chat_deployment: str
    search_client: SearchClient
    history_manager: SessionHistoryManager
```

lintの指摘を機械的に消すだけでなく、「引数が多すぎる」という指摘を設計上の重複解消のきっかけとして使えたのは、strict化を早めにやってよかった点だと思う。

### 2.3 mypy strict化:9件はTypedDict・isinstanceナローイング・castの使い分けで解消

`strict = true`を設定した直後の9件のエラーは、すべて型を緩めずに解消した。

- **TypedDict導入**: Cosmos DBのセッションメタデータ(`SessionMeta`)、Azure AI Search投入用ドキュメント(`SearchDocument`)を、動的な`dict`から構造化した型に変更
- **isinstanceによるナローイング**: YAMLフロントマターから読んだ`title`/`module_tags`を`SourceDocument`(dataclass)に渡す前に型を検証
- **`# type: ignore`(2箇所、理由コメント付き)**: `azure-search-documents`が実行時にmonkey-patchで追加する`SearchFieldDataType.Collection`は、mypyの静的解析からは呼び出し不可能なEnumに見える。ライブラリ側の実装詳細であり当方では修正できない
- **`cast`(3箇所、いずれもSDK境界でのみ使用)**: Cosmos DBの動的な応答を`SessionMeta`とみなす境界、OpenAI SDKが要求するリテラル型Unionと`list[dict[str, str]]`の不一致、`SearchDocument`(TypedDict)を`upload_documents`が要求する`list[dict[Any, Any]]`に渡す境界。いずれも自コードが読み書き双方を管理しており、実際のデータ形状を保証できる箇所のみに限定した

`# noqa`/`# type: ignore`を使う場合は必ず理由をコメントで明記し、理由が古くなったらエントリ自体を削除する、というルールをCLAUDE.mdに明文化した。

### 2.4 vultureが見つけた1件の「誤検知に見えて実は正しい指摘」

vulture導入直後、`Any`の未使用importが1件検出された。コードは`cast("list[dict[Any, Any]]", ...)`のように型引数を文字列で指定していたため、mypyはこの文字列内の`Any`を正しく解釈できていたが、vultureの静的解析からは使用箇所が見えていなかった。

```python
# 修正前: 型引数を文字列指定していたため、vultureからAnyの使用箇所が見えなかった
cast("list[dict[Any, Any]]", documents)
# 修正後: 非文字列形式に変更(mypyの解釈は変わらない)
cast(list[dict[Any, Any]], documents)
```

一見vultureの誤検知に見えたが、実際には「文字列アノテーションのせいで静的解析ツールから見えにくいコードになっている」という、それ自体は妥当な指摘だった。修正によりmypyの型チェック結果は変えずに、vultureからも使用が見える形にできた。

### 2.5 チェックを1本のスクリプトに集約する

`scripts/ci_check.sh`(pytest→ruff→mypy→vulture(report-only)の順で実行)を新設し、GitHub Actionsの`ci.yml`はこのスクリプトを呼び出すだけの薄いラッパーにした。

```yaml
name: CI
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync
      - run: ./scripts/ci_check.sh
```

チェック内容を変更する場合は必ず`ci_check.sh`側を直し、ワークフローファイル側には個別のチェックコマンドを追記しない、というルールをCLAUDE.mdに明記した。「ローカルでは通ったのにCIで落ちる」という食い違いを防ぐのが目的。実際にPR作成後、CIが`pull_request`トリガーで起動し、39秒(pytest 46件・ruff・mypy・vultureすべて)で完走することを確認した[^1]。

## 3. Step7後半:Azure App Serviceへのデプロイ

### 3.1 CLIでのリソース作成を諦め、ポータルに切り替えた経緯

Azure OpenAI/AI Search/Cosmos DB(Step1〜3)と同様、App Serviceのリソース自体もAzureポータルからユーザーに作成してもらう方針にした。理由は、このセッションからのAzure CLIログインが以下の2つの問題に阻まれ、CLIでのリソース作成が現実的でなかったため。

- テナントのセキュリティ既定値によるアクセスブロック(`AADSTS530035`)
- az-cli 2.90.0自体のバグ(`--allow-no-subscriptions`指定時、サブスクリプションが0件だと`_subscription_selector.py`内で`NoneType`に対して`.get()`を呼び出しクラッシュする)

### 3.2 リージョン単位のVMクォータ不足という誤算

当初Japan East・Basic(B1)プランでApp Serviceを作成しようとしたところ、`Microsoft.Web/serverFarms`のpreflight検証で失敗した。

```
デプロイの検証に失敗しました。役立つ可能性がある基本APIから得られた追加の詳細情報:
The template deployment ... Microsoft.Web/serverFarms ...
Current Limit (Total VMs): 0
```

無料枠のF1プランに切り替えても同じクォータ不足エラーが再現したため、SKUの問題ではなくJapan EastリージョンそのものにこのサブスクリプションのApp Service用コンピューティングクォータが割り当てられていないと判断した。Japan Westに変更したところ、B1で正常に作成できた。

結果として、App ServiceだけJapan West、Azure OpenAI/AI Search/Cosmos DBはJapan Eastという、リージョンを跨ぐ構成になった。意図した設計ではなく、クォータ制約への対応として生まれた構成であることは、コード上のコメントにもそのまま残している。

### 3.3 発行プロファイル方式でのデプロイ

Azure CLI/Azure AD経由のOIDC連携は3.1節のテナント事情で構築が困難だったため、発行プロファイル(Kudu基本認証)方式を採用した。ポータルの「発行プロファイルの取得」でダウンロードしたファイルの内容をGitHub Secretsに登録し、`.github/workflows/cd.yml`から参照する。

```yaml
name: CD
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync
      - run: uv export --format requirements.txt --no-dev --no-hashes -o requirements.txt
      - uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ vars.AZURE_WEBAPP_NAME }}
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          package: .
```

Azure App ServiceのOryxビルドは`requirements.txt`を前提とするため、uvの依存管理(`uv.lock`)からデプロイ直前に都度生成している。生成物はリポジトリにコミットせず、常に`uv.lock`と一致させる設計にした。

`main`マージ後、CDワークフローが自動起動し3分34秒でデプロイに成功した[^2]。

### 3.4 本番で実際に見つかったバグ:環境変数の登録漏れが生トレースバックを露出させた

デプロイ後、実際のURLをPlaywrightで操作して確認したところ、初回アクセス時にStreamlitのデフォルトの生トレースバック(ファイルパスを含む)がそのまま画面に表示されるバグに遭遇した。

原因は2つ重なっていた。1つは単純な設定漏れで、`AZURE_SEARCH_INDEX_NAME`のアプリケーション設定(環境変数)を登録し忘れていたこと。もう1つはコード側の設計上の見落としで、Azureクライアントの初期化処理(`_load_dependencies()`)の呼び出しが、質問応答時のエラーハンドリング(`_handle_query`の`try/except`)の**保護範囲の外側**にあり、Step5で作った「エラー詳細を画面に出さない」設計がこの初期化フェーズには適用されていなかった。

```python
# 修正前: _load_dependencies() が main() の中で try/except の外側にあった
def main() -> None:
    ...
    deps = _load_dependencies()  # ここで例外が出ると生トレースバックが画面に出る
    ...
```

環境変数を追加すれば表面上の症状は消えるが、それでは「別の設定不備が起きたら同じ問題が再発する」ため、コード側も修正した。`main()`全体を新しいラッパー関数で保護し、失敗時はStep5と同じ固定文言＋サーバー側ログのみのフォールバックに統一した。

```python
def _load_dependencies_safely(
    *, load_dependencies_fn: Callable[[], RagDependencies] = _load_dependencies
) -> RagDependencies | None:
    try:
        return load_dependencies_fn()
    except Exception:
        logger.exception("依存関係の初期化に失敗しました")
        return None
```

この修正は別PRとして切り出し、環境変数修正後の動作確認・修正後の再確認をどちらも実リソースに対して行った[^3]。「今動いているから良し」で終わらせず、原因のコード側の見落としまで潰したのは、CLAUDE.mdに明記した「テストでカバーすべきケース」のうち「エラー」の観点を、本番トラブルから逆算して補強した形になる。

### 3.5 リージョンを跨ぐ構成での実測レイテンシ

環境変数修正後、実際に質問を送信し、Azure AI Search・Cosmos DB(Japan East)、Azure OpenAI(埋め込み+チャット、実際に使用していたリソースはEast US 2 EUAP)への呼び出しを含めて、App Service(Japan West)から回答と引用元が表示されるまでの時間を計測した。3リージョンをまたぐ構成になっているが、これは意図した設計ではなく、後述(4.5節)の通りBicep側の既定値(Azure OpenAIもJapan East想定)とも一致していない、実際に検証で使ったリソースの生の姿である。

```
質問送信 → 回答・引用元表示まで: 約16.5秒
```

前回記事の`generate_answer`のロジック自体はリージョン間の呼び出しを意識した設計をしていない。それでもリージョンを跨ぐ構成で体感できないほどの遅延にはならなかった。ただしこれは1回きりの手動計測であり、負荷がかかった状態や複数ユーザー同時アクセス時の特性は検証していない。

## 4. Step8: BicepによるIaC化

### 4.1 実デプロイはせず、構文チェックに留めた判断

Step1〜7で個別にAzureポータルから作成したリソースを、Bicepで一括プロビジョニング可能な形にコード化した。ただし`az deployment group create`による実デプロイの動作確認は行っていない。理由は3.1節の通り、このセッションからのAzure CLIログインが安定して使える状態になかったこと、また実デプロイを試みると既存リソースと重複作成される・意図せず上書きされるリスクがあったため。`az bicep build`/`az bicep lint`による構文チェックのみ実施する方針を、着手前にユーザーと確認した。

```bash
./scripts/validate_bicep.sh
```

`main.bicep`・全モジュール・パラメータファイルともコンパイルエラー0件、lintも0件で通過した。

### 4.2 実際に遭遇した制約をパラメータの既定値に反映する

3.2節のJapan Eastクォータ制約は、Bicepのリージョンパラメータにもそのまま反映した。

```bicep
// 既定値をjapanwestにしているのは、実際の検証でjapaneast・B1プランのApp Service作成が
// 「Current Limit (Total VMs): 0」というVMクォータ不足のpreflightエラーで失敗し、
// F1(無料)に切り替えても再現したため。SKUではなくリージョン側のクォータ制約と判断し、
// japanwestに変更したところ解決した。
@description('App Serviceのリージョン。既定はjapaneastでのVMクォータ制約を回避したjapanwest')
param appServiceLocation string = 'japanwest'
```

他のリソース(Azure OpenAI/AI Search/Cosmos DB/監視)はすべて`japaneast`が既定で、App Serviceだけ`japanwest`になっている。理想化された構成図をコード化するのではなく、実際に遭遇した制約をそのままコードとコメントに残す方を選んだ。

### 4.3 APIキーの直接注入という妥協と、その代償を明記する

Azure OpenAI/AI Search/Cosmos DBの各キーは、各モジュールが`listKeys()`(AI Searchは`listAdminKeys()`)で取得し、`@secure()`出力としてmain.bicepに渡し、App Serviceのアプリケーション設定に直接埋め込む設計にした。

```bicep
@secure()
@description('Azure OpenAIアカウントのAPIキー(App ServiceのApplication Settingsに直接注入する用。本番運用ではKey Vault参照+Managed Identityを推奨)')
output primaryKey string = openAiAccount.listKeys().key1
```

これにより`az deployment group create`を1回実行するだけで、リソース作成からApp Serviceへの接続情報設定まで一気通貫で完結する。ただしこの方式には、APIキーがデプロイ履歴(`az deployment group show`等)に残りうるという弱点がある。本番運用ではKey Vaultにキーを格納し、App ServiceがManaged Identity経由でKey Vault参照する設計にすべきだが、今回は検証用の一括プロビジョニングのしやすさを優先し、簡略化したことをコード内コメントにも明記した。

### 4.4 モデル名・バージョンは既定値なしの必須パラメータにした

Azure OpenAIのモデル提供状況・バージョンは頻繁に更新されるため、実在するか未確認の値を既定値としてハードコードすることは避け、チャット・埋め込みそれぞれのモデル名・バージョンは既定値なしの必須パラメータとした。パラメータファイルにはプレースホルダー値を置き、デプロイ時点でポータル/CLIで確認してから値を埋めるようコメントで明記している。

```
param chatModelName = '<例: gpt-4.1-mini。デプロイ時点で利用可能なモデル名を指定>'
param chatModelVersion = '<デプロイ時点の最新バージョンを指定>'
```

前回記事(第1回)の2.2節で書いた「サブスクリプションのクォータ制約で設計時に想定していたモデルがそのままデプロイできるとは限らない」という教訓を、IaC側の設計判断にも一貫させた形になる。

### 4.5 正直に書いておきたいこと:BicepはStep1〜7の実リソースと完全には一致していない

3.5節で触れた通り、実際に検証で使っていたAzure OpenAIリソースは、当初想定していたJapan Eastではなく`East US 2 EUAP`というリージョンで稼働していた。一方、今回のBicep(`main.bicep`)の`openAiLocation`パラメータは既定値`japaneast`のままにしている。これは4.2節のApp Serviceリージョンのように「実際に遭遇した制約をパラメータに反映した」ケースとは違い、単純にBicepを書いた時点で実リソースの正確なリージョンを再確認していなかったための食い違いで、Bicepと実リソースの間の監査は行っていない。実デプロイを見送ったこととあわせて、「このBicepはStep1〜7の実際の構成を完全に再現するものではなく、事後的に書いた設計意図の記録」という位置づけであることを、正直に明記しておきたい。

## 5. 得られた知見

### 5.1 クォータはSKUではなくリージョン単位で刺さることがある

3.2節のApp Serviceクォータ不足は、B1(有料)とF1(無料)の両方で同じエラーが再現したことで初めて「SKUの問題ではない」と判断できた。もしB1だけを試して諦めていたら、「無料枠なら通るかもしれない」という誤った仮説のまま時間を使っていたはずだ。クォータエラーに遭遇したら、SKUを変えて再現するかを確認するのが切り分けの近道だという、実務的な教訓を得た。

### 5.2 IaC化は「実デプロイの検証」なしでも一定の価値がある

今回のBicepは実デプロイ未検証という限定的な成果ではあるが、それでも「Step1〜7で個別にポータル操作した内容を、後から一貫した形でコード化する」という作業自体が、各リソース間の依存関係(Cosmos DBの`defaultTtl`設計をBicep側にも反映する、App ServiceのApplication SettingsがAzure OpenAI/AI Search/Cosmos DBの出力値に依存する、等)を再確認する機会になった。実デプロイでの動作確認ができていない以上、「このBicepが実際に動く」という保証はまだできないが、少なくとも「今回の検証で実際に使った設定値・遭遇した制約」を将来の自分や他の人が再現可能な形で残せたことには意味があると考えている。

### 5.3 マネージドスタックは「サーバー管理」を消す代わりに「クラウド運用の意思決定」を持ち込む

過去のNeo4j×LangChain検証では、AuraDB Freeというマネージドサービスを使っていたとはいえ、デプロイ・CI/CD・IaCという運用面の意思決定はスコープに含まれていなかった。今回Azure App Serviceへのデプロイまで踏み込んだことで、クォータ制約への対処・発行プロファイルかOIDCかの認証方式選択・APIキーをデプロイ履歴に残すか手間をかけてKey Vaultに移行するか、といった、コードの正しさとは別軸の意思決定が次々に発生した。これはNeo4j検証には出てこなかった論点で、「マネージドサービスを本番運用に載せる」というフェーズに特有の負荷だと感じた。

## 6. まとめ:シリーズ全体を振り返って

3回にわたり、Azure OpenAI Service・Azure AI Search・Cosmos DB・App Serviceという4つのマネージドサービスを使い、ERP導入ナビゲーターRAGを設計・実装・評価・デプロイまで一貫して検証した。

| 記事 | 内容 | 主な発見 |
|---|---|---|
| [第1回](https://zenn.dev/yuninaka/articles/azure-rag-navigator-build) | インデックス設計・インジェスト・セッション管理 | 次元不一致・`.env`コピペ事故・チャンク分割バグ・セッション競合状態 |
| [第2回](https://zenn.dev/yuninaka/articles/azure-rag-navigator-eval) | RAG回答生成・チャットUI・精度評価 | golden_qa平均0.94〜1.00、キーワード網羅率の言い換え耐性の弱さ、Neo4j比較の前提の違い |
| 第3回(本記事) | デプロイ・CI/CD・IaC | Ruff/mypy strict化、リージョン単位のクォータ制約、本番トレースバック露出バグ、実デプロイなしIaCの限界 |

過去のNeo4jグラフRAG検証との対比で言えば、精度そのもの(golden_qaのスコア)はどちらも高水準に達しており、「マネージドスタックだから精度が劣る」というようなことはなかった。違いが顕著に出たのは、実装コスト・SDKバージョン差異への対応(第1回)・そして今回のデプロイ・運用面の意思決定の多さだった。個人の検証環境でここまでの運用面の複雑さ(クォータ・認証方式・シークレット管理)に向き合う機会は、グラフRAG検証の範囲では発生しなかった論点であり、今回Azureネイティブスタックで一気通貫の検証をしたことで初めて具体的な形で得られた知見だと思っている。

[^1]: `gh run watch`でGitHub Actionsの実行をリアルタイムに追跡して確認した。`uv sync`から`./scripts/ci_check.sh`まで問題なく完走した。
[^2]: GitHub Actionsの実行ログ(Actions画面、または`gh run view`)で所要時間を確認した。
[^3]: `.env`の`AZURE_OPENAI_API_KEY`を一時的に不正な値に書き換えて修正後のアプリを起動し、画面に固定文言のみが表示されることと、サーバー側ログに例外のスタックトレースが記録されることの両方を確認した(第2回記事3.2節と同じ検証手法)。検証後`.env`は正しい値に復元した。
