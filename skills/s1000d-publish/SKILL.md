---
name: s1000d-publish
description: 構造化した下書き（題名・段落・手順・警告・表・参照の木）から S1000D のデータモジュール（説明・手順・プロセス・故障探求）を 1 件ずつ作り、検証を通して CSDB に入れる手順。s1000d-writer-mcp の build_data_module で組み立て、残った修正は候補の中から利用者に選んでもらい、書き込み後に s1000d-verify-mcp で確かめ、DML との照合結果を含む Publish Report を返す。ユーザーが「この手順を S1000D の手順 DM にして」「仕様メモから説明 DM を作って」「この DM を書き直して版を上げて」「故障探求の DM を作りたい」「作った DM が DML に載っているか確認して」「技術文書をデータモジュールにして検証まで」などに言及したら、単発の build_data_module 呼び出しで済ませず必ずこの Skill を使う。体験型マニュアルの JSON からの一括変換は experience-to-s1000d。XML は書かず、S1000D への準拠は判定しない。
---

# s1000d-publish — 下書きから検証済みのデータモジュールへ

技術文書の書き手の材料（手順、仕様メモ、故障の切り分け）を、構造化した下書きにして、S1000D のデータモジュール（DM）を 1 件ずつ作る手順書。
自前の組み立て・検証の処理は持たない。組み立ては s1000d-writer-mcp、検証は s1000d-verify-mcp の応答を根拠にする。

共通の原則:

1. **XML を書かない。** XML は writer が組み立てる。この手順に、LLM が XML を書く工程は無い。書くのは下書きの木だけ
2. **判定はサーバーが下し、Skill は説明する。** XSD・BREX・参照の判定、修正の候補、DML の照合は MCP の応答をそのまま根拠にする。応答を読んで利用者に説明し、次の操作を決めるだけで、判定を上書きしない
3. **候補の外の値を使わない。** 修正は `fixes[]` の候補から選ぶ。`confidence: none` のときと、`valueForm` が `range` / `pattern` のときは利用者に聞く
4. **書き込む前に確かめる。** `write: true` は、利用者が書き込み先を確認してからにする
5. **検証状態を必ず報告に書く。** `schema-not-found` と `not-run` を「合格」と書かない。S1000D への準拠は判定しない
6. **前提の MCP が無いときは止まる。** 必須の MCP が接続されていなければ、その旨と接続の方法を伝えて止まる。任意の MCP が無いときは、報告の該当項目に「未実施（ツール未接続）」と書く

> [!NOTE]
> この Skill は ASD とは無関係の個人プロジェクトの一部。S1000D® は ASD の登録商標。S1000D の仕様書・XSD・既定 BREX は同梱されておらず、利用者が入手して `S1000D_SPEC_DIR` で指定する。

## 前提 MCP

| MCP | 必須/任意 | 最低版 | 役割 |
|---|---|---|---|
| s1000d-writer-mcp | **必須** | 0.1.1 | `build_data_module`（採番・組み立て・検証・書き込み）、`apply_fixes`（選んだ修正の適用） |
| s1000d-verify-mcp | **必須** | 0.6.1 | `verify_data_module`（書いた DM の検証）、`reconcile_dml`（DML との照合）、`check_references` |
| s1000d-brex-mcp | 任意 | 0.4.0 | `explain_brex`（プロジェクトの BREX の決まりの説明）、`lookup_value`（許可値） |
| s1000d-reader-mcp | 任意 | 0.7.1 | `read_data_module`（既存の DM を読んで書き直す） |

writer か verify が未接続なら、作業は成立しない。次を伝えて停止する。

- `/plugin install s1000d-writer-mcp@shuji-bonji` と `/plugin install s1000d-verify-mcp@shuji-bonji`
- または `npx @shuji-bonji/s1000d-writer-mcp` と `npx @shuji-bonji/s1000d-verify-mcp`
- XSD 検証には、利用者が s1000d.org から入手した配布物の展開先を `S1000D_SPEC_DIR` に指定する必要がある（Claude Desktop の plugin では環境変数が渡らないので `~/.config/s1000d-stack/config.json` の `specDir` に書く）

## 手順

### Phase 0 — 材料と出力先の確認

次を利用者に確かめる。推測で埋めない。

