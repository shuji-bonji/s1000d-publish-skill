# CSDB にある DM の構造を変える（`edit_data_module`）

CSDB にある手順・説明のデータモジュール（DM）に、節点を足す・取り除く・移す編集の手順。SKILL.md の「CSDB にある DM を直したいと頼まれたとき」から来る。

s1000d-writer-mcp の次の 3 つのツールを使う（0.6.0 以降）。どれも、元の XML の書き換える範囲だけを書き換え、ほかは 1 文字も変えない。DM を reader で読んで下書きから作り直すことはしない。

| ツール | 行うこと |
|---|---|
| `edit_data_module` | 節点を足す・取り除く・移す。`marking` を渡すと、変更の印を付ける |
| `start_revision` | 発行済みの版から、作業中の版を新しいファイルとして作る |
| `add_reason_for_update` | 更新の理由（`reasonForUpdate`）を 1 つ足す。本文には触らず、変更の印も付けない |

writer が 0.5.x のときは `edit_data_module` だけがあり、`marking` も無い。その場合は、作業中の版を始めることも、変更の印を付けることもできないと伝える。

## できること・できないこと

| 区分 | 内容 |
|---|---|
| 対象の DM | 手順（`proced`）と説明（`descript`）。プロセス・故障探求の DM には節点を足せない |
| 取り除ける・移せる要素 | `proceduralStep`、`para`、`warning`、`caution`、`note`、`listItem` |
| 足せる節点（下書きの `kind`） | `step`、`para`、`admonition`（`level` が要る）、`listItem` |
| できない | 表・図・前提条件の品目を足す・取り除く・移すこと。段落の文を書き換えること。`marking` を渡さずに、節点を別の内容に置き換えること（`marking` を渡すときは、remove と insert の組で書ける。E3） |
| `marking` を渡さない `edit_data_module` が変えないもの | 版（`issueInfo`）と、変更の印（`changeType`・`changeMark`・`reasonForUpdate`） |
| この手順で行わないこと | 発行（`inWork` を `00` に戻し、発行番号を上げる）、発行日・品質保証の状態の更新。文の一部への変更の印（`changeInline`）。表・図への変更の印 |

できないことを頼まれたら、できないと伝える。取り除いて足し直せば似た結果になる場合でも、利用者に確かめずに置き換えない（取り除いた要素の属性や `id` は戻らない）。

## 手順

### E1 — 何を変えるかと、版を確かめる

1. どの DM の、どこに、何をするのかを利用者に確かめる
2. reader の `read_data_module` を `includeLocation: true` で呼んで DM を読み、`issueInfo` の `inWork` と、`identAndStatus.issueType` を見る（同じ応答を E2 で使う。`section` は既定の `all` にする。`content` では識別・状態部が返らず、`outline` では `textLocations` が付かない）
3. `issueType` が `deleted`（削除済みの版）なら、作業中の版は始められない（`start_revision` が `deleted-issue` で断る）。版の扱いや `issueType` を聞く前に、そのことを利用者に伝える。下の選択肢のうち、「作業中の版を始めて、S1000D の決まりに沿って直す」は示さない

| `inWork` | 行うこと |
|---|---|
| `00` 以外（作業中の版） | 下の「変更の印を付けるかを確かめる」へ進む |
| `00`（発行済みの版） | **そのままでは編集できない**（`edit_data_module` が `released-issue` で断る）。下の 3 つから、利用者に選んでもらう |

発行済みの版のときに、利用者に示す選択肢:

| 選択肢 | 行うこと |
|---|---|
| 作業中の版を始めて、S1000D の決まりに沿って直す | E1a へ進む |
| 版を変えずに、このファイルを書き換える（試作・練習） | `allowReleasedIssue: true` で編集する。変更の印は付けられない。結果は、発行済みの DM を直すときの S1000D の決まりに沿わない |
| やめる | 編集しない |

