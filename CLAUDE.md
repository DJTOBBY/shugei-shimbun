# 手芸新聞

手芸・ハンドメイド業界のデジタル新聞。静的HTMLのみで構成され、GitHub Pagesで公開している。

- 公開URL: https://djtobby.github.io/shugei-shimbun/
- リポジトリ: DJTOBBY/shugei-shimbun
- 姉妹紙: [生涯学習新聞](https://djtobby.github.io/shogai-gakushu-shimbun/)（DJTOBBY/shogai-gakushu-shimbun）— 同じ構造・同じ発行手順

## 立て付け（毎号確認すること）

本紙は**個人が試作しているデジタル新聞**であり、実在の刊行物・法人ではない。この旨を `index.html`・`backnumbers.html`・`archive.html`・奥付（12面）に明記し続ける。

**編集長は[ChatGPTのカスタムGPTとして構築されたAI人格](https://chatgpt.com/g/g-6a5dbec4971481919d12695b6ee3df51-shou-yun-xin-wen-bian-ji-chang)であり、実在の人物ではない。** 紙面のどこかで編集長に言及するときは、必ずこの注記とリンクを添える。

## ファイル構成

```
index.html          トップページ（最新号カード＋本紙紹介＋編集長紹介＋RSS購読案内）
backnumbers.html    号一覧
archive.html        記事横断検索＋編集長コラム連載一覧（各号のarticles配列を抽出したJSONを埋め込む）
feed.xml            RSS
sitemap.xml         サイトマップ
robots.txt          / manifest.json
issues/YYYY-MM-DD.html   各号の本体（1ファイル完結。全12面）
icons/              favicon.ico, apple-touch-icon.png, icon-192.png, icon-512.png, icon-master.png
og/YYYY-MM-DD.png   各号のOGP画像（1200×630）
notes/news-pool.md  ネタ帳（紙面ではない内部メモ）
```

2026年7月28日にサイト構造が変わり、`index.html` は号一覧ではなく**トップ／ランディングページ**になった。旧`index.html`の号一覧は `backnumbers.html` に分離されている。

## 新号を発行する手順（7点セット）

**この7つを全て更新すること。1つでも漏れるとリンク切れや情報の不整合になる。**

1. **`issues/YYYY-MM-DD.html`** — 前号をコピーし、全12面・`articles[]`配列・`renderHeadlineBand()`のitems・`ISSUE_DATE`・`ISSUE_FEATURE`・情報取得日を新規内容に書き換える。**使い回しは厳禁**
2. **`index.html`** — `.latest-callout` 内の号数・日付・見出し・リンク先・`og:image`
3. **`backnumbers.html`** — `<ul class="issue-list">` の先頭に新しい `<li>` を追加。`og:image` も更新
4. **`feed.xml`** — `<item>` を先頭に追加
5. **`archive.html`** — 新号のarticles配列を抽出してJSONデータに追記（下記コマンド参照）。`og:image` も更新
6. **`sitemap.xml`** — 新号のURLを追加（`lastmod`は発行日）
7. **`og/YYYY-MM-DD.png`** — 新号1面のOGP画像を生成（下記コマンド参照）

### archive.html 用のデータ抽出

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('issues/YYYY-MM-DD.html','utf8');
const arr=new Function('return '+html.match(/const articles = (\[[\s\S]*?\n  \];)/)[1].replace(/;\$/,''))();
const out=arr.filter(a=>a.articleType!=='SHORTS').map(a=>({
  issueDate:'YYYY-MM-DD', issueLabel:'試作第N号', id:a.id, articleType:a.articleType,
  title:a.title, subtitle:a.subtitle||null, lead:a.lead, category:a.category, tags:a.tags,
  publishedAt:a.publishedAt
}));
console.log(JSON.stringify(out));
"
```
出力を `archive.html` の `const articles = [...]` の**末尾に追記**する（既存データは消さない）。

**注意**: archive.html には2026-07-20〜07-23号の `market-001` / `business-001` / `culture-001` が一字一句同一の本文で埋め込まれたまま残っている（当時の記事使い回しバグの痕跡）。これは実際の各号の中身と一致しているため、**修正せずそのままにする**。新号分を単純に追記すればよい。

### OGP画像の生成

```bash
# 1. toolbarを隠した一時ファイルを作る
python3 -c "
s=open('issues/YYYY-MM-DD.html').read()
s=s.replace('<div class=\"toolbar no-print\">', '<div class=\"toolbar no-print\" style=\"display:none\">',1)
open('/tmp/og-src.html','w').write(s)"

# 2. 撮影（--user-data-dir は必須。省略するとユーザーの通常のChromeに干渉する）
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless \
  --user-data-dir=/tmp/chrome-isolated --disable-gpu \
  --screenshot="og/YYYY-MM-DD.png" --window-size=1200,630 --hide-scrollbars \
  --virtual-time-budget=3000 "file:///tmp/og-src.html"
```

**`--user-data-dir` を必ず付けること。** 2026年8月5日、姉妹紙の作業でこれを省略して起動した結果、ユーザーの通常のChromeで印刷ダイアログが繰り返し開くという副作用が起きた。

（Chromeが無い環境で作業する場合は、OGP画像だけ後回しにして他の6点を先に仕上げてよい。）

## 面構成（全12面）

| 面 | 内容 |
|---|---|
| 1面 | 総合（マストヘッド／見出し帯／一面トップ／サブニュース） |
| 2面 | オピニオン（編集長コラム／編集部論説／今週の特集） |
| 3面 | 分野別ニュース・小欄短信・新着記事一覧 |
| 4面 | ホビー・キルト |
| 5面 | 市場・企業 |
| 6面 | 予告・教室 |
| 7面 | 全国催事暦（表形式） |
| 8面 | 刺繍・刺し子 |
| 9面 | 書籍・工芸 |
| 10面 | 特集・コラム・編集後記 |
| 11面 | 紙面索引 |
| 12面 | 奥付 |

## 記事データモデル

各号の `<script>` 内 `const articles = [...]` に記事オブジェクトを置く。`openArticleDialog()` が「続きを読む」で全文ダイアログを表示し、`?goto=記事ID` または `?goto=面ID` でディープリンクできる（archive.htmlはこれを使う）。**新号にも `?goto=` ハンドラのスニペット（スクリプト末尾）を必ず引き継ぐこと。**

主なフィールド: `id, articleType, title, subtitle, lead, body[], category, tags[], author, authorRole, publishedAt, updatedAt, sourceName, sourceUrl, sourcePublishedAt, factOrOpinion, editorNote, keyPoints[], relatedArticles[]`

記事種別: `NEWS / BREAKING / EDITORIAL / CHIEF_EDITOR / FEATURE / INTERVIEW / MARKET / MATERIAL / TECHNIQUE / CULTURE / BUSINESS / WORLD / OPINION / SHORTS`

`displaySlot: 'auto-op-articles-p2'` を付けると2面に自動でコラム欄が増える。`boxSlot` を付けたSHORTSは3面の該当短信枠に入る。

### 広告枠

スクリプト末尾の `ads` 配列に広告を登録すると、`data-ad-slot` のプレースホルダーが広告表示に切り替わる。掲載可能枠は `p1-1` / `p6-1` / `p7-1` / `p9-1`。現在は `p9-1` にトーホー株式会社（グラスビーズ製造・広島）を掲載中。

## 落とし穴（過去に実際に起きた事故）

### 1. p2の3ボックスは本文が2箇所にある

2面の編集長コラム・編集部論説・今週の特集は、**同じ本文がHTMLとJSの2箇所に別々に存在する**。

- 表示用の生HTML（`<div class="op-box">` 内の `<div class="col-excerpt">`）
- `articles[]` 配列内の `col-chief-editor-001` / `editorial-001` / `feature-001`（ダイアログ用）

2026年7月27日号で、articles[]側だけを書き換えてHTML側を直し忘れる事故が実際に起きた。**新号を書いたら必ず両方を確認すること。** 検証コマンドは下記。

### 2. 記事の使い回し（JS配列内の本文は見落としやすい）

2026年7月20日〜7月23日号で、`market-001`（推し活市場）・`business-001`（作品を安く売りすぎる作家たち）・`culture-001`（一粒のビーズから見る世界の交易史）の3記事が、一字一句同じ本文のまま複数号にわたって使い回されていた。

原因は、旧来の重複チェックが「`<script>`を除外して`<p>`タグを比較する」方式だったため、**HTMLに直接書き出されず `articles[]` 配列からダイアログ表示されるだけの記事が検査対象から漏れていた**こと。

**レンダリング後HTMLの`<p>`比較に加えて、各号の`articles[]`をNodeでeval抽出し、同一idの`lead`/`body`が号をまたいで一致していないか機械的に突き合わせること。** 市場分析・作家論・文化コラムのような常設コーナーほど使い回しやすい。

### 3. 同じネタを複数面で使い回さない

2026年7月27日号で、その日一番の実ニュース（広島ハンドメイドフェスの夏冬2シーズン化）を1面リード・編集長コラム・論説・市場面・催事予告面と5箇所以上で角度を変えて使ってしまい、「同じ系統の記事ばかりにならないように」という指摘を受けた。

**強いネタが1つしかない日でも、1面リード＋2面のコラム/論説1〜2本程度に絞る。** 4〜9面の各ページ（ホビー・キルト、市場・企業、催事予告、催事暦、刺繍、書籍）には、それぞれジャンルの異なる別ネタを充てる。ネタ探しの段階で複数ジャンルの実ニュースを集めておくこと。

### 4. 裏の取れない数字を書かない

確認できない数字・日付は書かない。書く場合は「未確認」「不明」と明記する。推測で埋めることは絶対にしない。

## 発行前の検証

```bash
# JS構文チェック
python3 -c "
import re;s=open('issues/YYYY-MM-DD.html').read()
open('/tmp/c.js','w').write(re.search(r'<script>(.*)</script>',s,re.S).group(1))" && node --check /tmp/c.js

# p2の3ボックスがHTML側とarticles側で一致しているか
node -e "
const fs=require('fs');
const html=fs.readFileSync('issues/YYYY-MM-DD.html','utf8');
const arr=new Function('return '+html.match(/const articles = (\[[\s\S]*?\n  \];)/)[1].replace(/;\$/,''))();
const byId={}; arr.forEach(a=>byId[a.id]=a);
const norm=t=>t.replace(/<[^>]+>/g,'').replace(/\s+/g,'').replace(/[、。「」（）]/g,'');
[...html.matchAll(/<div class=\"col-excerpt\">([\s\S]*?)<\/div>\s*<button type=\"button\" class=\"read-more\" data-article-id=\"([^\"]+)\"/g)]
.forEach(([,inner,id])=>{
  const h=norm(inner), a=byId[id]; if(!a) return console.log('✗ 未定義',id);
  const t=norm(a.body.join(''));
  console.log((t.includes(h.slice(0,60))?'✓':'✗'), id, h.length, t.length);
});"

# 前号との記事重複チェック（articles配列の中身を突き合わせる）
node -e "
const fs=require('fs');
const load=f=>{const h=fs.readFileSync(f,'utf8');
  return new Function('return '+h.match(/const articles = (\[[\s\S]*?\n  \];)/)[1].replace(/;\$/,''))();};
const a=load('issues/前号.html'), b=load('issues/新号.html');
const m={}; a.forEach(x=>m[x.id]=(x.body||[]).join(''));
b.forEach(x=>{const p=m[x.id]; if(p && p===(x.body||[]).join('')) console.log('⚠ 本文が前号と同一:', x.id, x.title);});
console.log('重複チェック完了');"
```

タグバランスの確認も行うこと。

## ネタ帳

`notes/news-pool.md` にネタ候補を溜める。記入フォーマットはファイル冒頭に記載。使ったネタは「状態」を `使用済み（YYYY-MM-DD号）` に書き換える。曖昧な情報や出典不明の話は載せないこと。

## 主なネタ源

- [日本手芸普及協会／手づくりタウン](https://www.tezukuritown.com/)
- 繊研新聞（アパレル・繊維業界紙）／[FASHIONSNAP](https://www.fashionsnap.com/)
- PR TIMES（メーカー各社のプレスリリース）
- 各展示会・イベントの主催者公式サイト（長野キルトフェスティバル、TOKYOハンドメイド祭、デザインフェスタ、ビーズアートショー、広島ハンドメイドフェスティバル等）
- 矢野経済研究所（市場データ）／IMARC Group、Business Research Insights（世界市場推計）
- 日本ヴォーグ社・主婦の友社などの新刊情報

## 編集方針

- 事実報道（NEWS等）と、意見・分析を含む記事（EDITORIAL・CHIEF_EDITOR・FEATURE・MARKET・BUSINESS・CULTURE）を、バッジと記事冒頭の注記で明確に区別する
- 全文転載はしない。公開情報を要約し、編集部の解説・見立てを加える形にする
- 出典は各記事末尾に必ず明記する。複数サイトに転載されている場合は、できる限り元の発表元を特定する
- 発行ごとに、その時々で最も注目すべき話題を編集部が選定して紙面を構成する（曜日による固定割当はしない）