| 項目 | 確かめること |
|---|---|
| 材料 | 文章、メモ、箇条書き、既存の DM のパスのいずれか |
| 新規か書き直しか | 書き直しなら、対象の DM（`dmc` か `path`）と、版の上げ方（`issuePolicy`: 発行番号を上げる / 作業中番号を上げる / 据え置き） |
| 出力先の CSDB（`csdbDirs`） | 1 件目が書き込み先。**無いと、参照を解決できず、書き込みもできない** |
| 機密区分（`securityClassification`） | **必須。** 値は解釈されず、そのまま書かれる。利用者に聞く |
| 言語 | `languageIsoCode` と `countryIsoCode`（例: `ja` / `JP`） |
| 適用性の表示文（`applic.displayText`） | 省くと XSD の検証で落ちる。全製品に当てはまるなら「すべて」など、利用者の言葉で |

### Phase 1 — 種類を決める

材料が説明（`descript`）・手順（`proced`）・プロセス（`process`）・故障探求（`fault`）のどれかを決める。**迷うときは利用者に聞く。** 使い分けと、種類ごとに置ける節点は [references/kinds.md](references/kinds.md)。

1 つの材料に複数の種類が混ざっているときは、DM を分けることを提案する（例: 概要は `descript`、作業は `proced`）。分けた DM は `dmRef` で結ぶ。

### Phase 2 — プロジェクトの決まりを読む

brex があれば、`explain_brex` で、利用者のプロジェクトの BREX を読み、書き手が守る決まり（使えない要素、許可値、機密区分など）を要約する。`brex` にはプロジェクトの BREX の DMC かファイルのパスを渡す（省略すると `S1000D_SPEC_DIR` の既定 BREX）。種類を絞るなら `context`（例: `proced`）を渡す。

- 要約は応答の内容だけから書く。BREX に無い決まりを足さない
- 機械で検査できない決まり（`nonContext`）は、書き手が守る項目として並べるだけで、守れているかの判定はしない
- brex が未接続なら、報告の BREX の項目に「未実施（ツール未接続）」と書く。書き込み後の検証（Phase 6）は、BREX があれば verify が行う

### Phase 3 — 採番の規則を決める

`allocation`（モデル識別コード、システム差異コード、SNS、**種類ごとの情報コード**、品目位置コード、CAGE コード）を利用者に聞く。

- **情報コードの既定は無い。** 省略は `input-error`（`/allocation/rules/infoCodeByKind/<種類>`）。意味はプロジェクトが決めるので、利用者が登録したコードを聞く
- 試作で見本の値を示すときは「試作用の値で意味を持たない。実納品では登録したコードを使う」と必ず添える
- **`brexDmRef`（従う BREX の DMC）**: 省くと、`S1000D_SPEC_DIR` にある既定 BREX が入る。**既定 BREX が見つからないときは、`brexDmRef` ごと省かれ、BREX は 1 層も照合されない**（`verification.brex.state` は `not-run`。writer 0.1.2 以降は `_notes` にも「BREX was not checked」と出る）。この状態を「BREX 合格」と書かない。プロジェクトの BREX があれば、その DMC を利用者に聞いて渡す
- 続きの作業なら、前回の応答の `ledger` を `allocation.ledger` に、`icnNext` を `allocation.icn.next` に渡す。**渡さないと、同じ `sourceId` でも分解コードが進んで別の DMC になる**。書き直しで DMC を変えたくないときは、台帳を渡す（保存先の例: `<プロジェクト>.ledger.json`。保存は利用者の確認の後）

### Phase 4 — 下書きを作る

材料から `DraftDataModule` の木を作る。書き方の要点と最小の例は [references/draft.md](references/draft.md)。

- 書くのは題名・段落・手順・警告・表・図・参照だけ。DMC・ICN・BREX の参照・版の細部は書かない
- **参照は `text` に埋め込まず `refs` に書く。** 機密区分は必須、`applic` は省かない
- 前提条件・事後条件は `requirements` 節点（手順の前後に 1 つずつ）
- 既存の DM を書き直すときは、reader の `read_data_module`（`dmc` か `path`、`csdbDirs`）で読み、同じ語彙の木として直す。機密区分が付いた DM は、本文を引用する前に利用者に確かめる。reader が未接続なら、利用者に木の元になる文章をもらう
- 未確認の値（機密区分、適用性、企業コード）を埋めない