**どれにするかは利用者が決める。** 自分の判断で `start_revision` を呼ばない。`allowReleasedIssue: true` も、利用者が「版を変えずに書き換えてよい」と明示したときだけ渡し、渡したときは報告に必ず書く。

### E1a — 作業中の版を始める（`start_revision`）

1. 作業中の版の `issueType` を、利用者に聞く。**自分で選ばない。**

   | `issueType` | 意味 | この後の編集 |
   |---|---|---|
   | `changed` | 変更があり、変更の印で示す | `edit_data_module` に `marking` を渡す |
   | `revised` | 全体を作り直した。変更の印は持たない | `marking` は渡せない（`issue-type-not-markable` で断られる） |
   | `status` | 識別・状態部だけを更新する | 本文を編集するかどうかを、利用者に確かめる（`marking` は渡せる） |
   | `new` | 発行番号が `001` の版から始めるときだけ選べる | 応答の `_notes` に、既定 BREX の BREX-S1-00030 に違反することが書かれる。そのまま利用者に伝える |

2. `revised` か `status` で、発行番号が `002` 以降のときは、更新の理由も聞く（文と `updateReasonType`）。前の版の理由はすべて消えるので、1 つも無いままだと既定 BREX の BREX-S1-00038 に違反する。聞いた理由は、`start_revision` の `reason` に渡す
3. `start_revision` を、`path`・`csdbDirs`・`issueType`（と `reason`）を付けて、`write` なしで呼ぶ。応答を読む

   | 項目 | 読み方 |
   |---|---|
   | `started` | `false` なら、何も行われていない。`refusals[]` の `reason` と `message` を利用者に伝える（`already-in-work`: 既に作業中の版。`deleted-issue`: 削除済みの版。`id-referenced`: 前の版で `delete` の印が付いた要素の `id` を、残る部分が指している） |
   | `fileName` | 作業中の版のファイル名。**元のファイルとは別の名前**（`_012-00_` が `_012-01_` になる） |
   | `removed` | 消した前の版の変更の印。`deletedElements[]` は、取り除いた要素（前の版で `changeType="delete"` だったもの）の場所 |
   | `newSchemaErrors[]` | 作業中の版にしたことで増えた XSD のエラー。1 つでもあれば書き込めない。そのまま利用者に伝える |
   | `referencedFromOtherObjects[]` | 取り除いた要素の `id` を、ほかの DM・PM が指している場所。1 つでもあれば書き込めない |

4. 利用者に、次を示して確認を得る: 新しいファイル名、`issueType`、消える前の版の変更の印と理由の数、取り除かれる要素（`deletedElements[]`）。**発行済みの版のファイルは変わらないこと**も伝える
5. 同じ引数に `write: true` を付けて呼ぶ。`written[]` に新しいファイルのパスが入る。`writeSkipped` が `write-conflict` なら、同じ名前のファイルが既にある。上書きせず、利用者に伝える
6. **これ以降は、新しいファイル（`written[0]`）を対象にする。** reader でそのファイルを `includeLocation: true` で読み直す（場所は、元のファイルのものを使い回さない。`section` は `content` か `all` にする。`outline` では `textLocations` が付かない）

### 変更の印を付けるかを確かめる

作業中の版を編集するときは、変更の種類を利用者に聞く。**自分で決めない。**

| 利用者の答え | 行うこと |
|---|---|
| 技術的な変更など、変更の印で示す変更 | `edit_data_module` に `marking` を渡す（E3 の「変更の印」）。DM の `issueType` が `changed`・`status`（と `rinstate-changed`・`rinstate-status`）のときだけ付けられる |
| 編集上の変更（誤字の訂正など。`urt01`） | `marking` を渡さずに編集する（編集上の変更に、変更の印を付けてはならない）。理由は、編集の後に `add_reason_for_update` で足す（E7） |
| 変更の印も理由も付けない（試作・練習） | `marking` を渡さずに編集する。報告に、印も理由も付けていないと書く |

