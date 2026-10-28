# Publish Report の雛形

DM を作って入れた後の報告（Phase 9）。見出しは変えず、該当が無い項目は「なし」と書く。未接続の任意 MCP に頼る項目は「未実施（ツール未接続）」と書く。複数の DM は、表の行を増やす。

````markdown
# Publish Report

## 1. 材料と目的
- 材料: <文章 / メモ / 既存の DM のパス>
- 新規 / 書き直し: <新規 / 書き直し（対象: <DMC>、版の上げ方: <issuePolicy>）>
- 種類の判断: <descript / proced / process / fault>（理由: <利用者と決めた根拠>）

## 2. 採番の規則
- モデル識別コード / システム差異コード: <値>
- SNS: <systemCode>-<subSystemCode>-<subSubSystemCode>-<assyCode>
- 情報コード: <種類> <infoCode>-<infoCodeVariant>（利用者が指定。試作の見本の値を使ったときは「試作用の値で意味を持たない」と書く）
- 品目位置コード / CAGE コード: <..> / <..>
- 機密区分: <値>（利用者の指定。値は解釈していない）
- 台帳: <前回の台帳を使った / 新規>。保存先: <パス / 未保存>。次の ICN の連番: <icnNext>

## 3. 作った DM
| DMC | 題名（techName / infoName） | 種類 | 版 | ファイル | 書き込み |
|---|---|---|---|---|---|
| <dmCode> | <..> | <kind> | <issueNumber>-<inWork> | <fileName> | <written のパス / 未書き込み> |

## 4. プロジェクトの決まり（BREX）
- 読んだ BREX: <DMC / 未実施（ツール未接続）>
- 書き手が守る決まりの要約: <`explain_brex` の応答から。無ければ「なし」>
- 人の確認が要る項目（`manualChecks`）: <件数と内容 / なし>

## 5. 検証状態
| 工程 | 結果 | 備考 |
|---|---|---|
| XSD | <valid / invalid / schema-not-found / not-run> | schema-not-found と not-run は「検証していない」 |
| BREX | <passed / violations / incomplete / not-run> | `manualChecks` は人の確認が要る |
| 参照 | <resolved / unresolved> | 内訳: 仕様書への参照のみ <n>、素材の原本なし <n>、その他 <n> |

- 検証した時点: <組み立て直後 / 書き込み後に `verify_data_module` を実施>
- この報告は S1000D への準拠を判定していない。`valid` と `passed` は、XSD と機械で検査できる BREX のルールに誤りが無かったことだけを表す

## 6. 自動で直したもの（`appliedFixes[]`）
| 場所 | 変更前 → 変更後 | 根拠（`source`） |
|---|---|---|

## 7. 利用者が選んだもの
| 問題 | 候補 | 選んだ値 | `apply_fixes` の結果（`applied` / `rejected`） |
|---|---|---|---|

- 候補が無く、利用者に聞いて決めたもの（`confidence: none`）: <一覧 / なし>

## 8. DML との照合
- 照合した DML: <DML のコード / DML なし（CSDB に DML が無い）/ 未実施（ツール未接続）>
- 件数: found <n> / issue-mismatch <n> / not-found <n> / not-checked <n>
- 作った DM が DML に載っているか: <載っている / **載っていない（`notListed`）** / DML なし>
- DML の作成・更新は writer にまだ無いため、載っていない DM は利用者のツールで DML に足す必要がある

## 9. 未解決のまま残ったもの
- <問題と、残した理由（利用者の判断 / 候補なし / 素材なし）>

## 10. 任意の MCP
- brex（決まりの説明）: <実施 / 未実施（ツール未接続）>
- reader（既存の DM の読み出し）: <実施 / 該当なし / 未実施（ツール未接続）>
````

## 書くときの注意

- 「検証済み」「合格」の語は、`valid` と `passed` の範囲を超えて使わない
- 自動で直したものと、利用者が選んだものを混ぜない（根拠が違う）
- 書き込んでいないときは、見出し 3 の「書き込み」を「未書き込み」にし、見出し 8 は「未実施（書き込み前）」にする
- DML に載っていないことは、失敗ではなく事実として書く（DML の運用は利用者側）
