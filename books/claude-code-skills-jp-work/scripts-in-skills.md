---
title: "スクリプトを組み込む：計算と点検をLLM任せにしない"
free: false
---

この章では、スキルにスクリプトを同梱して、計算や点検を確実に行う方法を説明します。題材は `invoice-jp` の消費税計算と、`keigo-email` の敬語チェックです。

## どこをスクリプトにするか

すべてをスクリプトにする必要はありません。このスキル集では、次の基準で分けています。

| LLM に任せる | スクリプトに任せる |
|---|---|
| 文章の組み立て、言い回し、構成 | 金額・税額の計算 |
| 文脈に応じた判断（相手との関係など） | 決まった規則による判定（必須項目の有無など） |
| ユーザーへの質問、説明 | 毎回同じ結果でなければ困る処理 |

目安は「**答えが1つに決まり、間違えると困るもの**」です。消費税の計算はまさにこれに当てはまります。LLM は四則演算をだいたい正しく行いますが、「税率ごとに1回だけ端数処理する」といった規則を、文脈によって守ったり守らなかったりします。

## 例1：消費税の計算（invoice_calc.py）

### 入力を JSON に決める

スクリプトと LLM の間は、**JSON でやりとりする**と安定します。Claude には「明細を JSON にしてスクリプトに渡す」とだけ指示し、JSON の形はスクリプトの冒頭に書いておきます。

```json
{
  "issuer": {"name": "山田デザイン", "registration_number": "T1234567890123"},
  "recipient": "株式会社サンプル",
  "date": "2026-10-04",
  "transaction_date": "2026-10-01",
  "price_mode": "exclusive",
  "rounding": "floor",
  "items": [{"name": "Webデザイン一式", "qty": 1, "unit_price": 100000, "rate": 10}]
}
```

### 計算の中心部分

実際のコードの中心部分です。

```python
def calculate(inv):
    mode = inv.get("price_mode", "exclusive")
    rounding = ROUNDING[inv.get("rounding", "floor")]
    totals = {}
    for item in inv["items"]:
        amount = Decimal(str(item["unit_price"])) * Decimal(str(item.get("qty", 1)))
        totals[item["rate"]] = totals.get(item["rate"], Decimal(0)) + amount

    summary = []
    for rate in sorted(totals):
        r = Decimal(rate) / 100
        base = totals[rate]
        if mode == "exclusive":
            tax = (base * r).quantize(Decimal(1), rounding=rounding)
            excl, incl = base, base + tax
        else:
            tax = (base * r / (1 + r)).quantize(Decimal(1), rounding=rounding)
            excl, incl = base - tax, base
        ...
```

ポイントは3つあります。

1. **先に税率ごとに合計し、そのあとで1回だけ端数処理します**。明細ごとに税額を出してから足すことはしません。
2. **`Decimal` を使います**。`float` で 0.1 などを扱うと誤差が出るため、金額計算には使いません。
3. **端数処理の方法（切り捨て・四捨五入・切り上げ）を入力で選べます**。どれを使うかは事業者が決めることなので、スクリプトでは決めつけません。

### 入力チェックも同じスクリプトで

計算の前に、必須項目の検証も行います。

```python
REG_NO = re.compile(r"^T\d{13}$")

def validate(inv):
    errors = []
    if not REG_NO.match(inv.get("issuer", {}).get("registration_number", "")):
        errors.append("issuer.registration_number must be 'T' + 13 digits (登録番号)")
    ...
```

エラーがあれば `{"ok": false, "errors": [...]}` を返して終了します。SKILL.md には「`ok: false` が返ったら、`errors` の内容をユーザーに伝えて直す」と書いておきます。これで、登録番号のない請求書を Claude が勝手に作ることはなくなります。

### テストで規則を固定する

スクリプトの価値は「毎回同じ結果になる」ことです。それを保証するのがテストです。

```python
def test_per_rate_differs_from_per_line(self):
    # 3 lines of 105 yen at 10%.
    # Per line (not allowed): floor(10.5) = 10, x3 = 30 yen.
    # Per rate (required):     floor(315 x 0.1) = floor(31.5) = 31 yen.
    inv = {**BASE, "items": [
        {"name": "部品A", "qty": 1, "unit_price": 105, "rate": 10},
        {"name": "部品B", "qty": 1, "unit_price": 105, "rate": 10},
        {"name": "部品C", "qty": 1, "unit_price": 105, "rate": 10},
    ]}
    self.assertEqual(calculate(inv)["by_rate"][0]["tax"], 31)
```