変更の印を付けるときは、次も聞く。

- 更新の理由の文と、`updateReasonType`（`urt02` 技術的な変更、`urt03` マークアップだけ、`urt04` 適用性だけ、`urt05` 参照だけ。`urt51` 以降はプロジェクトの値）
- `changeMark`（`"1"` は変更線などの表示を出す、`"0"` は出さない。プロジェクトの決まりによる）
- 同じ理由による変更が、この DM に既に入っているか。reader の応答の `identAndStatus.reasonForUpdates[]` に、その理由があるかを見る。あれば、その `id` を使う（E3）

### 利用者に聞く項目のまとめ方

聞く項目は多い。次の組ごとに、1 度にまとめて聞いてよい。**まとめて聞いても、決めるのは利用者である。** 答えを自分で補わない。

| 聞く時点 | まとめて聞いてよい項目 |
|---|---|
| E1 で DM を読み、内容を利用者に見せた後 | 何を変えるか。発行済みの版なら、版の扱い（3 つの選択肢）。作業中の版を始めるなら `issueType`。変更の種類（変更の印で示す変更か、編集上の変更か、印も理由も付けないか）。`revised`・`status` で発行番号が `002` 以降なら、更新の理由の文と `updateReasonType` |
| `start_revision` を書かずに呼び、結果を見せた後（E1a の 4） | 作業中の版を書き込んでよいか。変更の印を付けるなら、更新の理由の文・`updateReasonType`・`changeMark` |
| `edit_data_module` を書かずに呼び、結果を見せた後（E5） | 編集を書き込んでよいか。E7 を行うなら、その理由の文と `updateReasonType` |
| `add_reason_for_update` を書かずに呼び、結果を見せた後（E7 の 3） | 理由を書き込んでよいか |

- **書き込みの確認は、書かずに呼んだ結果を見せた後にだけ聞く。** 結果を見せる前に、後の書き込みの確認まで先に取らない
- 確認の後に、編集・文・値のどれかを変えたら、書かずに呼び直して結果を見せ、確認を取り直す

### E2 — 場所を得る

編集の場所は、reader の `read_data_module` を **`includeLocation: true`** で呼んだ応答にある、各節点の `location.path` で指す（例: `/dmodule/content/procedure/mainProcedure/proceduralStep[2]`）。`includeLocation` を渡さないと、応答に `location` も `textLocations` も無い（reader 0.12.0 以降）。

- path を自分で組み立てない。応答にある値を写す
- **場所を得るための読み取りは、`section` を `content` か `all`（既定）にする。** `section: "outline"` でも各節点に `location` は付くが、手順の節点の `textLocations` は付かない
- 手順（`kind: "step"`）の `text` は、その手順の直下にある段落（子を持たないもの）を、すべてつないだ文である。その段落には節点が無いので、`location` も無い。代わりに、手順の節点の `textLocations[]` に、つないだ段落ごとの場所が並びの順に入っている（`textLocations[0]` が `text` の最初の段落）。その段落だけを取り除く・移すときは、`textLocations[i].path` を `at` に写す（reader 0.13.0 以降）
- `text` のどの部分がどの段落かは、応答からは分からない（段落の文は空白 1 つでつないである）。段落が 2 つ以上あるときは、取り除く・移す段落が何番目かを、E5 で文と一緒に利用者に示して確かめる
- リストの項目（`listItem`）は、段落が 1 つのときにその段落が `text` になり、`textLocations` は付かない。項目そのもの（`listItem` の `location.path`）を対象にする。段落だけを取り除くと項目が空になり、XSD のエラーで書き込まれない
- reader が未接続（または 0.12.0 より前）で、利用者も path を示せないときは、編集しない。reader の接続・更新の方法を伝える。reader が 0.12.x のときは `textLocations` が無いので、手順の段落だけを取り除く・移す依頼は、できないと伝える

### E3 — 編集を組む

`edits[]` に、編集を 1 つずつ書く。

