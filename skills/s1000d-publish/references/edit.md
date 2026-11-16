# CSDB にある DM の構造を変える（`edit_data_module`）

CSDB にある手順・説明のデータモジュール（DM）に、節点を足す・取り除く・移す編集の手順。SKILL.md の「CSDB にある DM を直したいと頼まれたとき」から来る。

s1000d-writer-mcp の `edit_data_module`（0.5.1 以降）を使う。このツールは、元の XML の編集した範囲だけを書き換え、ほかは 1 文字も変えない。DM を reader で読んで下書きから作り直すことはしない。

## できること・できないこと

| 区分 | 内容 |
|---|---|
| 対象の DM | 手順（`proced`）と説明（`descript`）。プロセス・故障探求の DM には節点を足せない |
| 取り除ける・移せる要素 | `proceduralStep`、`para`、`warning`、`caution`、`note`、`listItem` |
| 足せる節点（下書きの `kind`） | `step`、`para`、`admonition`（`level` が要る）、`listItem` |
| できない | 表・図・前提条件の品目を足す・取り除く・移すこと。段落の文を書き換えること。節点を別の内容に置き換えること |
| **変えないもの** | **版（`issueInfo`）と、変更の印（`changeType`・`changeMark`・`reasonForUpdate`）** |

できないことを頼まれたら、できないと伝える。取り除いて足し直せば似た結果になる場合でも、利用者に確かめずに置き換えない（取り除いた要素の属性や `id` は戻らない）。

## 手順

### E1 — 何を変えるかと、版を確かめる

1. どの DM の、どこに、何をするのかを利用者に確かめる
2. reader の `read_data_module` を `includeLocation: true` で呼んで DM を読み、`issueInfo` の `inWork` を見る（同じ応答を E2 で使う）

| `inWork` | 行うこと |
|---|---|
| `00` 以外（作業中の版） | そのまま E2 へ進む |
| `00`（発行済みの版） | **そのままでは編集できない**（ツールが `released-issue` で断る）。下の説明を利用者に伝え、どうするかを聞く |

発行済みの版について伝えること:

- このツールは、版を上げず、変更の印も付けない
- S1000D では、発行済みの DM を直すときは作業中の版に進め、技術的な変更には変更の印と更新の理由を付ける。その操作は、このツールにも writer のほかのツールにも、まだ無い
- 試作や練習で、版を変えずに書き換えてよい場合だけ、`allowReleasedIssue: true` で編集できる

**`allowReleasedIssue: true` は、利用者が「版を変えずに書き換えてよい」と明示したときだけ渡す。** 自分の判断で付けない。付けたときは、報告に必ず書く。

### E2 — 場所を得る

編集の場所は、reader の `read_data_module` を **`includeLocation: true`** で呼んだ応答にある、各節点の `location.path` で指す（例: `/dmodule/content/procedure/mainProcedure/proceduralStep[2]`）。`includeLocation` を渡さないと、応答に `location` は無い（reader 0.12.0 以降）。

- path を自分で組み立てない。応答にある値を写す
- 親の節点の `text` として返る段落（手順の最初の段落など）は、節点が無いので `location` も無い。その段落を取り除く・移す依頼は、path を得られないので、できないと伝える。その段落の前後に足すときは、親の節点を基準にして `first-child`・`last-child` を使うか、隣の節点を基準にする
- reader が未接続（または 0.12.0 より前）で、利用者も path を示せないときは、編集しない。reader の接続・更新の方法を伝える

### E3 — 編集を組む

`edits[]` に、編集を 1 つずつ書く。

| `op` | 書く項目 |
|---|---|
| `insert` | `to`（基準の節点の path）、`position`、`node`（足す節点。下書きの節点 1 つ） |
| `remove` | `at`（取り除く節点の path） |
| `move` | `at`（移す節点の path）、`to`（基準の節点の path）、`position` |

`position` は `before`・`after`（基準の節点の前後）か、`first-child`・`last-child`（基準の節点の中）。

決まり:

- **複数の編集を渡すときも、path はすべて編集の前の文書のもの**を書く。前の編集でずれる番号を自分で計算しない
- **並びの順は XSD が決めている。** `proceduralStep` の中は、題名 → 警告・注意 → 段落・図・表など → 入れ子の手順、の順。警告を手順に足すなら `first-child`（題名があるときは、題名の `after`）。入れ子の手順は `last-child`
- `node` に `id` を書くのは、利用者が `id` を示したときだけ。`id` を作らない。文書に既にある `id` は断られる（`id-duplicate`）
- `node` の `refs` の `internalRef` は、文書に既にある `id` だけを指せる
- XML を書かない。`node` は下書きの節点（[draft.md](draft.md)）

