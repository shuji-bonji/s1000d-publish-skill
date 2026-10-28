# データモジュールの種類の使い分け

`DraftDataModule.kind` は 5 種類。利用者の材料がどれに当たるかを、この表で決める。迷うときは利用者に聞く。**種類ごとの情報コードは、プロジェクトが決める**ので、種類を決めたら `allocation.rules.infoCodeByKind` にその種類の分があるかを確かめる（無ければ利用者に聞く。既定は無い）。

## 使い分け

| `kind` | 何を書くか | 材料の例 | 迷いどころ |
|---|---|---|---|
| `descript` | 説明。ものや仕組みが何か、どうなっているか。読む人が操作しなくてよい | 機器の概要、各部の名前と役割、用語の説明、画面の構成 | 「〜する」という操作が並ぶなら `proced` |
| `proced` | 手順。順番に行う作業。前提条件と事後条件を持てる | 扉を開ける、部品を交換する、設定を変える | 途中で「はい・いいえ」で道が分かれるなら、手順に書き込まず `fault` か `process` |
| `process` | 流れ。画面や段階が順に移り、条件で分岐し、利用者への問いを持つ。個々の作業は別の DM（`proced`）に任せて参照する | 画面遷移、操作の全体の流れ | 1 本道の作業なら `proced`。流れの各段階の細かい作業は、別の `proced` に書いて `dmRef` で結ぶ |
| `fault` | 故障探求。問いに答えると次の手順へ進み、原因に行き着く | 「きっぷが出ない」「電源が入らない」 | 問いと答えの分岐が中心なら `fault`。分岐が無ければ `proced` |
| `pm` | パブリケーションモジュール。DM を章立てに並べた目次。本文は持たない | 取扱説明書の全体構成 | DM ではなく PM（`pmCode`）。`dmRef` で DM を並べるだけ |

> [!NOTE]
> ここでの説明は、使い分けの目安を自分の言葉で書いたもの。種類ごとの正式な定義や、使える要素の全体は、利用者が入手した S1000D の配布物を読んで確かめる。この Skill は仕様書の文を写さない。

## 種類ごとに置ける節点（`DraftNode.kind`）

writer（core の `DRAFT_ALLOWED_CHILDREN`）が受け付ける組み合わせ。「親」の列が `body` なら、`DraftDataModule.body` の直下。置けない組み合わせは、XML を組み立てる前に `input-error`（`path` と受け付ける種類つき）で返る。

### descript

| 親 | 置ける子 |
|---|---|
| `body` / `section` | `para`、`admonition`、`section`、`figure`、`table`、`list` |
| `para` | `list` |
| `list` | `listItem`、`definition`（同じ list に混ぜない） |
| `listItem` | `para`、`admonition`、`list` |
| `definition` | `para` |
| `admonition` | `para` |

### proced

| 親 | 置ける子 |
|---|---|
| `body` | `step`、`requirements`、`figure`、`table` |
| `step` | `step`、`para`、`admonition`、`figure`、`table`、`list` |
| `para` / `list` / `listItem` / `definition` / `admonition` | descript と同じ |
| `requirements` | `para` |

`requirements` は、`step` より**前**にあれば前提条件、**後**にあれば事後条件になる。それぞれ 1 つまで。

### process

| 親 | 置ける子 |
|---|---|
| `body` | `sequence`、`section`、`if`（トップに `sequence` を置くなら、それだけ） |
| `sequence` | `section`、`if` |
| `section` | `step`、`figure`、`table`、`dialog`（`dialog` は最後） |
| `if` | `sequence`（1〜2 個。then と else） |
| `step` 以下 | proced と同じ |

`section` は、子の節点を持つか、`refs` に `dmRef` を 1 つだけ持つ（その画面の手順を別の DM に任せる）かのどちらか。`dialog` と `if` の `id` は変数名で、同じ DM で重複できない。

### fault

| 親 | 置ける子 |
|---|---|
| `body` | `step`、`requirements`、`figure`、`table` |
| `step` | `para`、`admonition`、`figure`、`table`（入れ子の `step` は不可） |
| `requirements` | `para` |

`step` はすべて `id` が必要。問い（`question`）を持つ `step` は `choices`（答えと、次の `step` の `id` を指す `next`）が必要。`next` は同じ DM の `step` の `id` でなければならない。

### pm

| 親 | 置ける子 |
|---|---|
| `body` / `section` | `section`、`entry` |

`section` は子を 1 つ以上持つ。`entry` は `refs` に `dmRef` か `pmRef` をちょうど 1 つ持つ。

## 節点ごとの必須項目

| `kind` | 必須 | 注意 |
|---|---|---|
| `admonition` | `level`（`warning` / `caution` / `note`）と、`text` か `para` の子 | 警告・注意は入力の位置にかかわらず先頭に寄せて組み立てられる。`listItem` の中に置けるのは `note` だけ |
| `table` | `rows`（1 行目が見出し行。全行のセルの数が同じ） | 空の行や、セルの数の違う行は `input-error` |
| `figure` | `title` と、`refs` の `graphic` か `multimedia`（混ぜない） | 図の `target` は ICN か素材の id |
| `definition` | `title`（用語） | 本文は `text` か `para` の子 |
| `list` | 1 つ以上の項目 | 順序付きは `ordered: true` |
| `step`（proced・process） | `title`、`text`、`para` / `figure` / `table` の子のどれか | 入れ子の `step` は、他の子より後ろに置く |
| `dialog`（process） | `id`、`text` | 選択肢は `choices`（`next` は使わない） |
| `if`（process） | `id`、`question` | 子は `sequence` を 1〜2 個 |
