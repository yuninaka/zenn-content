---
title: "デジタルネイティブPDFなのに「O」を「〇」と誤読した、Document Intelligenceの話"
emoji: "📄"
type: "tech"
topics: ["azure", "ocr", "rag", "llm", "claude"]
published: true
---

## 1. 概要

### 1.1 やったこと(3行サマリ)

- グラフRAG検証シリーズ([第1弾](https://zenn.dev/yuninaka/articles/zenn-graph-rag-limitations)〜[第3弾](https://zenn.dev/yuninaka/articles/zenn-graph-rag-backreference-detection))で「今後の課題」に挙げていた、Azure Document Intelligenceによる実際のAI-OCR連携を検証した
- AI画像生成でサンプル書類を作る案は日本語テキストの描画品質の問題で早々に断念し、**実データをそのままHTML帳票に流し込んでPDF化する**、正解が確実な方式に切り替えた
- 35件をOCRにかけた結果、平均類似度0.9990という高精度だったが、**デジタルネイティブPDF(テキスト埋め込み済み)なのに視覚的な誤読が発生する**という意外な発見があった

リポジトリ: [graph-rag-maintenance-demo](https://github.com/yuninaka/graph-rag-maintenance-demo)
(この記事時点のコード: [`article-4-ocr-integration`タグ](https://github.com/yuninaka/graph-rag-maintenance-demo/tree/article-4-ocr-integration))

### 1.2 結果だけ先に

| 指標 | 結果 |
|---|---|
| 検証件数 | 35件 |
| 完全一致 | 33件 |
| 平均類似度(正規化後) | **0.9990** |

数字だけ見れば「ほぼ完璧」だが、満点でなかった2件の中身が面白い。1件はアルファベットの「O」を丸記号「〇」と誤読するという、**画像として視覚的にOCRしていることを示す証拠**だった。

## 2. なぜAI-OCR連携が必要だったか

これまでのシリーズでは、`data/maintenance_logs.jsonl`という**最初から構造化済みのテキスト**を入力にしていた。しかし実務では、点検記録は紙の書類やスキャンされたPDFから始まる。README冒頭の仮想要件マッピング表にも「AI-OCRで構造化したデータのグラフ化」という項目があり、これまではこの部分を合成データで代替(スキップ)していた。今回はここに実際に手を付ける。

狙いは、既存の`build_knowledge_graph.py`のLLM抽出部分はそのまま使い、その手前に「スキャン書類→構造化テキスト」という前処理を追加することだった。

```
[これまで]
data/maintenance_logs.jsonl(構造化済み合成データ)
        │
        └─→ build_knowledge_graph.py(LLM抽出)→ Neo4j

[今回検証した部分]
[点検記録表(PDF)]
        │
        └─→ Azure Document Intelligence(OCR + 構造抽出)
```

## 3. サンプル書類の準備:AI画像生成という誤った入口

最初に考えたのは、ChatGPT(DALL-E)で「点検記録表の写真」を生成する案だった。しかし検討の結果、これは筋が悪いと判断して早々に方向転換した。

理由は、**AI画像生成は密度の高い日本語テキストの描画が不得意**だからだ。点検記録表には「日付」「設備名」「症状」「原因」といった項目名と、それに対応する記入内容が大量に含まれる。画像生成モデルにこれを正確に、読める形で描画させるのは現状難しい。仮に生成できても、文字が崩れていたら「正解データ」として使えず、OCR精度の検証そのものが成立しない。

代わりに採用したのは、**既存の実データ(`data/maintenance_logs.jsonl`)をそのままテキストとしてHTML帳票に流し込み、PDF化する**という方式だ。文字は100%正確なまま、書類らしい見た目を作れる。

```python
HTML_TEMPLATE = """<!DOCTYPE html>
...
<div class="report-id">報告書No. {report_id}</div>
<h1>設備点検・トラブル報告書</h1>
<table class="meta">
  <tr><th>点検日</th><td>{date}</td></tr>
  <tr><th>対象設備</th><td>{equipment}</td></tr>
  <tr><th>報告者</th><td>{reporter}</td></tr>
</table>
<div class="body-box">
  <div class="label">状況・原因・対処内容</div>
  {text}
</div>
"""
```

PDF化には`weasyprint`(純Python実装のHTML/CSS→PDF変換ライブラリ)を採用した。Playwrightのようなヘッドレスブラウザを使う方法も検討したが、ブラウザバイナリのダウンロードが不要で軽量な点を優先した。

```python
from weasyprint import HTML
HTML(string=html, base_url=str(OUTPUT_DIR)).write_pdf(pdf_path)
```

`python scripts/render_inspection_form.py`(引数なし)で、35件全てのHTML+PDFを一括生成できるようにした。

## 4. Document Intelligenceとの連携

Azureで無料枠(F0)のDocument Intelligenceリソースを作成し、`prebuilt-layout`モデル(汎用的なレイアウト解析+テキスト抽出)にPDFを投げる。SDKは`azure-ai-documentintelligence`(バージョン1.0.2)を使用した。

```python
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.core.credentials import AzureKeyCredential

client = DocumentIntelligenceClient(endpoint=ENDPOINT, credential=AzureKeyCredential(KEY))
with open(pdf_path, "rb") as f:
    poller = client.begin_analyze_document("prebuilt-layout", body=f)
result = poller.result()
extracted_text = result.content
```

抽出結果(`result.content`)を、正解データ(元のJSONLの内容を帳票テンプレート通りに組み立てたもの)と`difflib.SequenceMatcher`で比較し、類似度スコアを算出する。

### つまずき:単純な文字列比較だとノイズを誤りとしてカウントする

1件目(R001)で試したところ、類似度は0.9574だった。しかし差分を見ると、**文字の中身自体はすべて正しく認識**されており、ズレていたのはレイアウト由来の違いだけだった。

- 中黒「・」が視覚的に似た別のUnicode文字「·」として認識される
- 「点検日 2026-01-15」のようにラベルと値が同じ行にある部分が、OCR結果では改行で分かれる
- PDFの行の折り返し位置に、余計なスペースが挿入される(「摩耗しており」→「摩 耗しており」)

これらは文字認識の誤りではなく書式面の差なので、比較前に正規化する処理を追加した。

```python
def normalize(text: str) -> str:
    text = text.replace("·", "・")
    text = re.sub(r"\s+", "", text)  # 空白・改行をすべて除去
    return text
```

正規化後、R001の類似度は1.0000になった。ここで初めて「文字認識自体は完璧」と確認できた。

## 5. 35件全件実行:見つかった2つの誤り

正規化した比較ロジックで35件全件を実行した。結果は冒頭の通り、33件が完全一致、平均0.9990。残る2件の中身を詳しく見る。

### R012: アルファベット「O」を丸記号「〇」と誤読

正解: 「継手の**O**リング(型番OR-14)を交換」
OCR結果: 「継手の**〇**リング(型番OR-14)を交換」

アルファベットの大文字「O」が、視覚的に似た丸記号「〇」に置き換わっていた。類似度は0.9944で、ほぼ全問正解に近いが、この1文字だけ違っていた。

### R013: 印影を選択マークと誤検出

OCR結果の末尾に、正解データには存在しない`:unselected:`というタグが付加されていた。類似度は0.9695。帳票右下に配置した「確認済」という円形の印影(赤い枠のスタンプ)を、Document Intelligenceが**チェックボックスのような選択可能な要素として誤検出**し、「未選択」と判定したためと考えられる。他の33件にも同じスタンプがあるが、この誤検出が起きたのはR013だけだった。

## 6. なぜこの2つの誤りが重要な発見なのか

**このPDFはweasyprintで生成した、文字がテキストとして埋め込まれている「デジタルネイティブPDF」である。** 素朴に考えれば、Document IntelligenceがPDFのテキストレイヤーをそのまま抽出するなら、文字レベルの誤認識は起きないはずだった(コピー&ペーストで正確な文字列が取れるのと同じ理屈)。

しかし実際には、視覚的に類似した文字を混同する(O→〇)という、**紙をスキャンした画像に対する視覚的なOCR特有の誤り**が発生した。この事実は、Document Intelligenceが単純にPDFの埋め込みテキストを再利用しているのではなく、**実際にページを画像として扱い、視覚的な認識処理を行っている可能性が高い**ことを示している。

これは実務上も意味のある発見だと思う。「デジタルで作成されたPDFだから文字は完璧に取れるはず」という思い込みは正しくなく、実際のOCRサービスを使う以上、視覚的な誤読のリスクは(たとえテキスト埋め込みPDFであっても)常に考慮する必要がある。

## 7. 得られた知見

### 7.1 「正解が確実なテスト素材」を作る手間は惜しまない方がいい

AI画像生成でサンプル書類を作る案を早々に見送り、実データをそのままテキストとして流し込む方式に切り替えたのは、結果的に正しい判断だった。もし画像生成した書類でテストしていたら、生成されたテキストが正しいのかOCRの結果が正しいのか区別がつかず、今回のR012・R013のような**具体的で再現性のある発見**には辿り着けなかったはずだ。テスト素材の正確性に妥協すると、後段の検証結果の解釈がすべて曖昧になる。

### 7.2 「ほぼ100%」の中の「ほぼ」に価値がある

平均類似度0.9990という数字だけを見て「ほぼ完璧、以上」で終わらせることもできた。しかし残り0.1%の中身(33件と35件の差)を実際に見に行ったことで、Document Intelligenceの実際の処理方式についての具体的な仮説(視覚的なOCRを行っている)が得られた。第3弾記事で書いた「評価指標が検知できない誤りは目視でしか見つからない」と同じ教訓が、今回もそのまま当てはまった。

### 7.3 軽量な選択肢を優先することの効果

PDF化にPlaywright(ヘッドレスブラウザ)ではなくweasyprint(純Python実装)を選んだのは、依存関係を軽くするための判断だったが、結果として実装・実行がシンプルになり、35件の一括生成もスムーズだった。今回のような単純な帳票レイアウトであれば、フルブラウザエンジンは過剰な選択肢だったと思う。

## 8. まとめと今後

Azure Document Intelligenceによる実際のAI-OCR連携を検証し、平均類似度0.9990という高精度を確認できた。同時に、デジタルネイティブPDFでも視覚的な誤認識が起きるという、実務上重要な発見も得られた。

一点、正直に書いておくと、**今回検証したのは「OCRの抽出精度」までで、その先の「OCR結果を`build_knowledge_graph.py`のパイプラインに接続する」部分は未実装**のままだ。`result.content`から`equipment`・`symptom`・`cause`等の構造化フィールドへのマッピング層を書けば、README冒頭で構想していた「紙の書類→OCR→ナレッジグラフ→ハイブリッド検索→回答」という一気通貫のパイプラインが完成する。

残っている課題は次の3つ。

1. **OCR結果を既存パイプラインに接続する変換層の実装**: `result.content`(または`key_value_pairs`)を、既存の`load_records()`が期待する形にマッピングする
2. **本物のスキャン/写真での検証**: 今回はデジタルネイティブPDFでの検証にとどまっており、実際に紙に印刷してスキャンした画像や、手書き文字を含む書類でどこまで精度が出るかは未検証
3. **もっと広いデータでの検証**: 記事3から持ち越している、件数が数百〜数千件規模になったときの挙動の検証

次回は、これらのいずれかに進みたい。

---

**このシリーズはこれまで4記事構成です。**

- 第1弾: [グラフRAG × ベクトルRAG 個人実証:設備保全ナレッジベースで見えた限界](https://zenn.dev/yuninaka/articles/zenn-graph-rag-limitations)
- 第2弾: [グラフRAGの「名寄せ」を直したら、直した先で別のバグが見つかった話](https://zenn.dev/yuninaka/articles/zenn-graph-rag-symptom-normalization)
- 第3弾: [件数が増えても壊れない「履歴を踏まえた抽出」をグラフRAGに実装する](https://zenn.dev/yuninaka/articles/zenn-graph-rag-backreference-detection)
- 第4弾(本記事): Azure Document Intelligenceとの実際のAI-OCR連携を検証