### E4 — 書かずに呼ぶ

`edit_data_module` を、`path`（DM のファイル）、`csdbDirs`、`edits` を付けて、`write` なしで呼ぶ。応答の次の項目を、この順に読む。

| 項目 | 読み方 |
|---|---|
| `applied` | `false` なら、1 つも適用されていない（全部か無しか）。`refusals[]` を読む |
| `refusals[]` | 断られた編集ごとの `reason`・`message`・`location`。下の表で対処する |
| `newSchemaErrors[]` | 編集で増えた XSD のエラー。**1 つでもあれば、その結果は書き込めない。** `message` を読み、位置か編集を変えて、**元の文書に対して**呼び直す。`apply_fixes` には渡さない |
| `referencedFromOtherObjects[]` | 取り除いた `id` を、CSDB のほかの DM・PM の `dmRef` が指している場所。**1 つでもあれば書き込めない。** 利用者に伝え、指している側を先に直すか、編集をやめるかを聞く |
| `verification` | 編集の後の検証状態。Phase 5 と同じ読み方 |
| `fixes[]` | 編集の前からあった問題への候補。この手順では扱わない（必要なら、書き込んだ後に Phase 6 の `apply_fixes`） |
| `changes[]` | 適用した編集。報告に使う |

`refusals[].reason` の対処:

| `reason` | 意味 | 対処 |
|---|---|---|
| `released-issue` | 発行済みの版 | E1 に戻り、利用者に聞く |
| `location-not-found`・`location-ambiguous` | path が節点を 1 つに決めない | reader の応答の path を写し直す |
| `unsupported-node` | 対象でない要素 | できないと伝える |
| `unsupported-destination` | 足せない親、または DM の種類 | できないと伝える |
| `not-allowed-here` | XSD で、その親がその要素を子に取れない | 別の場所を利用者と決める |
| `id-referenced` | 取り除く節点の `id` を、同じ DM のほかの場所が指している | `referencedBy[]` の場所を利用者に示し、どうするかを聞く。指している側を黙って取り除かない |
| `id-duplicate` | 足す節点の `id` が既にある | 別の `id` を利用者に聞く |
| `draft-input-error` | `node` の形が受け付けられない | `message` の箇所を直す |
| `move-into-self`・`conflicting-edits` | 編集どうしが食い違う | 編集を組み直す |
| `internal-mismatch`・`ranges-unavailable` | ツールが安全に書き換えられないと判断した | 編集しない。そのまま利用者に伝える |

### E5 — 利用者に示して、確認を得る

`applied` が `true` で、`newSchemaErrors[]` と `referencedFromOtherObjects[]` が空になったら、次を示す。

- 変わる場所と内容（`changes[]`。足した節点は、応答の `xml` のその部分）
- 検証状態（`verification`）
- **版と変更の印は変えていないこと。** `allowReleasedIssue: true` を渡したなら、発行済みの版を書き換えること
- 書き込むと、`path` のファイルが上書きされること

### E6 — 書き込んで、確かめる

1. 利用者の確認の後に、同じ `edits` を `write: true` で呼ぶ。`written[]` にファイルのパスが入る。`writeSkipped` が返ったら、`reason` と `detail` をそのまま伝える
2. verify の `verify_data_module` で、書いたファイルを検証する
3. 報告する。[report.md](report.md) の雛形のうち、該当する項目だけを使い、「材料と目的」に「CSDB にある DM の構造の編集」と書く。次を必ず入れる
   - 対象の DM（DMC、版、ファイル）
   - 適用した編集（`changes[]`）
   - 版と変更の印を変えていないこと。`allowReleasedIssue: true` を使ったかどうか
   - 検証状態

## やらないこと

- 利用者が明示していないのに `allowReleasedIssue: true` を渡すこと
- `newSchemaErrors[]` がある結果を、`apply_fixes` や `allowInvalid` で通そうとすること
- path や `id` を自分で作ること
- 編集できないときに、DM を reader で読んで下書きから作り直すこと
- 「版を上げた」「変更の印を付けた」と報告すること（このツールは、どちらも行わない）
