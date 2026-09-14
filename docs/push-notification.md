# 新号のプッシュ通知（設計と手順）

新しい号を発行したら、TOHO BEADS アプリポータル（https://app.tohobeads.jp/ ）の
利用者へWeb Pushで通知する仕組み。

## 現状（2026年9月14日時点）

| 要素 | 状態 |
|---|---|
| 受信側（`sw.js` の push ハンドラ、未読バッジ、タップ遷移） | **実装済み** |
| PWA manifest（standalone、アイコン192/512） | **実装済み** |
| 購読処理（`pushManager` / `applicationServerKey`） | **フロント側に実装済み** |
| `/api/subscribe`・`/api/announcements`・`/api/events` | **フロントは呼んでいるがサーバー側が404** |
| 新号の検知とペイロード送出（GitHub Actions） | **本リポジトリに実装済み**（`.github/workflows/notify-new-issue.yml`） |
| 購読情報の保存と Web Push 送信（サーバー） | **未実装** ← 残っているのはここだけ |

## 全体の流れ

```
新号を push（issues/YYYY-MM-DD.html を追加）
   ↓ GitHub Actions が「追加された号」だけを検知
   ↓ <title> と og:description からペイロードを組む
   ↓ POST → PUSH_NOTIFY_ENDPOINT
   ↓ サーバーが購読者全員へ Web Push を送信
   ↓ app.tohobeads.jp の sw.js が受信 → 通知表示＋バッジ+1
   ↓ 通知をタップ → 新号のページが開く
```

## GitHub Actions が送るペイロード

`sw.js` の `push` ハンドラが `{title, body, url}` を読む実装になっているため、それに合わせてある。

```json
{
  "title": "手芸新聞 — 2026年8月16日号（試作第8号）",
  "body": "【一面トップ】ユネスコ本部（パリ）で9月8〜11日に開かれた…",
  "url": "https://djtobby.github.io/shugei-shimbun/issues/2026-09-14.html",
  "tag": "shugei-2026-08-16",
  "paper": "shugei-shimbun",
  "issue": "2026-09-14"
}
```

- `tag` は同じ号の通知が二重に出ないようにするためのもの。`showNotification` の `tag` に渡すとよい
- `paper` は手芸新聞と生涯学習新聞を区別するためのもの。購読者が紙を選べるようにする場合に使う

## 必要な設定（山仲さんの作業）

### 1. VAPID鍵ペアを生成する

Web Push には送信元を証明する鍵が要る。ローカルで一度だけ生成する。

```bash
npx web-push generate-vapid-keys
```

出力される Public Key / Private Key を控える。

- **Public Key** … app.tohobeads.jp のフロント（`applicationServerKey`）とサーバーの両方で使う
- **Private Key** … サーバーだけが持つ。**絶対に公開リポジトリに置かない**

### 2. GitHub Secrets に登録する

各リポジトリの Settings → Secrets and variables → Actions → New repository secret

| 名前 | 値 |
|---|---|
| `PUSH_NOTIFY_ENDPOINT` | `https://app.tohobeads.jp/api/notify`（実装後のURL） |
| `PUSH_NOTIFY_TOKEN` | 任意の長い文字列。サーバー側と一致させる（誰でも通知を送れてしまうのを防ぐ） |

**この登録は私（Claude）にはできません。** シークレットの入力は山仲さんご自身でお願いします。
未設定のままでもワークフローはエラーにならず、警告を出してスキップします。

### 3. サーバー側APIを実装する

app.tohobeads.jp は LiteSpeed で動いているため、PHPが使える想定。必要なのは2本。

#### `/api/subscribe`（POST）— 購読の登録

フロントが `pushManager.subscribe()` で得た購読情報を保存する。

```php
<?php
// api/subscribe.php
header('Content-Type: application/json');
$raw = file_get_contents('php://input');
$sub = json_decode($raw, true);
if (!$sub || empty($sub['endpoint'])) {
    http_response_code(400);
    exit(json_encode(['error' => 'invalid subscription']));
}
// endpoint をキーにして重複を防ぐ（DBがあればテーブルに入れる）
$dir = __DIR__ . '/../data/subs';
@mkdir($dir, 0700, true);
file_put_contents($dir . '/' . hash('sha256', $sub['endpoint']) . '.json', $raw);
echo json_encode(['ok' => true]);
```

> 保存先はWebから読めない場所に置くこと。上の例の `data/` は `.htaccess` で拒否するか、
> ドキュメントルートの外に置く。

#### `/api/notify`（POST）— 通知の送信

GitHub Actions から呼ばれ、保存済みの購読者全員へ送る。

```php
<?php
// api/notify.php   事前に: composer require minishlink/web-push
require __DIR__ . '/../vendor/autoload.php';
use Minishlink\WebPush\WebPush;
use Minishlink\WebPush\Subscription;

// --- 認証（GitHub Actions 以外から叩かれないように） ---
$expected = getenv('PUSH_NOTIFY_TOKEN');
$given = $_SERVER['HTTP_AUTHORIZATION'] ?? '';
if (!$expected || $given !== "Bearer $expected") {
    http_response_code(401); exit(json_encode(['error' => 'unauthorized']));
}

$payload = file_get_contents('php://input');   // {title, body, url, tag, ...} をそのまま転送

$webPush = new WebPush(['VAPID' => [
    'subject'    => 'https://app.tohobeads.jp/',
    'publicKey'  => getenv('VAPID_PUBLIC_KEY'),
    'privateKey' => getenv('VAPID_PRIVATE_KEY'),
]]);

$sent = 0; $gone = 0;
foreach (glob(__DIR__ . '/../data/subs/*.json') as $f) {
    $sub = Subscription::create(json_decode(file_get_contents($f), true));
    $webPush->queueNotification($sub, $payload);
    $sent++;
}
foreach ($webPush->flush() as $report) {
    // 410/404 が返る購読は失効しているので削除する（放置すると毎回失敗し続ける）
    if (!$report->isSuccess() && $report->isSubscriptionExpired()) {
        $h = hash('sha256', $report->getRequest()->getUri()->__toString());
        @unlink(__DIR__ . "/../data/subs/$h.json");
        $gone++;
    }
}
echo json_encode(['ok' => true, 'sent' => $sent, 'expired_removed' => $gone]);
```

VAPID鍵とトークンは `.env` や サーバーの環境変数に置き、**ソースには直接書かない**。

## テスト手順

1. GitHub の Actions タブ → 「新号のプッシュ通知」→ Run workflow
2. `dry_run` を `true` のままにすると、送信せずペイロードだけ表示される
3. ペイロードが正しければ `false` にして実行し、実機に通知が届くか確認

## 注意点

- **iOS**: ホーム画面に追加したPWAでのみWeb Pushが動く。Safariのタブで開いただけでは通知は届かない
- **失効した購読**: ブラウザのデータ削除やアプリ再インストールで購読は無効になる。410/404が返ったら
  サーバー側で削除する（上の実装に含めてある）
- **重複通知**: 同じ号のワークフローが二重に走っても、`tag` が同じなら通知は上書きされて1件に見える
- ワークフローは `issues/*.html` が**追加**されたときだけ発火する。既存号を修正しても通知は飛ばない
