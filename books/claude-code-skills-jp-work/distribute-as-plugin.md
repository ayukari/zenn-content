---
title: "プラグインとして配布する：marketplace とバージョン管理"
free: false
---

この章では、作ったスキルをプラグインにまとめ、GitHub から誰でもインストールできるようにします。題材は、この本で使っている `jp-backoffice-skills` の実際の構成です。

## プラグインにすると何がよいか

`.claude/skills/` にスキルを置くだけでも使えます。それでもプラグインにする理由は3つあります。

- **2行でインストールできる**：使う人はリポジトリをコピーする必要がありません。
- **更新を配れる**：スキルを直したら、使う人は `update` で最新版を受け取れます。
- **バージョンで管理できる**：「どの版で動いていたか」を記録できます。

## リポジトリの構成

`jp-backoffice-skills` の構成は次のとおりです。

```
jp-backoffice-skills/
├── .claude-plugin/
│   └── marketplace.json        # マーケットプレイス（プラグインの一覧）
├── plugins/
│   └── jp-backoffice-skills/
│       ├── .claude-plugin/
│       │   └── plugin.json     # プラグインの情報
│       └── skills/
│           ├── invoice-jp/
│           ├── keigo-email/
│           └── ...
├── evals/                      # 6章の evals
├── .github/workflows/test.yml  # 5章の CI
├── README.md
├── CHANGELOG.md
└── LICENSE
```

ポイントは、**1つのリポジトリが「マーケットプレイス」と「プラグイン」の両方を持っている**ことです。マーケットプレイスは「プラグインの一覧」で、プラグインは「スキルのまとまり」です。

## plugin.json

プラグインの情報です。

```json
{
  "name": "jp-backoffice-skills",
  "version": "0.1.0",
  "description": "インボイス対応請求書・敬語メール・稟議書・日報・経費帳など、日本の事務作業向けスキル集",
  "author": { "name": "ayukari" },
  "license": "MIT",
  "keywords": ["japanese", "invoice", "keigo", "backoffice", "agent-skills"]
}
```

`skills/` フォルダの中のスキルは、プラグインに含まれるものとして読み込まれます。スキルを1つずつ登録する必要はありません。

## marketplace.json

マーケットプレイスの情報と、含まれるプラグインの一覧です。

```json
{
  "name": "jp-backoffice",
  "description": "Japanese back-office Agent Skills: qualified invoices, keigo email, ringi, daily reports, tech writing",
  "owner": { "name": "ayukari" },
  "plugins": [
    {
      "name": "jp-backoffice-skills",
      "source": "./plugins/jp-backoffice-skills",
      "description": "インボイス対応請求書・敬語メール・稟議書・日報・経費帳など、日本の事務作業向けスキル集"
    }
  ]
}
```

マーケットプレイスの名前（`jp-backoffice`）と、プラグインの名前（`jp-backoffice-skills`）は別のものです。インストールするときは `プラグイン名@マーケットプレイス名` の形で指定します。

## 公開前に検証する

公開する前に、`claude plugin validate` で形式を確かめます。

```bash
claude plugin validate .
```

```
Validating marketplace manifest: .../.claude-plugin/marketplace.json

√ Validation passed
```

JSON の書き間違いやパスの誤りは、ここで見つかります。CI に入れておくと、壊れた状態で push することを防げます。

## 使う人のインストール手順

GitHub に push すれば、使う人は次の2行でインストールできます。

```bash
claude plugin marketplace add ayukari/jp-backoffice-skills
claude plugin install jp-backoffice-skills@jp-backoffice
```

1行目でマーケットプレイスを登録し、2行目でプラグインをインストールします。インストールの範囲は `--scope` で選べます（`user`・`project`・`local`）。チーム全員で使う場合の設定は、次の章で扱います。

更新とアンインストールは次のとおりです。

```bash
claude plugin marketplace update jp-backoffice   # マーケットプレイスの情報を更新
claude plugin update jp-backoffice-skills@jp-backoffice
claude plugin uninstall jp-backoffice-skills@jp-backoffice
```

README には、このインストール手順と「インストール後に何と頼めばよいか」の例を、最初のほうに書いておきます。

## バージョン管理

### バージョン番号の付け方

`plugin.json` の `version` は、次の考え方で上げています。

| 変更 | 例 | 上げる場所 |
|---|---|---|
| 修正（使い方は変わらない） | lint のルール追加、誤字修正 | 0.1.0 → 0.1.1 |
| 機能追加 | 新しいスキル、スクリプトの新しい入力項目 | 0.1.0 → 0.2.0 |
| 使い方が変わる変更 | JSON の形を変える、スキル名を変える | 0.x の間は 0.2.0 → 0.3.0、1.0 以降は 1.0.0 → 2.0.0 |

スキル名の変更は、使う人の手順やほかのスキルからの参照を壊します。気軽に変えないようにします。

### CHANGELOG を書く

変更内容は `CHANGELOG.md` に書きます。

```markdown
## [0.1.0] - 2026-10-05

### Added
- `invoice-jp`: qualified-invoice calculator (one rounding per tax rate), ...
- 18 evals (3 per skill) and GitHub Actions CI for script tests.

### Notes
- Bundled scripts are referenced as `${CLAUDE_SKILL_DIR}/scripts/...`
  so they work from any working directory.
```

「Notes」には、使う人に知っておいてほしい設計上の判断を書きます。4章で紹介した `${CLAUDE_SKILL_DIR}` の修正は、以前の版を手元で改造していた人に影響するため、ここに残しました。

### リリースを作る

バージョンを上げたら、GitHub のリリースを作ります。手順は次のとおりです。

1. `plugin.json` の `version` と `CHANGELOG.md` を更新して push する
2. GitHub のリポジトリ画面で「Releases」→「Draft a new release」を開く
3. タグ名（例：`v0.1.0`）を新しく入力し、CHANGELOG の該当部分を本文に貼る
4. 「Publish release」を押す（タグもこのとき作られます）

この本のスキル集でも、v0.1.0 はこの手順で公開しました。コマンドラインからタグを push できない環境でも、GitHub の画面からならリリースを作れます。

## 公開前のチェックリスト

- [ ] `claude plugin validate .` が通る
- [ ] スクリプトのテストが CI で通る
- [ ] evals の最新結果が `evals/results/` にある
- [ ] スクリプトのパスがすべて `${CLAUDE_SKILL_DIR}` になっている
- [ ] README の最初にインストール手順と使い方の例がある
- [ ] `plugin.json` の `version` と CHANGELOG がそろっている
- [ ] LICENSE がある
- [ ] 個人情報や社内情報がスキルや例文に残っていない

最後の項目は特に注意してください。業務用のスキルは、実際の書類をもとに作ることが多いはずです。例文の会社名・金額・登録番号は、架空のもの（`T1234567890123` のような明らかなダミー）に置き換えてから公開します。

## まとめ

- 1つのリポジトリに marketplace.json と plugin.json を置くと、2行でインストールできる
- インストールは `プラグイン名@マーケットプレイス名` で指定する
- 公開前に `claude plugin validate` で形式を確かめる
- version と CHANGELOG をそろえ、GitHub のリリースで版を残す
- 例文の個人情報・社内情報は公開前に必ず置き換える

次の章では、このしくみを使って、チームでスキル集を共有・運用する方法を説明します。