下書きを作ったら、**書く前に利用者に内容を見せて確認する**。木をそのまま見せず、題名・手順の番号と文・警告の位置が分かる形に直す。

### Phase 5 — 書かずに組み立てて検証する

`build_data_module` を `write: false` で呼ぶ（`draft`、`allocation`、`csdbDirs`）。`includeXml` という引数は無い（XML は常に応答に付く。長いので、利用者に XML を貼って見せない）。

**`input-error`**（`error.code`）が返ったときは、何も書かれていない。`error.path` の箇所だけを直して呼び直す（読み方は [references/draft.md](references/draft.md)）。`path` が `/allocation/...` のときは、下書きではなく採番の規則の誤りなので、利用者に聞く。

応答の読み方:

| 項目 | 読むこと |
|---|---|
| `dmCode`、`fileName`、`issueInfo` | 付いた DMC、ファイル名、版 |
| `verification` | `schema` / `brex` / `references` の状態（下の書き方） |
| `appliedFixes[]` | 自動で直したもの（`confidence: exact`）。変更前後を報告に載せる |
| `fixes[]`、`byConfidence` | 残った修正 |
| `ledger`、`icnNext` | 次の呼び出しに渡す |
| `_notes` | 補った値（`issueDate`、企業コード、`preliminaryRqmts` の補いなど）。省かず確かめる |

検証状態の書き方（原則 5）:

| 応答 | 書くこと | 書いてはいけないこと |
|---|---|---|
| `schema.state: valid` | 「XSD の検証で違反なし」 | 「S1000D 準拠」 |
| `schema.state: schema-not-found` | 「XSD が見つからず、XSD 検証は行われていない」。`S1000D_SPEC_DIR` の設定を案内 | 「合格」「問題なし」 |
| `brex.state: passed` | 「機械で検査できる BREX のルールで違反なし。`manualChecks` は人の確認が要る」 | 「BREX 準拠」 |
| `brex.state: incomplete` / `not-run` | 「BREX は未確認」 | 「合格」 |
| `references.state: unresolved` | 内訳を確かめて書く（次の注意） | 一括で「参照切れ」「問題なし」 |

> [!NOTE]
> `references.state` が `unresolved` でも、原因が仕様書の章への `dmRef`（モデル識別コード `S1000D`。既定 BREX への参照など）だけのことがある。`fixes[]` が空で `omitted.specificationReferences` が `unresolvedCount` と同じなら、残りは仕様書への参照だけで、CSDB に無いのが通常。内訳に迷ったら verify の `check_references` で確かめ、「仕様書への参照のみ未解決」と書く。`csdbDirs` を渡さないと、参照を探す先が無いという `none` の修正が 1 件付く。

> [!NOTE]
> **`totals` や参照の状態を読むときの注意（spec モード: 配布物の既定 BREX を使うとき）。** 既定 BREX は仕様書の配布物（`S1000D_SPEC_DIR`）の中にあり、CSDB のフォルダーには無い。`csdbDirs` に書き込み先だけを渡すと、`brexDmRef` が指す既定 BREX への参照が未解決として数えられることがある（core 0.8.1 以降、つまり verify 0.6.1 では、BREX の工程でたどれた層は解決済みになる）。逆に、`verify_csdb` に既定 BREX のフォルダーまで渡すと、既定 BREX 自身の DM も検証の対象に入り、`totals` の件数に加わる。自分が作った DM の件数と合わせるときは、`results[]` の `path` で分けて数える。

### Phase 6 — 残った修正を片付ける

`fixes[]` の各項目を `confidence` で分ける。

| `confidence` | 意味 | 扱い |
|---|---|---|
| `exact` | 候補が 1 つで値も決まっている | サーバーが自動で適用済み（`appliedFixes[]`）。報告に載せるだけ |
| `several` | 候補が複数 | 各候補の `value`、`description`、`source` を示して、利用者に選んでもらう |
| `none` | 候補を決められない | `problem` と `note` をそのまま伝えて、利用者に聞く。値を作らない |

候補の `valueForm` が `range` / `pattern` のときは値ではなく**制約**。範囲や正規表現を示し、合う値を利用者が決める。

選んでもらった修正は `apply_fixes` で適用する。

