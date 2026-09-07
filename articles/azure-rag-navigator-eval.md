---
title: "個人実証：AzureネイティブRAGでERP導入ナビゲーターを構築した(第2回:回答生成・チャットUI・精度評価編)"
emoji: "🧭"
type: "tech"
topics: ["azure", "openai", "streamlit", "rag", "llm"]
published: true
---

## 1. 概要

### 1.1 やったこと(3行サマリ)

- [前回記事](https://zenn.dev/yuninaka/articles/azure-rag-navigator-build)で構築したAzure AI Search(ハイブリッド検索)・Cosmos DB(セッション管理)を統合し、Azure OpenAIによるRAG回答生成ロジック(引用元提示・マルチターン対話)とStreamlitチャットUIを実装した
- golden_qa(18問・キーワード網羅率方式)で3つのチャンク分割方針を比較し、`fixed_512`平均1.000 / `fixed_256`平均0.972 / `heading_aware`平均0.944という結果を得た
- 過去のNeo4j×LangChainグラフRAG検証(9問・平均1.00)と同一の評価手法で比較したが、質問数・題材が異なるため単純な優劣比較はできないという結論に至った。その代わりに見えてきたのは「評価手法そのものの限界」という共通の論点

題材・リポジトリは[前回記事](https://zenn.dev/yuninaka/articles/azure-rag-navigator-build)と同じ(架空のERP製品「ERPNavi」、実データ不使用)。

リポジトリ: [azure-rag-erp-navigator](https://github.com/yuninaka/azure-rag-erp-navigator)
(この記事時点のコード: [`article-2-generation-and-eval`タグ](https://github.com/yuninaka/azure-rag-erp-navigator/tree/article-2-generation-and-eval))

### 1.2 結果だけ先に

| チャンク分割方針 | golden_qa平均スコア(18問) |
|---|---|
| `fixed_512`(固定512トークン) | 1.000 |
| `fixed_256`(固定256トークン) | 0.972 |
| `heading_aware`(見出し単位・チャットUIの既定戦略) | 0.944 |

18問中16問は3戦略とも満点で、差が出たのはわずか2問。両方とも「RAGの回答内容自体は正しいが、期待キーワードと違う言い回しをしたために未マッチになった」ケースだった。詳細は4章で扱う。

## 2. Step4: RAG回答生成ロジック(引用元提示・マルチターン対話)

### 2.1 プロンプト設計:「根拠がなければ正直に言う」を明文化する

ERP業務の担当者向けチャットボットで一番避けたいのは、参考情報にない内容をもっともらしく生成してしまうことだと考え、システムプロンプトで明示的に禁止した。

```python
SYSTEM_PROMPT = """あなたはERP／基幹システム「ERPNavi」の導入・設定手順を案内する
ナビゲーターアシスタントです。以下のルールを厳守してください。

- 必ず「参考情報」に記載された内容のみを根拠に回答してください。
  参考情報にない内容を推測で補ってはいけません。
- 参考情報だけでは回答できない場合は、正直に「提供された情報からは回答できません」と述べ、
  社内の担当部署への問い合わせを促してください。
- 回答の各主張の末尾に、根拠とした参考情報の番号を [1] のように付記してください。
- 業務担当者にも分かりやすいよう、手順は箇条書きで示してください。
"""
```

検索結果は番号付きの「参考情報」としてユーザーメッセージに埋め込み、モデルには`[1]`のような番号で出典を付けさせる方式にした。

```python
def build_user_message(query: str, hits: list[SearchHit]) -> str:
    context = "\n\n".join(_format_context_entry(i, hit) for i, hit in enumerate(hits, start=1))
    return f"### 参考情報\n{context}\n\n### 質問\n{query}"
```

`generate_answer`は、クエリの埋め込み生成→ハイブリッド検索→履歴取得→Azure OpenAI呼び出し→履歴保存、という一連の流れを1関数に統合した。

```python
def generate_answer(
    *, query: str, session_id: str, deps: RagDependencies,
    top_k: int = DEFAULT_TOP_K,
    chunk_strategy: str = DEFAULT_CHUNK_STRATEGY,
    history_turns: int = DEFAULT_HISTORY_TURNS,
) -> RagAnswer:
    query_vector = embed_texts(deps.openai_client, deps.embedding_deployment, [query])[0]
    hits = hybrid_search(deps.search_client, query, query_vector, top=top_k, chunk_strategy=chunk_strategy)
    citations = build_citations(hits)

    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        *deps.history_manager.build_chat_messages(session_id, max_turns=history_turns),
        {"role": "user", "content": build_user_message(query, hits)},
    ]
    response = deps.openai_client.chat.completions.create(
        model=deps.chat_deployment,
        messages=cast(list[ChatCompletionMessageParam], messages),
    )
    answer = response.choices[0].message.content or FALLBACK_ANSWER

    deps.history_manager.append_turn(session_id, query, answer, citations)
    return RagAnswer(answer=answer, citations=citations)
```

チャットUIの既定チャンク戦略は`heading_aware`にした。見出し単位でまとまりが保たれ、引用元の`section_path`も直感的になる(「在庫管理モジュール設定ガイド > 在庫閾値アラートの設定」のように意味のある単位で表示できる)という理由からだが、この判断が4章のgolden_qa結果と少しねじれることになる。

### 2.2 コードレビューで見つかった2つのバグ

**引用元のタイトルが二重表示される**: `section_path`は前回記事の設計上、見出しパスの先頭に既にタイトル(H1)を含む。にもかかわらず引用元の組み立て関数が`hit.title`と`hit.section_path`を両方連結していたため、「在庫管理モジュール設定ガイド 在庫管理モジュール設定ガイド > 在庫閾値アラートの設定」のようにタイトルが二重に表示されるバグがあった。

```python
def _format_context_entry(index: int, hit: SearchHit) -> str:
    # section_pathは見出しパスの先頭に既にtitle(H1)を含む(chunkers.pyの設計)ため、
    # ここでhit.titleを前置するとタイトルが二重表示される。
    return f"[{index}] {hit.section_path}\n{hit.content}"
```

同じ重複が動作確認用CLI(`scripts/ask_question.py`)の引用元表示にも波及していたため、両方修正し、重複しないことを確認する回帰テストを追加した。

**Azure OpenAIが`content=None`を返すケースが未処理だった**: コンテンツフィルタが作動した場合など、`response.choices[0].message.content`が`None`になるケースを考慮していなかった。フォールバック文言を用意し、その場合も「その質問には回答できなかった」という事実として履歴に残す(保存自体はスキップしない)方針にした。

```python
FALLBACK_ANSWER = "回答を生成できませんでした。担当部署にお問い合わせください。"
...
answer = response.choices[0].message.content or FALLBACK_ANSWER
```

### 2.3 実リソースでの動作確認:マルチターンの文脈保持

同一セッションで2問続けて投げ、指示語の解決を確認した[^1]。

```
1問目: 「ERPNaviの初期設定はどこから始めればいいですか？」
  → 初期設定ガイドの該当セクション(テナント作成・会社情報登録・組織階層設定・
    チェックリスト)から正しく検索・引用され、手順が箇条書きで回答された

2問目(同一セッション): 「その後、会計年度はいつ決めればいいですか？」
  → 「その後」という省略された指示語を、1問目の履歴を踏まえて「初期設定の続き」と
    正しく解釈し、会計年度設定について回答した
```

Cosmos DBへの保存も`get_history`で確認し、2ターンとも引用元付き(各5件)で保存されていることを確認した。

## 3. Step5: Streamlit簡易チャットUI

### 3.1 回答精度以外に必要な設計

golden_qaで測れるのは回答の中身の精度だけで、「業務担当者が迷わず使えるか」は別の設計判断が要る。今回意識したのは次の3点。

- **質問例ボタン**: 初見の担当者は「何を聞けばいいか分からない」まま離脱しうるため、代表的な質問3つをサイドバーにワンクリックで試せるようにした
- **引用元は既定で折りたたみ**: 回答本文を主役にしつつ、根拠を確認したい場合はすぐ`st.expander`で展開できるようにした。常時展開だと画面が長くなり本文が読みにくくなる
- **エラーもチャットとして表示**: 例外発生時に画面がブランク化すると「壊れた」と誤解されやすいため、エラーもアシスタントの発言として表示し会話の流れを保つようにした

```python
EXAMPLE_QUESTIONS = [
    "ERPNaviの初期設定はどこから始めればいいですか？",
    "在庫の発注点アラートが届かない場合はどうすればいいですか？",
    "月次締め処理ができないときの原因は何ですか？",
]
```

### 3.2 コードレビューで見つかったバグ:エラーメッセージの詳細が画面に漏れていた

実装当初は例外の`str()`をそのまま画面に埋め込んでいた。Azure SDKの例外メッセージは実装詳細(エンドポイントの一部やリソース名など)を含みうるため、業務担当者向け画面には固定文言のみを表示し、詳細は`logger.exception`でサーバー側ログにのみ残す設計に修正した。

```python
USER_FACING_ERROR_MESSAGE = "エラーが発生しました。担当部署にお問い合わせください。"

def _generate_answer_safely(
    *, query: str, session_id: str, deps: RagDependencies,
    generate_answer_fn: Callable[..., RagAnswer] = generate_answer,
) -> RagAnswer | None:
    try:
        return generate_answer_fn(query=query, session_id=session_id, deps=deps)
    except Exception:
        logger.exception("RAG回答生成に失敗しました (session_id=%s)", session_id)
        return None
```

`generate_answer_fn`を引数化しているのは、Streamlitの描画呼び出し(`st.*`)を含まないこの関数を、実際のAzure接続なしに例外処理の分岐だけ単体テストするため。

この修正の検証は、ダミーの例外を投げるだけでは不十分だと考え、`.env`の`AZURE_OPENAI_API_KEY`を一時的に不正な値に書き換えてアプリを実際に起動し、質問を送信するところまで確認した[^2]。画面には固定文言のみが表示され、APIキーの値や例外メッセージが一切出ないこと、サーバー側ログには`openai.AuthenticationError`のスタックトレースが記録されること(ログ本文にもAPIキーは含まれない、Azure側が汎用的な401エラーを返すため)を確認し、検証後`.env`は正しい値に復元した。

### 3.3 Playwrightでの動作確認

Node.js版Playwright(ヘッドレスChromium)でチャットUIを実際に操作した[^3]。

- 初期表示: サイドバーにセッションID・「新しい会話を始める」ボタン・質問例3件が表示される
- `st.chat_input`から実際に質問を送信し、Azure OpenAI/AI Searchを介した回答と「引用元を表示」エキスパンダーが表示される(2.2節で修正したタイトル重複がないことも目視確認)
- サイドバーの質問例ボタンをクリックしても同様に質問が送信され、回答が返る
- 「新しい会話を始める」ボタンをクリックすると、セッションIDが新しいものに変わり画面上の会話履歴がクリアされる
- ブラウザコンソール・ページエラーは検出されなかった

## 4. Step6: golden_qaによる精度評価

### 4.1 評価手法:Neo4j検証と同じ「キーワード網羅率」

指標にはキーワード網羅率(期待キーワードのうち実際の回答に含まれる割合)のみを採用し、RAGAS(LLM-as-judgeによるfaithfulness等)は見送った。追加のライブラリ依存・評価のたびに発生する追加のLLM呼び出し課金というコストに対し、過去のNeo4j検証と同一手法で比較できるメリットを優先した判断。

```python
def keyword_coverage(answer: str, expected_keywords: list[str]) -> float:
    if not expected_keywords:
        return 0.0
    hits = sum(1 for keyword in expected_keywords if keyword in answer)
    return hits / len(expected_keywords)
```

18問はERPNaviの5マニュアル・FAQ・トラブルシュート事例集の全カテゴリから出題し、3つのチャンク分割方針それぞれで`generate_answer`(本番と同じロジック)に通した。評価用セッションは質問ごとに使い捨てにし、通常の会話履歴と混ざらないようにしている。

### 4.2 結果:僅差だが、チャットUIの既定戦略が最下位という逆転

| チャンク分割方針 | 平均スコア |
|---|---|
| `fixed_512` | 1.000 |
| `fixed_256` | 0.972 |
| `heading_aware` | 0.944 |

2.1節で書いた通り、チャットUIの既定戦略には`heading_aware`を選んでいる。理由は引用元の`section_path`が意味のある単位でまとまり、業務担当者にとって読みやすいという定性的な判断だった。ところがgolden_qaでは、その`heading_aware`が3戦略中もっとも低いスコアになった。

ただし差はわずか(1.000と0.944)で、18問中16問は3戦略とも満点だった。差が出たのは以下の2問のみ。

- **fixed_256 / q14**(新規仕入先登録の必須チェック、スコア0.50): 回答は「反社会的勢力チェックリストとの照合」を正しく説明していたが、期待キーワードの一つ「コンプライアンスチェック」(原文の用語)を使わずに言い換えていたため未マッチだった
- **heading_aware / q15**(支払サイト変更の遡及適用、スコア0.00): 回答は「既存の買掛金には適用されません」と内容自体は正しいが、期待キーワード「遡及適用」という語をそのまま使わずに言い換えていたため未マッチだった

この2件だけを見て「`heading_aware`は精度が低い」と結論づけるのは早計だと考えている。18問という小規模な質問数に対し差はわずか2問分であり、個々の生成の言い回しのブレに起因する可能性の方が高い。より確度の高い比較には質問数を増やすか、言い換えに頑健なLLM-as-judge等の指標を併用する必要があるというのが率直な結論。それでも「引用元の読みやすさを優先して選んだ既定戦略が、キーワード網羅率では最下位だった」という事実そのものは、精度指標と使い勝手のどちらを優先するかという設計判断が実際にトレードオフになりうることを示す実例として記録しておきたい。

### 4.3 golden_qa側のバグを2件発見・修正

評価スクリプトの初回実行時、以下2問が3戦略とも異常に低いスコアだった。回答文を実際に読んだところ、RAGの回答自体は正しく、`expected_keywords`の表記が回答と噛み合っていない、golden_qa側のバグだと判明した。

- **q05**(SSO連携の注意点): 期待キーワードに`"SAML"`を指定していたが、検索結果の該当チャンクには`SAML`という語自体は含まれており、モデルは同義語の`IdP`で説明することが多かった。検索の見落としではなく生成側の言い換え選択だったことを、実際に検索でヒットした上位チャンクの内容まで確認した上で判断した。キーワードを`"IdP"`に変更し、3戦略とも1.00に改善
- **q15**(初回発見時): 期待キーワードを`"遡及適用されず"`(活用形まで固定)にしていたため、回答が`"遡及適用されません"`という自然な言い換えをしただけで不一致になっていた。活用語尾を含まない`"遡及適用"`に変更し、`fixed_512`/`fixed_256`は1.00に改善(`heading_aware`は4.2節の通り別の言い換えでなお0.00のまま残った)

修正は既存の回答文をそのまま再採点する形で行い、追加のAzure OpenAI呼び出しは発生させていない。

## 5. 得られた知見

### 5.1 キーワード網羅率は「同じ内容の言い換え」を拾えない

4.2節・4.3節で見た4件(q05・q14・q15×2戦略)は、すべて「回答内容は正しいが、期待キーワードと違う言葉を使ったために未マッチになった」ケースだった。これはNeo4jグラフRAG検証でも観測された、この評価手法に共通の弱点で、意外な発見ではない。ただ今回改めて確認できたのは、この弱点は「評価データの作り方が甘かった」で全て説明がつくわけではないという点だ。q05・q15の初回バグは評価データ側の表記の問題として修正できたが、q14・q15(heading_aware)は評価データを直しても残った。後者は「モデルがどう言い換えるか」という生成側の自由度に起因しており、評価データをどれだけ精緻にしても原理的に拾いきれない部分がある。

### 5.2 Neo4jグラフRAGとの比較:単純な優劣比較はできない

過去のNeo4j×LangChain Agentによるグラフ×ベクトルRAG検証は、シリーズ最終的に9問全問でscore=1.00に到達している。今回のAzureネイティブ構成は18問でfixed_512が1.000、他の2戦略も0.94〜0.97という結果だった。数字だけ見れば「両方とも高精度」で終わってしまうが、公平な比較のためにいくつか前提の違いを明記しておきたい。

- **質問数が異なる**(9問 vs 18問)。母数が倍以上違うため、同じ「全問正解に近い」という結果でも重みが異なる
- **題材・ドメインが異なる**(設備保全の点検記録という時系列的・関係性の強いデータ vs ERP業務マニュアルという比較的独立した文書群)。Neo4jのグラフ構造が効くのは「過去の点検履歴を踏まえた抽出」のような関係性の強いクエリで、今回のERPNaviのFAQ・マニュアル検索はそもそもグラフ構造の恩恵を強く必要とするデータ形状ではない可能性がある
- **Neo4j検証側も、9問全問満点に到達した経緯で期待キーワードを緩めた質問が2問あったことを、その記事自身が明記している**。「評価基準を実装に合わせて緩めていないか」という自問は、Neo4j検証・今回のAzure検証の両方で共通して必要になった

この2つの検証を通じて言えるのは、「どちらのアーキテクチャが優れているか」という結論ではなく、「キーワード網羅率という評価手法そのものが、アーキテクチャによらず同じ種類の穴(言い換えに弱い)を持つ」ということだと考えている。アーキテクチャ選定は、精度の数字だけでなく実装コスト・運用のしやすさ・題材とのデータ形状の相性で判断すべき、という当たり前の結論に落ち着いた。

### 5.3 精度指標と使い勝手はトレードオフになりうる

4.2節の通り、チャットUIの既定戦略に選んだ`heading_aware`はgolden_qaで最下位だった。今回は差がわずかだったため既定戦略を変更する判断はしていないが、もし差が大きければ「引用元の読みやすさ」と「回答精度」のどちらを優先するかという意思決定が必要になっていたはずだ。精度評価は「どの設定が優れているか」を機械的に決めるためのものではなく、こうしたトレードオフの存在を可視化するためのものだと捉えている。

## 6. まとめと今後

Step4(RAG回答生成・引用元提示)、Step5(Streamlitチャット UI)、Step6(golden_qa精度評価)を実装し、実リソースに対する検証を通じて、引用元タイトルの二重表示・`content=None`未処理・エラーメッセージの画面漏れ・golden_qa側のキーワード表記ミス2件という実際の問題を発見・修正した。golden_qaでは3つのチャンク分割方針を比較し、いずれも僅差ながら`fixed_512`が最高スコアだったこと、チャットUIの既定戦略がキーワード網羅率では最下位だったことを確認した。

この記事は全3回シリーズの第2回です。

- **第1回**: [Step1〜3(インデックス設計・インジェストパイプライン・セッション管理)](https://zenn.dev/yuninaka/articles/azure-rag-navigator-build)
- **第2回(本記事)**: Step4〜6(RAG回答生成・引用元提示・チャットUI・golden_qa精度評価)
- **第3回(予定)**: Step7〜8(Azure App Serviceデプロイ・GitHub Actions CI/CD・Bicepによる IaC化)。マネージド構成での運用上の利点・注意点を扱う予定

次回は実際にAzure App Serviceへデプロイし、CI/CDパイプラインとBicepによるIaC化、そして「読まなくても壊れないコードベース」を目指した品質ゲート整備の記録に進みます。

[^1]: `uv run python scripts/ask_question.py "<質問文>" <セッションID>`で1問1答形式の動作確認ができる。同一セッションIDを指定すると履歴が引き継がれる。今回は`uv run python scripts/ask_question.py "ERPNaviの初期設定はどこから始めればいいですか？" demo-step4-verify`に続けて同一セッションIDで2問目を投げ、Cosmos DBの`get_history`で保存内容も確認した。
[^2]: `uv run streamlit run src/app/streamlit_app.py --server.headless true`でアプリを起動し、`.env`の`AZURE_OPENAI_API_KEY`を検証用に書き換えた状態でブラウザから質問を送信、画面表示とサーバー側ログの両方を確認した。
[^3]: Playwrightの`chromium.launch(headless=True)`でブラウザを起動し、`page.goto`でStreamlitの起動URLにアクセスした上で、`page.get_by_role`等でサイドバーのボタン・`st.chat_input`を実際にクリック・入力して動作を確認した。
