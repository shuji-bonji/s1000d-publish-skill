# 下書き（`DraftDataModule`）の書き方

`build_data_module` の `draft` に渡す木。**書くのは題名・段落・手順・警告・表・図・参照だけ**で、DMC・ICN・BREX の参照・`qualityAssurance` などの識別・状態部は書かない（writer が決める）。XML も書かない。

## 要点

| 項目 | 規則 |
|---|---|
| 必須の項目 | `kind`、`language`、`title`（`techName` は必須、`infoName` は任意）、`securityClassification`、`body` |
| 識別 | `sourceId` を付ける。DMC を省くと、`sourceId` と `allocation` から採番される。**同じ `sourceId` には同じ DMC が付く**（台帳 `ledger` を渡すとき）。DMC を自分で指定したいときだけ `dmCode` を書く（pm は `pmCode`）。`sourceId`・`dmCode`・`pmCode` のどれも無いと入力の誤り |
| 機密区分 | `securityClassification` は**省略できない**。値は解釈されずそのまま書かれる。利用者に聞く。許可値はプロジェクトの BREX にある（brex の `lookup_value`、`target: "securityClassification"`） |
| 適用性 | `applic` を**省くと `applic` 要素が置かれず、XSD の検証で落ちる**（XSD は `applic` か `applicRef` を要求する）。最低限 `{ "displayText": "すべて" }` を書く。機械で判定したい条件は `assert`（複数なら `andOr`）で足す。値は利用者に聞く |
| 企業コード | `responsiblePartnerCompany`・`originator` を省くと `allocation.icn.cageCode` が入る |
| 発行日 | `issueDate`（`YYYY-MM-DD`）を省くと実行日。応答の `_notes` に書かれる |
| 版 | `issueInfo` を省くと、新規は `001-00`、既存の DM があれば `issuePolicy` に従う（既定は発行番号を上げる） |
| 参照 | **`text` の中に埋め込まず `refs` に書く。** `dmRef` / `pmRef`（`target` は DMC / PMC。`title` は参照先の題名）、`internalRef`（`target` は同じ DM の `id`。`title` は表示する文字）、`graphic` / `multimedia`（`figure` の `refs` にだけ置ける）。文の中の参照は、`text` の後ろに `refs` の順に並ぶ |
| 文字 | `text` の `<` と `&` はそのまま書く（XML の実体への変換は writer が行う） |
| 前提条件・事後条件 | proced・fault は、`step` の前後に `requirements` 節点を置く。各 `para` の `text` が 1 つの条件になる（`refs` は付けられない）。**`step` の中に「前提条件」という手順を書かない** |
| `id` | `internalRef` の参照先や、fault の `step`、process の `dialog` / `if` に要る。`id` は XML の id として使える文字（英数字と `-` `_`、先頭は英字）にする |
| 値の確認 | DMC・ICN・版・BREX の参照・列挙値は書かない。分からない値は利用者に聞く |

> [!IMPORTANT]
> 利用者に未確認の値（機密区分・適用性の表示文・企業コード）を、この Skill が埋めてはならない。聞いて、答えをそのまま書く。

## `input-error` の `path` の読み方

下書きや採番の規則に誤りがあると、`isError` ではなく次の形で返り、何も書かれない。

```json
{ "error": { "code": "input-error", "path": "/body[1]", "detail": "node kind \"section\" cannot be placed in the body of a \"proced\" data module (accepted: \"step\", \"requirements\", \"figure\", \"table\")" } }
```

| `path` | 指す場所 |
|---|---|
| `/body[2]` | `draft.body` の **2 番目**（1 始まり） |
| `/body[2]/children[1]` | その節点の `children` の 1 番目 |
| `/body[1]/rows[3]` | `table` の 3 行目 |
| `/body[1]/refs[1]` | その節点の `refs` の 1 番目 |
| `/allocation/rules/infoCodeByKind/proced` | `allocation` の中の項目（`draft` ではなく採番の規則の誤り。情報コードの欠落なら利用者に聞く） |
| `/kind` | `draft.kind` |