- `selections[]` に、`fixes[]` の `location` と**候補 1 つ**（`action`、`target`、`value`）を写す。候補の外の値や範囲に合わない値は `rejected[]` に理由付きで返る
- `apply_fixes` は**既存の文書**（`path` か `xml`）を受ける。まだ書いていない DM は、Phase 5 の応答の `xml` を渡す。書いた後なら `path` を渡す（`write: true` なら、その場で上書きされる）
- `action: "provide"` の候補は、文書ではなく製品の値（`product`）を与えるもの
- 適用後に再検証される。`verification` を読み直す

下書きの内容に直す問題（文の誤り、足りない節）は、修正の候補ではなく、Phase 4 に戻って下書きを直す。

### Phase 7 — 書き込む

1. 書き込み先（`csdbDirs` の 1 件目）、ファイル名、版を利用者に示し、**確認を得る**
2. `build_data_module` を `write: true` で呼ぶ。`overwrite` は既定の `false` のまま。`write-conflict` が返ったら、同じ名前のファイルがあるので、上書きするか、版を上げるかを利用者に聞く
   - **版に注意する。** 下書きに `issueInfo` を決めていないと、CSDB に同じ DMC の DM が既にあるとき、版は `issuePolicy`（既定は発行番号を上げる）に従って進む。つまり同じ下書きを `write: true` でもう一度呼んでも `write-conflict` にはならず、次の版（`_002-00`）が書かれる。意図しない版が増えないように、版を決めてから呼ぶ（下書きに `issueInfo` を入れる）。書き直しや呼び直しのときは、前の応答の `issueInfo` を下書きに渡す。同じ版を置き換えるのは、利用者が明示したときの `overwrite: true` だけ
3. XSD に違反する DM は書かれない（`schema-invalid`）。`allowInvalid: true` は、利用者が明示したときだけ
4. 応答の `written`（書いたパス）を控える。`ledger` と `icnNext` の保存を提案する
5. 書いた DM を `verify_data_module`（`path`、`csdbDirs`）で確かめる。結果を Phase 5 の書き方で報告する。書いた直後のファイルが verify・reader から見えるかは Phase 8 の注意を読む

### Phase 8 — DML との照合

verify の `reconcile_dml` を `csdbDirs` で呼ぶ（`path` を省くと、CSDB にある DML と DDN をすべて照合する）。

- **書いた直後の DM が見えること。** verify 0.6.1・reader 0.7.1 以降は、サーバーを起動した後に書いたファイルも次の呼び出しから見える。それより古い版は、起動時に作った CSDB の索引を使い続けるので、書いた直後の DM・PM が見えない（見つからない、件数に入らない）ことがある。その場合は、該当の MCP（verify・reader）を起動し直してからやり直す。作った DM が `not-found` や `notListed` に出ない（そもそも見えない）ときは、DML のせいと決める前にこれを疑う
- 応答の `lists[]` が空なら、CSDB に DML が無い。「DML なし」と報告する
- 各 DML の `counts`（`found` / `issue-mismatch` / `not-found` / `not-checked`）と、`notListed`（CSDB にあるのに DML に載っていない DM・PM）を読む。**作った DM が `notListed` にあれば、DML に載っていない**
- `issue-mismatch` は、DML が指す版と CSDB の版が違う。`foundIssueInfo`（CSDB の最新の版）を添えて報告する
- **DML の作成・更新は writer にまだ無い。** 載っていなければ、利用者に伝える。DML に行を足す作業は、利用者のツールで行う
- `dmlType` などの値は解釈しない。応答の値をそのまま書く

### Phase 9 — Publish Report

[references/report.md](references/report.md) の雛形で報告する。複数の DM を続けて作ったときは、DM ごとに Phase 4〜8 を繰り返し、報告は 1 つにまとめる（`ledger` は DM ごとに更新して次へ渡す）。

## やらないこと

- XML を書く、または DMC・ICN・BREX の参照を自分で作ること
- 情報コード・機密区分・適用性を、利用者に聞かずに決めること
- 候補の外の値で修正すること
- `schema-not-found` と `not-run` を合格と書くこと
- `valid` と `passed` を「S1000D に準拠している」と書くこと
- BREX の決まりを、応答に無い内容で説明すること
- DML を作る・更新する（writer にまだ無い）