| `op` | 書く項目 |
|---|---|
| `insert` | `to`（基準の節点の path）、`position`、`node`（足す節点。下書きの節点 1 つ） |
| `remove` | `at`（取り除く節点の path） |
| `move` | `at`（移す節点の path）、`to`（基準の節点の path）、`position` |

`position` は `before`・`after`（基準の節点の前後）か、`first-child`・`last-child`（基準の節点の中）。

変更の印（`marking`。作業中の版で、利用者が印を付けると決めたときだけ）:

| 書く項目 | 内容 |
|---|---|
| `marking.reason` | その理由による**最初の**編集のとき。`text`（理由の文）と `updateReasonType`。識別・状態部に `reasonForUpdate` が 1 つ足され、付けた印がそれを指す。`id` は書かない（文書に無い値が作られる） |
| `marking.reasonRefId` | 同じ理由による **2 回目から**の編集のとき。識別・状態部に既にある `reasonForUpdate` の `id`（前の応答の `reasonForUpdate.id`、または reader の `identAndStatus.reasonForUpdates[].id`）。理由は足されない |
| `marking.changeMark` | `"1"` か `"0"`。利用者に聞いた値 |

- `reason` と `reasonRefId` は、どちらか一方だけを書く
- **1 回の呼び出しの編集は、すべて同じ理由を指す。** 理由が違う編集は、呼び出しを分ける
- 印の付き方: insert は足した要素に `add`。remove は**要素を取り除かずに** `delete` の印を付けて残す。move は、同じ親の中の移動だけができ、親に `modify` が付く
- 節点を別の内容に置き換えたいときは、同じ呼び出しで、元の節点の remove と、その節点の `after`（か `before`）への insert を書く。元の節点は `delete` の印で残り、新しい節点に `add` の印が付く
- 置き換える先の種類が元と違うとき（手順を注記にする、など）は、元の節点の隣に置けないことがある。並びの順（下の「決まり」）に合う場所への insert にする。たとえば、入れ子の手順を注記に置き換えるなら、注記は親の手順の中の、入れ子の手順より前に足す。同じ呼び出しの remove と insert で書くことは変わらない
- 親が変わる移動は断られる（`move-across-parents`）。元の場所の remove と、行き先への insert で書く
- 印の文・種類・`changeMark` を、自分で補わない

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
| `changes[]` | 適用した編集。報告に使う。`marking` を渡したときは、`mark`（印を付けた要素、`changeType`、指した理由の `id`）が付く。`kept: true` は、取り除かずに `delete` の印を付けて残した節点。`mark` が無い編集は、印が付いていない（`add`・`delete` の印を持つ要素の中の編集か、この作業中の版で足した要素の取り消し） |
| `reasonForUpdate` | `marking` を渡したとき、印が指す理由。`added: true` は、この呼び出しで足したもの。次の同じ理由の編集では、この `id` を `marking.reasonRefId` に渡す |

`refusals[].reason` の対処:

| `reason` | 意味 | 対処 |
|---|---|---|
| `released-issue` | 発行済みの版 | E1 に戻り、利用者に聞く。`marking` を渡したときは、`allowReleasedIssue` があっても断られる |
| `issue-type-not-markable` | DM の `issueType` が、変更の印を持てる値でない | 利用者に伝える。印を付けずに編集するか、やめるかを聞く。`issueType` を自分で書き換えない |
| `editorial-reason` | 理由が編集上の変更（`urt01`） | `marking` を渡さずに編集し、理由は `add_reason_for_update` で足す |
| `marking-input-error` | `marking` の形が受け付けられない（文が空、`reasonRefId` の `id` が無い、など） | `message` を読み、足りない値を利用者に聞く |
| `already-marked-deleted` | 取り除く節点に、既に `delete` の印がある | 既に取り除く印が付いていると伝える |
| `move-across-parents` | `marking` があり、移す先の親が違う | remove と insert で組み直す（利用者に確かめる） |
| `parent-not-markable` | `marking` があり、親が変更の印を持てない（説明 DM のいちばん上の段落の並べ替えなど） | remove と insert で組み直す（利用者に確かめる） |
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
- 変更の印（`marking` を渡したとき）: どの要素に、どの `changeType` が付くか（`changes[].mark`）。取り除く節点は、消えずに `delete` の印で残ること。足される理由（`reasonForUpdate`）
- `marking` を渡していないとき: **版と変更の印は変えていないこと。** `allowReleasedIssue: true` を渡したなら、発行済みの版を書き換えること
- 書き込むと、`path` のファイルが上書きされること

