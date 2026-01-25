# 1. 全体像

結論：高速に候補を集める“Embedding”と、精密に並べ替える“Reranker”の2段構え。
テキスト/画像/文書画像/動画を同一の表現空間で扱う。

```全体図
文書画像/テキスト/動画
          |
          | ① Embeddingでベクトル化（事前計算して保存できる）
          v
   [Vector Index / ANN検索]
          |
          | ② 類似Top-K候補を取得（高速）
          v
   (候補K件だけ)
          |
          | ③ Rerankerで精密スコアリング（遅いが精密）
          v
     [最終順位 / 最終割当]

```
<br>
# 2. Embeddingは何をしているのか
# 2.1 全体図

Dual-Tower / Bi-encoderという、２本の軸を持つモデルで動いている。
Query（探したいもの） と Document（候補） を 別々にモデルへ入れて、
それぞれ 固定長ベクトルに変換し、2つのベクトルの近さ（主にcosine類似度）で「関連度っぽさ」を測る。

```Dual Tower
           ┌───────────────┐
Query ---->│ Encoder (Tower) │----> q_vec (ベクトル)
           └───────────────┘

           ┌───────────────┐
Doc  ----->│ Encoder (Tower) │----> d_vec (ベクトル)
           └───────────────┘

score = cosine(q_vec, d_vec)
```

## 2.2 実際に用いる埋め込みベクトルはどこから取る？
Embeddingは ベースモデル最終層の[EOS]トークンの隠れ状態を最終表現として取り出している。

各層 𝑙=1..𝐿

各位置 𝑡=1..𝑇

最終層の [EOS] トークンの隠れ状態とは、最終層 l＝L の出力（= last_hidden_state）
その中の [EOS]（文末を表す特殊トークン）が置かれた位置のベクトルのこと。下図の星の部分

```[EOS]の概念図

                  位置 t（トークンの並び）
        t=1     t=2     t=3         ...     t=T-1    t=T(EOS)
l=1   [ ● ]   [ ● ]   [ ● ]        ...     [ ● ]    [ ● ]
l=2   [ ● ]   [ ● ]   [ ● ]        ...     [ ● ]    [ ● ]
l=3   [ ● ]   [ ● ]   [ ● ]        ...     [ ● ]    [ ● ]
 ...    ...     ...     ...                 ...      ...
l=L   [ ● ]   [ ● ]   [ ● ]        ...     [ ● ]    [ ⭐︎ ]
```
## 2.3 なぜ最終層の [EOS] トークンの隠れ状態をとるだけで全文・全画像を代表できるのか

Qwen3-VL のバックボーンが **causal attention（自己回帰マスク）**を用いているから。
自己回帰モデルでは、位置tの表現はそれ以前の全トークンを参照して更新されるため、最後のトークンは、前にある全てを見ている。

## 2.3 「隠れ状態」って何ですか？
直感的には、**各トークンの“理解メモ**

例として
```
私は りんご を 食べた
```

をトークン列として入れると、モデルは各位置に対してこんな「内部メモ」を作る：

「私は」位置の hidden state：主語っぽい・人間っぽい…などの特徴
「りんご」位置の hidden state：食べ物っぽい・目的語っぽい…などの特徴
「食べた」位置の hidden state：動詞・過去形・文末…などの特徴

実際には「食べ物っぽい」みたいなラベルが付いてるわけではなく、2048次元や4096次元の数値のベクトルとして表現されている。 


より具体的に、Qwen3-VL-EmbeddingはTransformerを用いているが、
Transformerは層が積み重なっていて、層ごとに「各トークンの表現」を更新する。

```
0層（埋め込み直後）：単語そのもの＋位置情報のベクトル
1層：周囲（前文脈）の情報を混ぜた表現
…
最終層：タスクに使いやすい、より抽象的な表現
```

この 各層の各位置で出るベクトルが hidden state 。