`detail` は、受け付ける種類や足りない項目を書いている。**`path` の箇所だけを直して呼び直す。** 他の箇所を作り直さない。

## 最小の例

### descript

```json
{
  "draft": {
    "sourceId": "overview",
    "kind": "descript",
    "language": { "languageIsoCode": "ja", "countryIsoCode": "JP" },
    "title": { "techName": "指定席券売機", "infoName": "概要" },
    "securityClassification": "01",
    "applic": { "displayText": "すべて" },
    "body": [
      { "kind": "para", "text": "指定席券売機は、きっぷの購入と予約の受け取りができる機械である。" },
      {
        "kind": "section",
        "title": "画面の構成",
        "children": [
          { "kind": "para", "text": "初期画面には 2 つのボタンがある。" },
          {
            "kind": "table",
            "title": "ボタンと役割",
            "rows": [["ボタン", "役割"], ["きっぷを買う", "新しくきっぷを買う"], ["予約を受け取る", "予約したきっぷを受け取る"]]
          }
        ]
      }
    ]
  },
  "allocation": {
    "modelIdentCode": "KENBAIKI",
    "systemDiffCode": "A",
    "rules": {
      "sns": { "systemCode": "10", "subSystemCode": "2", "subSubSystemCode": "0", "assyCode": "00" },
      "infoCodeByKind": { "descript": { "infoCode": "040", "infoCodeVariant": "A" } },
      "itemLocationCode": "D"
    },
    "icn": { "cageCode": "K0001", "next": 1 }
  },
  "csdbDirs": ["/path/to/csdb"]
}
```

### proced

```json
{
  "draft": {
    "sourceId": "open-door",
    "kind": "proced",
    "language": { "languageIsoCode": "ja", "countryIsoCode": "JP" },
    "title": { "techName": "指定席券売機", "infoName": "前面扉を開ける" },
    "securityClassification": "01",
    "applic": { "displayText": "すべて" },
    "body": [
      { "kind": "requirements", "children": [{ "kind": "para", "text": "電源が入っていること" }] },
      {
        "kind": "step",
        "title": "扉を開ける",
        "text": "保守鍵で前面扉を開ける。",
        "children": [
          { "kind": "admonition", "level": "caution", "children": [{ "kind": "para", "text": "手を挟まないように注意する。" }] },
          { "kind": "step", "text": "鍵を右に回す。" },
          { "kind": "step", "text": "取っ手を引く。" }
        ]
      },
      { "kind": "requirements", "children": [{ "kind": "para", "text": "扉が閉まっていること" }] }
    ]
  },
  "allocation": {
    "modelIdentCode": "KENBAIKI",
    "systemDiffCode": "A",
    "rules": {
      "sns": { "systemCode": "10", "subSystemCode": "2", "subSubSystemCode": "0", "assyCode": "00" },
      "infoCodeByKind": { "proced": { "infoCode": "520", "infoCodeVariant": "A" } },
      "itemLocationCode": "D"
    },
    "icn": { "cageCode": "K0001", "next": 1 }
  },
  "csdbDirs": ["/path/to/csdb"]
}
```

> [!WARNING]
> 上の例の `securityClassification`、`applic`、モデル識別コード、SNS、情報コード、CAGE コードは**試作用の値で意味を持たない**。実際の DM では、利用者に聞いた値、プロジェクトで登録したコードを使う。

`csdbDirs` は 1 件目が書き込み先。無いと参照が解決できず、書き込みもできない。

## 既存の DM を書き直すとき

reader の `read_data_module`（`path` か `dmc`）で読み、返った本文の木を同じ語彙の下書きに直す。読んだ木は書き戻すと同じ木になるように作られている。ただし reader が知らない構造は書けない。reader の応答に読み取れなかった要素（`unknown`）があれば、その部分は下書きに写せないので、利用者に伝える。
