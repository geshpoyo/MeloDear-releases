# api/ — MeloDear のバージョン・キルスイッチ

`version-policy.json` は、**深刻な不具合を含むバージョンが利用者の手元で動き続けるのを止める**ための配信ファイルです。
MeloDear は起動時にこの URL を読みます。

```
https://raw.githubusercontent.com/geshpoyo/MeloDear-releases/main/api/version-policy.json
```

> **`rules` を空にしたまま置いておくこと。** ファイルが存在しない（404）と、
> アプリは「制限なし」として起動し続けます。**止めたくても止められない**状態になります。

---

## 平常時（今の状態）

```json
{
  "schemaVersion": 1,
  "updatedAt": "2026-09-06T00:00:00Z",
  "signature": "",
  "rules": []
}
```

`rules` が空なら、アプリは**何も表示せず**通常どおり起動します。

---

## 止めたいとき

`rules` に 1 件足して push するだけです。反映は各利用者の次回起動時です。

```json
{
  "schemaVersion": 1,
  "updatedAt": "2026-09-06T12:00:00Z",
  "signature": "",
  "rules": [
    {
      "id": "block-0.2.2-19-library-corruption",
      "match": { "versions": ["0.2.2+19"], "below": "" },
      "action": "block",
      "allowReadOnly": true,
      "message": "このバージョンには、タグ書き戻しでライブラリが壊れる不具合があります。次の版が出るまで閲覧のみでお使いください。",
      "moreInfoUrl": "https://melodear.geshtalt.jp/known-issues/"
    }
  ]
}
```

### フィールド

| キー | 意味 |
|---|---|
| `id` | 任意の識別子。人が読むためのもので、動作には影響しない |
| `match.versions` | 対象バージョンの配列。**`"0.2.2+19"` は完全一致**、**`"0.2.2"` は `+N` を問わずその `X.Y.Z` の全ビルド** |
| `match.below` | 非空なら「`current` < `below`」で一致。`"0.3.0"` と書けば 0.3.0 未満が全部対象 |
| `action` | `"block"` または `"warn"` |
| `allowReadOnly` | `action` が `block` のときのみ意味を持つ。下表参照 |
| `message` | 利用者に見せる文言。**なぜ止めるのかを書く** |
| `moreInfoUrl` | ［詳細を開く］で開く URL |

### `action` × `allowReadOnly`

| 組み合わせ | 利用者に起きること |
|---|---|
| `warn` | 起動を妨げない。通知バーに `message` を出すだけ |
| `block` ＋ `allowReadOnly: true` | ダイアログ。［閲覧のみで続行］で**再生と閲覧はできる**が、書き込み系（解析・タグ編集・取込み・MCP・OpenSubsonic の書き込み）が全部止まる |
| `block` ＋ `allowReadOnly: false` | ダイアログに続行ボタンが出ない。［終了］でアプリが終わる。**最終手段** |

### 覚えておくこと

- **先頭のルールから評価し、最初に一致した 1 件が勝つ。** 順序に意味があります
- `versions` と `below` が両方非空のときは **OR**（どちらかに一致すれば一致）
- **知らない `action` は無視して次のルールへ進む。** 将来フィールドを増やしても古いクライアントが壊れません
- `signature` は**現在は検証していません**。フィールドがあるからといって真正性の保証にはなりません
- バージョン比較は MeloDear 独自の規則です。`X.Y.Z` を数値比較し、`+N` は `X.Y.Z` が同じときだけ数値比較します（SemVer は build metadata を順序に使わないので、そこが違います）

### 解除するとき

`rules` を `[]` に戻して push します。次回起動で制限が解けます。

---

## 通信の性質

- 送信するのは**実行中のバージョン番号だけ**です。ライブラリの内容や利用状況は送りません
- タイムアウトは 10 秒。**取得に失敗しても起動は妨げません**（fail-open）
- 一度取得した内容は `%APPDATA%\MeloDear\version-policy-cache.json` にキャッシュされ、
  **オフラインでもブロックは効き続けます**
- キャッシュも無く取得にも失敗した場合は「制限なし」として起動します

## 変更するときの手順

1. このファイルの `rules` を編集する
2. `updatedAt` を現在時刻（UTC・ISO 8601）に更新する
3. `python -m json.tool api/version-policy.json` で JSON として妥当か確かめる
4. commit して `main` に push する

**JSON が壊れていると、アプリはキャッシュを使うか制限なしで起動します**（壊れたファイルで利用者を止めることはありません）。
