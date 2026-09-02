---
title: "OCRの誤読をLLMが勝手に直した話——グラフRAGパイプライン完成編"
emoji: "🧩"
type: "tech"
topics: ["azure", "ocr", "rag", "llm", "claude"]
published: true
---

## 1. 概要

### 1.1 やったこと(3行サマリ)

- 前回記事([デジタルネイティブPDFなのに「O」を「〇」と誤読した、Document Intelligenceの話](https://zenn.dev/yuninaka/articles/zenn-graph-rag-ocr-integration))で「未実装」として残した、**OCR結果を既存のナレッジグラフ構築パイプラインに接続する変換層**を実装した
- Document Intelligenceの構造化レスポンス(`tables`/`paragraphs`)を使うことで、前回`result.content`(生テキスト)ベースの比較で見つかった誤り(`:unselected:`タグの混入)がそもそも再現しないことに気づいた
- OCRの誤読(「O」を「〇」と誤認識)を人手で補正せずそのまま下流に流したところ、**LLMによるエンティティ抽出段階で「Oリング」という一般的な部品名として自動的に補正され**、最終的な回答精度に一切影響しなかった

リポジトリ: [graph-rag-maintenance-demo](https://github.com/yuninaka/graph-rag-maintenance-demo)
(この記事時点のコード: [`article-5-pipeline-completion`タグ](https://github.com/yuninaka/graph-rag-maintenance-demo/tree/article-5-pipeline-completion))

### 1.2 結果だけ先に

| データソース | evaluate.pyスコア |
|---|---|
| 合成データ(`maintenance_logs.jsonl`)直接使用 | 全体1.00 |
| **OCR由来データ**(PDF→Document Intelligence→変換層) | **全体1.00** |

「紙の書類(PDF)→OCR→ナレッジグラフ→ハイブリッド検索→回答」という一気通貫パイプラインが完成し、OCRの入り口で発生した誤読が最終的な回答精度に影響しないことを実測できた。

## 2. 前回までのおさらい:何が未実装だったか

前回記事では、Azure Document Intelligenceで35件の点検記録表PDFをOCRし、平均類似度0.9990という高精度を確認した。しかしそこで検証したのは「OCRの抽出精度」までで、**その先が繋がっていなかった**。

```
[前回まで]
[点検記録表(PDF)] → Document Intelligence(OCR) → (ここで終わり、比較のみ)

[今回]
[点検記録表(PDF)] → Document Intelligence(OCR) → 変換層 → build_knowledge_graph.py → Neo4j
```

OCRの結果(`result.content`という文字列)を、既存の`build_knowledge_graph.py`が読み込む`{report_id, date, equipment, reporter, text}`という構造化レコードに変換する層がなければ、パイプラインとして完成しない。今回はここに手を付けた。

## 3. 変換層の設計:`content`ではなく`tables`/`paragraphs`を使う

前回は`result.content`(ページ全体を連結した生テキスト)を正解データと比較していたが、今回は個々のフィールドを正確に切り出す必要がある。そこでDocument Intelligenceが返す、もう少し構造化されたレスポンスを使うことにした。

```python
print('=== tables ===')
for t in result.tables:
    for cell in t.cells:
        print(f'[{cell.row_index},{cell.column_index}] {cell.content!r}')
```

実際に叩いて確認したところ、点検日・対象設備・報告者は**3行2列のテーブル**として認識されていた。

```
[0,0] '点検日'       [0,1] '2026-01-15'
[1,0] '対象設備'     [1,1] 'コンベアA'
[2,0] '報告者'       [2,1] '田中'
```

report_idと本文は`paragraphs`(段落ごとの構造)から、それぞれ`pageHeader`ロールの内容(「報告書No. R001」)と、「状況・原因・対処内容」ラベルの次から「確認済」スタンプの手前までの段落として抽出した。

```python
def extract_meta_from_table(result):
    table = result.tables[0]
    cells = {(c.row_index, c.column_index): c.content for c in table.cells}
    return {"date": cells[(0,1)], "equipment": cells[(1,1)], "reporter": cells[(2,1)]}

def extract_body_text(result):
    paragraphs = [p.content for p in result.paragraphs]
    label_idx = next(i for i, p in enumerate(paragraphs)
                      if "状況" in p and "原因" in p and "対処内容" in p)
    end_idx = next((i for i in range(label_idx+1, len(paragraphs))
                     if paragraphs[i].strip() == "確認済"), len(paragraphs))
    return "".join(paragraphs[label_idx+1:end_idx])
```

方針として、**OCR出力は人手で補正しない**ことにした。Document Intelligenceが実際に返した文字列(誤読を含む)をそのまま下流の抽出パイプラインに流し、ノイズがどう伝播するかを実測することが今回の狙いだったからだ。

## 4. 発見1: 見るAPIフィールドによって「見える誤り」が変わる

35件を変換して正解データと突き合わせたところ、興味深いことに気づいた。前回記事で見つけたR013の誤り(`:unselected:`タグの混入)が、**今回は発生しなかった**。

実際にR013の`paragraphs`を1件ずつ確認すると、「確認済」というスタンプは通常の段落として認識されており、`:unselected:`のような文字列はどこにも見当たらない。

```
8 '状況 · 原因 ·対処内容' None
9 '搬送ロボット1号機が経路の途中で急停止...(本文)' None
10 '確認済' None
```

前回の`:unselected:`は`result.content`(全パラグラフを結合した生テキスト)にのみ現れる副作用だったようだ。おそらく、Document Intelligenceが図形要素(印影の丸枠)をチェックボックス的な選択マークとして検出した際のメタ情報が、`content`という統合ビューに混入する形で出力され、一方で`paragraphs`という個別の構造化ビューにはこの種の注釈が含まれない、という実装上の違いがあると考えられる。**同じOCR結果でも、どのAPIフィールドを読むかによって「見える誤り」が変わる**、というのは実務上覚えておく価値のある教訓だと思う。

## 5. 発見2: 自分の検証スクリプトのミスにも気づいた

35件のテキストフィールドを正解データと比較したところ、最初は35件全てで不一致という結果になった。慌てて中身を見ると、ほとんどは前回記事で既に発見済みの「行の折り返し位置に余計なスペースが挿入される」という書式ノイズだった。空白を除去して再比較したところ、35件中32件は完全一致、残り3件(R007、R012、R017)に差分が残った。

R012は既知の「O→〇」誤読として予想通りだったが、**R007とR017は前回記事では検出されていなかった**新顔だった。中身を見ると、原因はすぐに分かった。

```
正解: フィルターを清掃・交換し、
OCR : フィルターを清掃·交換し、
```

これも中黒(・)が別のUnicode文字(·)に置き換わる、前回から分かっていた表記ゆれだった。ただし今回は**タイトルやラベルの中黒ではなく、本文中の「清掃・交換」のような列挙表現の中黒**で発生していた。前回作った比較スクリプト(`ocr_document_intelligence.py`)ではこの中黒正規化を既に入れていたが、今回新しく書いた検証コードでは空白除去しか入れておらず、正規化し忘れていたのが原因だった。中黒も正規化して再確認したところ、35件中34件が完全一致、残るのはR012のみになった。

**前回すでに直した正規化処理を、新しい検証コードで再現し忘れる**というのは、地味だが実際によくあるミスだと思う。過去に見つけた表記ゆれのリストは、検証コードを書き直すたびに引き継ぐ必要がある。

## 6. パイプライン全体の実行

`build_knowledge_graph.py`に`MAINTENANCE_LOGS_PATH`環境変数を追加し、データソースを合成データからOCR由来データに差し替え可能にした。

```bash
# OCR由来データでナレッジグラフを構築
MAINTENANCE_LOGS_PATH=data/maintenance_logs_ocr.jsonl python src/build_knowledge_graph.py
```

Neo4jを再構築したところ、Equipmentノードは想定通り8設備・8ノードに正しく統合され、総ノード数も合成データを直接使った場合とほぼ同じ規模になった。

## 7. 最重要発見:OCRの誤読はLLM抽出段階で自動修正された

ここが今回一番面白かった部分だ。R012のOCR由来テキストには「継手の**〇**リング(型番OR-14)を交換」という誤読がそのまま含まれている。これを`build_knowledge_graph.py`のLLM抽出に通した結果は、こうなった。

```python
{
  'action': '継手のOリング交換、冷媒再充填',
  'part': 'Oリング(型番OR-14)',
  ...
}
```

**入力は「〇リング」だったのに、出力は「Oリング」に直っている。**

LLMは「Oリング」という一般的な工業部品名(ゴム製の環状パッキン)を知識として持っているため、文脈(「型番OR-14」「継手」「交換」)から、視覚的な誤読(〇)を意味的に正しい表記(O)へと、意識せず補正したのだと考えられる。これは自分が明示的に指示したことではなく、LLMによるエンティティ抽出という処理の**副産物として自然に起きた**。

実際にこのOCR由来データで`evaluate.py`を実行したところ、9問全問がscore=1.00となり、合成データを直接使った場合と完全に同じ結果になった。R012の誤読は、最終的な回答精度には一切影響しなかった。

## 8. 得られた知見

### 8.1 OCRの誤りが即・パイプライン全体の誤りになるとは限らない

前回記事では「デジタルネイティブPDFでも視覚的な誤読が起きる」というOCR単体の弱点を発見した。今回はその続きとして、「OCRが多少間違えても、後段のLLM抽出が一般常識で吸収してくれる場合がある」という、パイプライン全体で見たときの頑健性を実測できた。これはOCR単体の精度評価だけでは見えてこない視点で、**パイプラインを実際に最後まで繋いで動かして初めて分かること**だった。

ただし、これは「Oリング」という非常に一般的で文脈から推測しやすい用語だったから起きたことでもある。固有名詞や型番の数字部分が誤読された場合(例えば型番「OR-14」の数字自体がOCRで誤読されていたら)、LLMが正しく補正できたとは限らない。「LLMが誤読を直してくれるから、OCRの精度は気にしなくていい」という結論には飛躍しないよう注意したい。

### 8.2 「どのAPIフィールドを読むか」で見える誤りが変わる

同じOCR実行結果でも、`result.content`(生テキスト)を見るか`result.paragraphs`(構造化データ)を見るかで、観測される誤り(前回のR013の`:unselected:`)が変わった。「OCRサービスの精度」という一括りの評価ではなく、「どのAPIをどう使うか」まで含めて検証しないと、実態を正確に捉えられないと実感した。

### 8.3 過去に直した正規化処理は、検証コードを書き直すたびに引き継ぐ必要がある

5章で書いた通り、前回のスクリプトで既に対応していた中黒の正規化を、今回新しく書いた検証コードで一瞬忘れていた。これは実装のバグというより、**過去の知見をどう資産化して次のコードに引き継ぐか**という運用上の課題だと感じている。今回は`docs/troubleshooting_log.md`に記録が残っていたおかげで、差分を見た瞬間に「あ、これ前回対応した中黒のやつだ」と気づけた。記録を残す習慣の効果を、地味だが実感できた場面だった。

## 9. まとめと今後

前回記事で残した「OCR結果を既存パイプラインに接続する」という課題を実装し、「紙の書類→OCR→ナレッジグラフ→ハイブリッド検索→回答」という一気通貫パイプラインを完成させた。OCRの誤読(R012)がLLM抽出段階で自然に補正され、最終的な回答精度に影響しなかったという発見は、単体の精度評価だけでは得られない、パイプライン全体を通したからこその知見だった。

残っている課題は次の2つ。

1. **本物のスキャン/写真での検証**: 今回もデジタルネイティブPDFでの検証にとどまっており、紙に印刷してスキャンした画像や、手書き文字を含む書類でどこまで精度が出るか、そしてLLM抽出がそのノイズをどこまで吸収できるかは未検証
2. **ベクトルインデックス側のOCR対応**: 今回`build_knowledge_graph.py`(ナレッジグラフ構築)側のみOCR由来データに対応させた。`build_vector_index.py`(ベクトルインデックス)側は合成データのままで、こちらも接続すれば本当の意味で完全なパイプラインになる

次回は、これらのいずれか、あるいは合成データの拡張に進みたい。

---

**このシリーズはこれまで5記事構成です。**

- 第1弾: [グラフRAG × ベクトルRAG 個人実証:設備保全ナレッジベースで見えた限界](https://zenn.dev/yuninaka/articles/zenn-graph-rag-limitations)
- 第2弾: [グラフRAGの「名寄せ」を直したら、直した先で別のバグが見つかった話](https://zenn.dev/yuninaka/articles/zenn-graph-rag-symptom-normalization)
- 第3弾: [件数が増えても壊れない「履歴を踏まえた抽出」をグラフRAGに実装する](https://zenn.dev/yuninaka/articles/zenn-graph-rag-backreference-detection)
- 第4弾: [デジタルネイティブPDFなのに「O」を「〇」と誤読した、Document Intelligenceの話](https://zenn.dev/yuninaka/articles/zenn-graph-rag-ocr-integration)
- 第5弾(本記事): OCR結果を既存パイプラインに接続し、誤読の伝播を実測