テストには、**間違った方法だと結果が変わる例**を選びます。105円の明細が3つある場合、明細ごとに切り捨てると30円、税率ごとにまとめて切り捨てると31円になります。

実は最初に書いたテストは「333円×3」という例でした。これは明細ごとでも税率ごとでも99円になり、規則の違いを検出できないテストでした。この章を書くために見直して気づき、上のテストを追加しました。テストは「通る」だけでなく、「間違った実装なら落ちる」ことを確認してください。

## 例2：敬語チェック（keigo_lint.py）

2つめの例は、文章のチェックです。こちらは「答えが1つ」ではありません。それでも、**定番の誤りを漏れなく拾う**作業は、スクリプトが確実です。

### ルールはデータとして分ける

チェックのルールはコードに直接書かず、JSON ファイルに分けています。

```json
{"id": "ossharareru", "pattern": "おっしゃられ", "suggest": "おっしゃる",
 "why": "二重敬語（おっしゃる＋られる）", "level": "error"}
```

こうしておくと、プログラムを書かない人でもルールを追加できます。チームで「うちでよく見る誤り」を育てていけるのが利点です。

### 判定は3段階にする

`level` は `error`・`warn`・`info` の3段階です。

- `error`：文脈によらず誤り（二重敬語、相手の行為への謙譲語）
- `warn`：相手によっては失礼（「了解しました」）
- `info`：好みの問題（「させていただく」の多用）

SKILL.md には「`error` は必ず直す。`warn` と `info` は相手との関係を踏まえて判断する」と書きます。**スクリプトが拾い、LLM が判断する**という役割分担です。

## SKILL.md からの呼び出し方

4章で説明したとおり、スクリプトのパスは `${CLAUDE_SKILL_DIR}` で書きます。

````markdown
2. 明細をJSONにして、次のスクリプトに渡す。
   ```bash
   python3 ${CLAUDE_SKILL_DIR}/scripts/invoice_calc.py invoice.json
   ```
   `ok: false` が返ったら、`errors` の内容をユーザーに伝えて直す。
````

呼び出し方と一緒に、**結果をどう扱うか**まで書くことが大切です。「エラーが出たらどうするか」が書いていないと、Claude はエラーを無視して自分で計算し直すことがあります。

## スクリプトを書くときの注意

- **標準ライブラリだけで書く**。使う人の環境に追加パッケージがあるとは限りません。このスキル集のスクリプトは Python の標準ライブラリだけで動きます。
- **入出力は JSON か、プレーンテキスト**。Claude が読み書きしやすい形にします。
- **エラーメッセージに日本語の項目名を入れる**。`registration_number (登録番号)` のように書いておくと、Claude がそのままユーザーに説明できます。
- **終了コードを使い分ける**。問題があるときは 0 以外で終了させると、Claude が失敗を見落としにくくなります。

## CI でテストを自動実行する

テストは、push のたびに自動で実行するようにしておきます。このスキル集では、GitHub Actions で次のように実行しています。

```yaml
- name: Run skill script tests
  run: |
    set -e
    for d in $(find plugins -type d -name scripts); do
      (cd "$d" && python -m unittest -v)
    done
```

スキルごとの `scripts` フォルダを探して、すべてのテストを実行するだけの設定です。スキルを追加しても設定を変える必要はありません。

## まとめ

- 答えが1つに決まり、間違えると困る処理はスクリプトにする
- LLM とスクリプトの間は JSON でやりとりする
- ルールはデータとして分け、チームで育てられるようにする
- 「スクリプトが拾い、LLM が判断する」役割分担にする
- テストは「間違った実装なら落ちる」例で書く
- SKILL.md には、呼び出し方と結果の扱い方をセットで書く
- テストで規則を固定し、CI で自動実行する

次の章では、スキル全体の品質を evals で測る方法を説明します。スクリプトのテストだけでは見つからない問題を、実際にどう見つけて直したかを紹介します。
