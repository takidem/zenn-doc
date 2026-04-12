---
title: ""
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CTF", "PHP"]
published: true
---

## [writeup]Impossible Puzzle

### 対象の問題

[Impossible Puzzle](https://alpacahack.com/daily/challenges/impossible-puzzle)

### コード見てみる

docker-compose.yamlはいたってシンプルな内容でした。

```yaml
services:
  web:
    build: ./web
    restart: unless-stopped
    init: true
    ports:
      - ${PORT:-3000}:80
    environment:
      - FLAG=Alpaca{**** REDACTED ****}
```

Dockerfileも超シンプルです。

```yaml
FROM php:8.4.19-apache
COPY index.php /var/www/html/
```

index.phpが肝になりますが、変数AとBを比較して、長さは違うけど、内容が一緒であればフラグが出てきそうです。
PHPのいわゆる `緩やかな比較` を利用したら、 `0` と `00` が一緒になるので、後段の条件文意に進めると思います。

```php
<?php

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $A = $_POST['A'] ?? '';
    $B = $_POST['B'] ?? '';

    // type check
    if (!is_string($A) || !is_string($B)) {
        die('Invalid input');
    }
    // check
    if (strlen($A) != strlen($B) && $A == $B){
        echo "<blockquote>Congratulations! Here is your flag:\n" . getenv('FLAG') . "</blockquote>";
    }
    else {
        echo "<blockquote>Try again!</blockquote>";
    }
}

?>

<!DOCTYPE html>
<html>

<head>
    <meta charset="utf-8" />
    <title>Impossible Puzzle</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" href="https://unpkg.com/sakura.css/css/sakura.css" type="text/css" />
    <script src="https://cdn.jsdelivr.net/npm/dompurify@3.3.1/dist/purify.min.js"></script>
</head>

<body>
    <h1>Impossible Puzzle</h1>
    <p>Enter two strings A and B. If they are equal in content but different in length, you will get the flag!</p>
    <form method="post">
        <label for="A">A:</label>
        <input type="text" id="A" name="A" required>
        <label for="B">B:</label>
        <input type="text" id="B" name="B" required>
        <button type="submit">Submit</button>
    </form>
</body>
</html>
```

### docker compose upして確認

```bash
docker compose up -d
```

こんなWeb画面になります。

![Impossible Puzzle 001](/images/2bd0e29bcdae1b/001.png)

この入力欄で `0` と `00` を入れてみます。いけそうです。

![Impossible Puzzle 002](/images/2bd0e29bcdae1b/002.png)

alpacahack上で仮想環境を立ち上げて、接続して同じことを実行します。

![Impossible Puzzle 003](/images/2bd0e29bcdae1b/003.png)