### E6 — 書き込んで、確かめる

1. 利用者の確認の後に、同じ `edits`（と、渡していたなら同じ `marking`）を `write: true` で呼ぶ。`path` のファイルが上書きされるので、`overwrite` は付けない（`add_reason_for_update` も同じ）。`written[]` にファイルのパスが入る。`writeSkipped` が返ったら、`reason` と `detail` をそのまま伝える
2. verify の `verify_data_module` で、書いたファイルを検証する
3. 報告する。[report.md](report.md) の雛形のうち、該当する項目だけを使い、「材料と目的」に「CSDB にある DM の構造の編集」と書く。次を必ず入れる
   - 対象の DM（DMC、版、ファイル）
   - 適用した編集（`changes[]`）
   - 作業中の版を始めたか（`start_revision`）。始めたなら、元のファイルと新しいファイル、`issueType`
   - 付けた変更の印（`changes[].mark`）と、指している理由（`id`・`updateReasonType`・文）。印を付けていないなら、付けていないこと
   - `allowReleasedIssue: true` を使ったかどうか
   - **発行はしていないこと**（`inWork` は `00` に戻していない。発行番号・発行日・品質保証の状態は変えていない）
   - 検証状態

### E7 — 変更の印を付けない変更の理由を足す（`add_reason_for_update`）

編集上の変更（`urt01`）の理由と、`issueType` が `revised`・`status` の版の理由は、`add_reason_for_update` で足す。変更の印で示す変更の理由には使わない（その理由は、`edit_data_module` の `marking.reason` が印と一緒に足す）。

1. 理由の文と `updateReasonType` を、利用者に聞く。自分で書かない
2. `add_reason_for_update` を、`path`（作業中の版のファイル）・`csdbDirs`・`reason` を付けて、`write` なしで呼ぶ。`added` が `false` なら、`refusals[]` を伝える（`released-issue`: 発行済みの版。先に E1a）
3. 足される理由（`reasonForUpdate`）を利用者に示し、確認の後に `write: true` で呼ぶ
4. 報告に、足した理由（`id`・`updateReasonType`・文）を書く

## やらないこと

- 利用者が明示していないのに `allowReleasedIssue: true` を渡すこと
- `newSchemaErrors[]` がある結果を、`apply_fixes` や `allowInvalid` で通そうとすること
- path や `id` を自分で作ること
- 編集できないときに、DM を reader で読んで下書きから作り直すこと
- 利用者に聞かずに、`start_revision` を呼ぶこと、`issueType`・理由の文・`updateReasonType`・`changeMark` を決めること
- 編集上の変更（`urt01`）に `marking` を渡すこと。変更の印で示す変更の理由を、`add_reason_for_update` で足すこと
- 同じ理由の 2 回目からの編集で、`marking.reason` を渡して理由を増やすこと（`marking.reasonRefId` を使う）
- 作業中の版を始めた後に、元の（発行済みの）ファイルを編集すること
- 書かずに呼んだ結果を見せる前に、書き込みの確認を取ること。確認の後に編集・文・値を変えたのに、書かずに呼び直さずに書き込むこと
- 「発行した」「版を上げた」と報告すること（発行番号は上がらない。行ったのは、作業中の版を始めることだけ）
- `marking` を渡していないのに、「変更の印を付けた」と報告すること