「隠れ状態」は昔のニューラルネット（RNNなど）由来の呼び方。
モデル内部の状態（外に直接見せる出力ではない）が、そこから最終出力（次トークン予測、分類、埋め込みなど）を作るという意味で “hidden” と呼ぶ。
<br>
# 3. Rerankerは何をしているのか
Single-Tower / Cross-encoderで動く。
Query と Doc を 一緒にモデルへ入れて、cross-attentionで「Queryの各要素」と「Docの各要素」を突き合わせる。
**“このペアは関連する？”**を精密に判定する。

```
[Instruction][Query][Doc] ---> Single Encoder (Cross-Attention) ---> score
```
Rerankerでは関連度スコアを 特別トークンyes/noの生成確率で表現している。

Transformerの注意機構（attention）は、ざっくりこう
![118F7B21-4504-4E6E-8017-AF453DC247C5_4_5005_c](https://github.com/user-attachments/assets/cfc7163b-3732-46f4-8098-326afac3eb2a)

 ・Q（Query）：どこを見たいか
・K（Key）：各トークンの「見出し」
・V（Value）：各トークンの「中身」

「Query側のトークンが、Doc側のトークンを見に行く」のが cross-attention のコア
<br>
# 4.入力フォーマットについて

Embedding側は instructionをsystem messageとして渡し、デフォルトは “Represent the user’s input.”としている。
まとめるとこんな感じ

```
Embedding:
[Instruction][Query/Doc][PAD]  --->  last hidden states  ---> embedding

Reranker:
[Instruction][Query][Doc][ASSISTANT] ---> LM head ---> P("yes") 等 ---> score
```
<br>
# 5.学習方法について
# 5.1　CLIPとの類似性
CLIPと似たシステムを採用している。損失関数はInfoNCE（コントラスト学習）を用いている。
ここで、CLIP＝「画像エンコーダ＋テキストエンコーダのモデルと学習枠組み」
InfoNCE＝「対照学習で使う損失関数」くらいのイメージ。
CLIPとInfoNSEの違いは２つ

違い1：対象
・InfoNCEは何にでも使える一般の損失
・CLIPは「画像とテキストのbi towerモデル」＋「その損失で学習する」という具体的なシステム

違い2：CLIPは 双方向（対称）
InfoNCEは片方向でも定義できるが、CLIPは基本 画像→テキスト と テキスト→画像の両方を足す構成.

# 5.2 InfoNSEのしていること
InfoNCE（Information Noise-Contrastive Estimation）は、1つのクエリ q に対して
・正例：𝑑<sup>+</sup>
・負例：𝑑^<sup>-</sup>
の中から **「正例を当てる多クラス分類」**をさせて損失関数を出す。

典型形は以下の通り：
<img width="544" height="72" alt="Screenshot 2026-01-25 at 12 17 25" src="https://github.com/user-attachments/assets/2e96b8f2-c70e-4a4e-a95b-d8895180087c" />

**「正例を、負例集合の中で当てる」**という softmax分類 をやっていて、その **負の対数尤度（クロスエントロピー）**を損失関数としている

ここで各記号の意味は以下の通り

q：クエリ（query）。
例：検索なら「質問文」、CLIPなら「画像 or テキスト」の片方。
embeddingだと埋め込みベクトルのこと。

s(q,d)：類似度スコア（similarity）
qと𝑑の近さを数値化したもの。大きいほど似ている。
今回はコサイン類似度を用いている。
<img width="419" height="80" alt="Screenshot 2026-01-25 at 12 24 28" src="https://github.com/user-attachments/assets/165a1cec-4efd-438c-b1c5-df0d553decf4" />
L2正規化している場合、コサイン類似度は [−1,1] に収まる。

τ：温度
・τが小さい → softmaxが尖る → 「少しの差でも強く勝敗がつく」＝分離が強くなるが不安定にもなりやすい
・𝜏が大きい → softmaxがなだらか → 学習は安定するが分離が弱くなりやすい

