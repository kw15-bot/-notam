# geoplot-mil NOTAM機能 引き継ぎ資料

作成日: 2026-09-21 / 対象: 「航行警報海域可視化」(geoplot-mil) への **NOTAM地図化機能の追加**

> このファイルは NOTAM 機能の引き継ぎ専用。既存の MSA(海事局航行警告) 側の引き継ぎは別紙 `HANDOFF.md`
> (geoplot-mil_bundle_2026-09-20.zip 内)を参照。そちらの未完了事項（デプロイ・ワークフロー実行確認など）は本資料では扱わない。

---

## 0. 30秒サマリ

- **目的**: 中国・香港・マカオ・台湾が発行するNOTAMを10分おきに自動収集し、航行警報海域と同じように地図化する。失効分はArchiveに蓄積する。
- **データ源**: FAA の **NMS-API**（認証情報は取得済み。環境は **staging(`api-staging.cgifederal-aim.com`)** と確定）。
- **2026-09-24 に実データ検証済み**: probe（差分）と `--bootstrap il`（1,717件）の両方を staging で実行。収集スクリプトは実データで最後まで動作し、§8 の未確認事項の大半が確定した。詳細は §8・§14。
- **実データ検証中に1件バグを発見し修正済み**: APIが座標未設定を `Point[0,0]` で返すケースを実座標として誤採用していた（`build_geometry()`）。修正・回帰テスト追加・テスト計32件、すべて合格。
- **できていること**: 収集スクリプト・調査スクリプト・GitHub Actions ワークフロー2本・テスト33件（合成データ33件＋実データでの通し確認2回）。`focus_tag`(目的別の粗い絞り込みタグ。§5.4参照)を追加済み。**ビューア(`geoplot-mil.html` v1.9.0)へのNOTAMレイヤー統合も実装済み**（§10）。
- **次にやること**: 残る未確認事項（§8の#9〜11: Qコード公式確認・再配布条件・レート制限）の確認 → **本番(prod)環境かどうかの確認**（今回はstagingのみ）→ 初回投入(`--bootstrap il`)→ ワークフロー運用開始 → ビューアの実ブラウザでの動作確認。

---

## 1. ユーザーの決定事項（確定済み）

| 項目 | 決定 |
|---|---|
| 対象 | **中国が発行するNOTAMのみ**。当初「全世界」だったが絞り込み。**中国本土・香港・マカオ・台湾**を含める（ユーザー指示） |
| 収集頻度 | 10分おき程度 |
| 運用形態 | GitHub で公開 + **cron-job.org から GitHub Actions(workflow_dispatch) を叩く**（既存MSA側と同じ運用） |
| Archive | 失効・取消は**即時Archive移行**で問題ない。完全削除はしない（蓄積） |
| 確度判定 | **不要。人間が実施する**（軍事かどうかの自動判定機能は作らない） |
| 開発方針 | 形が決まるまでは git を使わず**手元で試したい** |
| 認証情報 | NMS-API の KEY/SECRET は取得済み（環境は不明 → §8 参照） |

> 用語: 台湾・香港・マカオは、ICAO地名指標の先頭2文字で機械的にグループ分けしているだけ（`area_group` = CN/HK/MO/TW）。表示ラベルは「中国本土/香港/マカオ/台湾」。

---

## 2. 調査結果（データ源の比較）

**結論: FAA NMS-API が本命。** 申請制だが、緯度経度・半径での検索、GeoJSON出力、`lastUpdatedDate` による差分取得（新規・更新・キャンセルを含む）に対応し、10分おきのポーリングに向く。

| ソース | 評価 |
|---|---|
| **FAA NMS-API** | 採用。FAA が唯一の正式な情報源にする方針。取得は FAA へのメール申請（`NOTAMS@faa.gov`）。GeoJSON/AIXM 5.1 |
| FAA SWIM (SCDS) / NMS-NDS・NPS | JMS によるプッシュ配信。常時接続が必要なので **GitHub Actions(cron方式)では受けられない**（設計上の判断）。SCDS は非運用目的・利用契約あり |
| ICAO API Data Service | 蓄積版は3時間ごとの取得、リアルタイム版は地点リスト必須、無料枠は約100コール。全世界巡回に不向き。取得元は米国 DINS（NMS移行後の稼働は未確認） |
| EUROCONTROL EAD (INO) | 世界の国際NOTAMを集めた中央リポジトリだが、EAD顧客向け・契約が必要（料金は未確認） |
| Notamify / Aviation Edge | 商用API。NMS が使えない場合のフォールバック候補 |
| 各国AIS直接 | 国ごとに別実装。中国の国内向けNOTAMを公開する一般向けサイトは**調べた範囲では見つからなかった** |
| **旧 `external-api.faa.gov/notamapi`** | **使わない**。旧システム側で廃止方向。2026-07-28時点の確認で 401 |

**参考実装（個人開発・OSS）**
- `RISCfuture/notams`（TypeScript）: NMS を定期取得→PostgreSQL→独自JSON API。最も近い参考実装
- `aerocontext`（Rust）: NMS と Leidos の両方を照合
- `faa-nms-api`（npm）: 型付きクライアント、本番/ステージング切替、トークン自動更新
- `NagoDede/notamloader`（Go）: 各国AISのWebフォーム取得。2021年で更新停止
- パーサ: `dfelix/notam-decoder`（JS）、`svoop/notam`（Ruby・保守終了・ICAO附属書15準拠のみ）

**制約・リスク（再掲）**
1. **NMS に流れるのは国際配信されたNOTAMが中心と推測**（未検証）。中国の国内向けのみのNOTAMは見えない可能性。
2. **NMSデータの再配布条件は未確認**。GitHub を public にして全文を公開してよいか、FAA(`NOTAMS@faa.gov`)への照会を推奨。確認できるまでは private リポジトリ、または `raw_text` を出力から外す選択肢。
3. NOTAM は運航・安全判断に使うものではない旨の注意（SCDS の条件）。本ツールは情報可視化用途。

---

## 3. NMS-API 仕様の要点（`nms-api.yaml` **1.0.18**, 2026-02-12 が最新）

ユーザー提供の `nms-api-1_0_17.yaml` は旧版。差分はロケーション形式の緩和と、GeoJSON内のフィールド名変更（`simpleText`/`formattedText`、旧 `domestic_message`/`icao_message`）など。コレクタは両方の名前を受ける。

### 接続
| 環境 | ホスト |
|---|---|
| FIT | `https://api-fit.cgifederal-aim.com` |
| Staging (Pre-Prod) | `https://api-staging.cgifederal-aim.com` |
| Prod | `https://api-nms.aim.faa.gov` |

- **認証**: `POST {host}/v1/auth/token`（**`/nmsapi` を付けない**）、Basic認証 `client_id:client_secret`、body `grant_type=client_credentials`（`application/x-www-form-urlencoded`。Content-Type を余計に付けると `Required param : grant_type` エラー）。
- トークン有効期間 **1799秒(30分)**。切れると 401。配布Excelの KEY = client_id、SECRET = client_secret。
- API本体は `{host}/nmsapi/v1/...`、ヘッダ `Authorization: Bearer <token>`。ブラウザからは不可（機械間インターフェース）。
- FAQ上の想定利用: 「直近3分など必要な期間の差分を取る」「全件が必要なら分類ごとの一括取得」。

### エンドポイント
| パス | 内容 |
|---|---|
| `GET /v1/ping` | 疎通（仕様書のパス一覧には無いが FAQ/curl例に記載） |
| `GET /v1/notams` | フィルタ検索。**ヘッダ `nmsResponseFormat: AIXM\|GEOJSON` が必須**。条件はAND結合。パラメータ無しはエラー |
| `GET /v1/notams/checklist` | チェックリスト（`accountability`/`classification`/`location`） |
| `GET /v1/notams/il`, `/il/{classification}` | 初期ロード。**AIXM(SOAP封筒)専用**。→ **このプロジェクトでは使わない** |
| `GET /v1/locationseries` | ロケーション↔国際シリーズ(accountId, AFTNアドレス)対応。`lastUpdatedDate` あり |
| `GET /v1/content/{token}` | 一括ファイルの取得。FAQ上は `/nmsapi/v1/content/{token}` で**Bearerトークン必須**（旧サンプルJSONの GCS署名URL直リンクは古い） |

### `/v1/notams` のフィルタ
`accountability`, `classification`, `location`(3〜4文字), `notamNumber`, `nmsId`, `feature`, `freeText`, `effectiveStartDate`/`effectiveEndDate`, `lastUpdatedDate`, `latitude`+`longitude`+`radius`(radius ≤ 100NM), `allowRedirect`。
- **FIR・国での絞り込みは無い** → 中国分は**全世界の差分を取ってクライアント側で絞る**設計にした。
- **`lastUpdatedDate`: 窓は最大24時間**。新規・更新・キャンセル(非有効)を返す。処理が30秒超で **408**。
- **`classification` を単独指定**すると、その分類の全件を含む**圧縮ファイルへの相対パス**（5分で失効）を返す。`nmsResponseFormat` に従い **AIXM か GeoJSON**。→ コレクタの `--bootstrap il` はこの経路（GeoJSON指定）。ファイル内部の構造は**未確認**（パーサは寛容に作ってある）。
- `classification` の値: `INTERNATIONAL, MILITARY, LOCAL_MILITARY, DOMESTIC, FDC`（MILITARY は米軍系。中国NOTAMは INTERNATIONAL と推測 → probe で確認）。

### GeoJSON の中身（1件）
`properties.coreNOTAMData.notam` に `id, series, number, year, type, issued, affectedFir, selectionCode(Qコード), location, icaoLocation, classification, accountId, effectiveStart, effectiveEnd, estimated, schedule, lowerLimit, upperLimit, minimumFl, maximumFl, coordinates, radius, text, lastUpdated, cancelationDate` など。`notamTranslation`(type=`ICAO`/`LOCAL_FORMAT`) に整形済み全文。`geometry` は GeometryCollection（点＋面）。

---

## 4. 実データから得た知見（ユーザー提供のNOTAM2件）

```
A4703/26 NOTAMN Q)ZXXX/QRTCA/IV/BO/W/000/999/2536N11651E039 A)ZGZU ZSHA B)2609172330 C)2609180330
E) A TEMPORARY RESTRICTED AREA ESTABLISHED BOUNDED BY: N230636E1162812-N234042E1174425-N233012E1174957-N225820E1163232- N230636E1162812. HEIGHT: 35,000M AND BELOW. F)GND G)UNL
A4705/26 NOTAMN Q)ZSHA/QRTCA/IV/BO/W/000/999/2343N11742E017 A)ZSHA B)2609172330 C)2609180330
E) A TEMPORARY RESTRICTED AREA ESTABLISHED BOUNDED BY: N240013E1174036-N235732E1175242-N232609E1174457-N232853E1173231- N240013E1174036 . HEIGHT: 50,000M AND BELOW. F)GND G)UNL
```
ユーザーによれば、これらは実際に**軍事用途と判定されたもの**（判定の根拠は聞いていない。MSA航行警告との重なりの可能性があるが未確認）。

**Q) 行の読み方**: `FIR / Qコード / 交通(IV=IFR+VFR) / 目的(BO) / 範囲(W=航行警報) / 下限FL / 上限FL / 中心座標+半径NM`
- `QRTCA` = Q + **RT**(臨時制限空域) + **CA**(発動)。2文字目 R=空域制限、W=警告。読み方は ICAO Doc 8126 に基づく私の理解で、**公式表での確認は未了**。
- **Qコードは軍事の証拠にならない**（理由の欄が無い。ロケット打ち上げ・要人・イベント等でも使われる）。逆に**軍事関連がQR/QWだけとも限らない**（QA=経路変更、QF=飛行場閉鎖、QG=GNSS妨害、QXXXX 等でも出うる）。→ ユーザー決定により、確度の自動判定は作らず、`category`(restriction/warning/other)を絞り込み用の分類としてだけ提供。
- 時刻: B)=開始、C)=終了（UTC, YYMMDDHHMM）。上例は 2026-09-17 23:30Z 〜 09-18 03:30Z。
- **`ZXXX`** = 複数FIRにまたがるNOTAMのQ項FIR欄（中国の仮コード）。→ 接頭辞 `ZX` を中国扱いにした。
- **A) が複数地点**（`ZGZU ZSHA`）になりうる。NMSがこれを `icaoLocation` にどう入れるかは未確認 → 空白等で分割して判定。

**Q項の中心座標の意味と信頼性**
- 影響範囲を**大まかに囲む円**の中心と半径（度分のみ、半径はNM。999=FIR全体以上）。**検索・絞り込み用の目安で、正確な境界ではない**。正確な形は E) 項。
- 2件で検証: 半径は多角形の外接円にほぼ一致（39NM / 17NM。計算値 39.5 / 17.2 でわずかに小さい＝厳密な外接ではない）。中心は A4705 では 0.7NM 差で一致するが、**A4703 では多角形の重心から138NM（主に緯度方向）ずれ**、Q項の円は多角形を含んでいない。原因（誤記か算出方法か）は不明。
- ⇒ **図形は E) 項の座標列を正とし、Q項の円は代替にとどめる**。ずれは `qline_offset_nm` で記録。
- NMS の緯度経度・半径検索はおそらくQ項の値を使うので、A4703 のようなNOTAMは検索で漏れうる（推測）。差分取得方式には影響しない。

**E項の座標書式**: `N230636E1162812` = 北緯23°06′36″ 東経116°28′12″（度分秒。度分のみの `N2306E11628` もパース対象）。先頭点に戻ってリングが閉じる。

---

## 5. 成果物

```
HANDOFF_NOTAM.md                       本資料
notam_cn_probe.py                      調査用（実データで前提を確認する。書き込みは probe_out/ のみ）
notam_cn_collect.py                    収集本体（差分取得・絞り込み・GeoJSON化・Archive）
.github/workflows/notam-probe.yml      probe を Actions で手動実行
.github/workflows/notam-collect.yml    収集（cron-job.org から起動）
dev/test_notam_cn_collect.py           テスト31件（標準ライブラリのみ・合成データ）
dev/mock_nms.py                        probe用の合成データのモックAPI（127.0.0.1:8765）
.gitignore_notam_snippet.txt           .gitignore に追記する内容
```
依存: Python 3.9+ 標準ライブラリのみ。

### 5.1 `notam_cn_probe.py`（調査）
実データで次を1回で確認する: ①認証 ②差分の窓が何時間幅まで通るか（408の有無） ③中国・香港・マカオ・台湾のNOTAMが返るか、および `classification`/`accountId`/`affectedFir`/Qコード/座標+半径/図形の入り方 ④`locationseries` から地点と accountId が引けるか（引ければ `accountability` でサーバ側絞り込みできる可能性）⑤FIR候補・空港の個別問い合わせ件数。
- 環境変数: `NMS_CLIENT_ID`, `NMS_CLIENT_SECRET`, `NMS_ENV`(fit|staging|prod、**既定 prod**), `NMS_HOST`(テスト用上書き), `CN_EXTRA_PREFIXES`, `PROBE_OUT`
- 出力: `probe_out/report.json`, `cn_sample.json`, `locationseries_cn.json`。**トークン・シークレットは一切出力しない**（テスト済み）。
- `cn_sample.json` はそのまま収集スクリプトの `--fixture` に渡せる。

### 5.2 `notam_cn_collect.py`（収集）
**対象判定**: `icaoLocation` か `affectedFir` を空白等で分割し、**4文字トークン**の先頭2文字が次のもの。
`CN: ZB ZG ZH ZJ ZL ZP ZS ZU ZW ZY ZX / HK: VH / MO: VM / TW: RC`
（**4文字に限る理由**: 米国ARTCCの3文字コード `ZBW ZHU ZJX ZLA ZSE ZUA` と先頭2文字が衝突するため。モンゴル ZM、北朝鮮 ZK は対象外）

**フロー**: 状態読込 → 取得(差分 or fixture or bootstrap) → 対象抽出・レコード化 → 置換/取消リンク → Archive移動 → GeoJSON生成 → 変化があるファイルだけ書き込み。

**CLI**
| オプション | 意味 |
|---|---|
| `--out-dir` | 出力先（既定 `./notam_out`） |
| `--fixture <json>` | API を呼ばず取り込む（Feature配列 / NMS応答 / FeatureCollection） |
| `--bootstrap il` | 初回全件投入。`GET /v1/notams?classification=<分類>&allowRedirect=false`(GEOJSON)→content→gz解凍→JSON/JSONL |
| `--bootstrap locations` | 代替。`locationseries` から対象地点を列挙→`location=` で個別問い合わせ |
| `--il-class` | il の分類（既定 INTERNATIONAL） |
| `--extra-locations` | locations 方式で追加する地点(カンマ区切り、FIRコード等) |
| `--keep-ended-days` | 失効後に有効側へ残す日数（**既定0=即時Archive**） |
| `--keep-notice-days` | 取消通知(C)を有効側に残す日数（既定2。順序入れ替わり対策） |
| `--heartbeat-hours` | 変化が無くても meta.json を更新する間隔（既定3時間） |
| `--now` | 現在時刻の上書き（テスト用） |
| `--dry-run` | 何も書かない |

環境変数: `NMS_CLIENT_ID`, `NMS_CLIENT_SECRET`, `NMS_ENV`(既定 prod), `NMS_HOST`。**終了コード: 0=成功 / 2=取得失敗**（Actions を赤くし、`last_success` を進めない）。

**差分の窓**: `since = 最終成功 - 10分`（重ね取り）。上限 23時間30分（超えたら警告して丸める→取りこぼしの可能性があるので bootstrap を再実行）。初回(meta無し)は直近6時間＋警告。408等は5秒待って1回だけ再試行。

### 5.3 出力構成
```
notam_out/
  state_active.json                 有効・未来のNOTAM(＋取消通知)。真実の源 {id: レコード}
  notam_cn.geojson                  ビューア用。有効(active)と未来(upcoming)のみ。毎回作り直し
  archive/notam_YYYY-MM.geojson     失効・取消したNOTAM。失効した月ごと。追記のみ・idで重複排除
  meta.json                         last_success 等。データ変化時＋3時間おきだけ更新（コミット抑制）
```
**ライフサイクル**: `upcoming`(開始前) → `active` → `expired`(終了) / `cancelled`(後続のR/Cで打ち切り) → **同じ回のうちに Archive へ移動**。取消通知(type C)は図形を持たず、描画せず、2日だけ有効側に保持してからArchiveへ。終了時刻が無いNOTAM(PERM等)は自動ではArchiveしない（取消でのみ終わる）。
**置換/取消の検出**: NOTAM本文の `A0110/26 NOTAMR A0101/26` の参照から元NOTAMの終了を発行時刻で打ち切る（追加のみ・消さない）。API側に「取消済み」を示す項目があるかは未確認。`cancelationDate` は、サンプルでは `effectiveEnd` と同値のことがあり、意味は不確か（現状は min(終了, 取消日, 打切り) を有効終了として使用）。
**無駄コミット対策**: 内容が同一なら書き込まない。Archive済みが差分に再登場しても `first_seen` を保持してファイルが変わらない。

**図形の優先順位**（`geometry_source`）: `api`(APIの面) > `text-polygon`(E項の座標列) > `qline-circle`(Q項の円・半径≤250NM・64角形) > `qline-point`(Q項の点・半径>250NMなど) > `api-point`。図形が全く取れないNOTAMは state には残すが geojson には出さない。

**Feature の properties**（既存 `military.geojson` の命名に寄せた）
`title, date, issuer(=accountId), raw_text, valid_start, valid_end, valid_raw, kind(area|point), source("nms"), nms_id, number, series, notam_type, area_group(CN|HK|MO|TW), icao_location, fir, q_code, category(restriction|warning|other), focus_tag(restriction|flag|watch|leisure|plaintext|admin), lower, upper, radius_nm, geometry_source, qline_offset_nm, estimated_end, status(active|upcoming|expired|cancelled|cancel-notice), ended_by, first_seen, last_updated`
- `category`: QR* = restriction、QW* = warning、その他 = other（**Qコード分類の根拠は Doc 8126 の主題区分に基づく想定**。2026-09-24 に FAA 7930.2 Appendix B の正式テーブルで裏取り済み）。
- `focus_tag`（2026-09-24 追加）: `category` を補い、Wグループの中身をさらに粗く仕分ける絞り込み用タグ。**軍事かどうかの自動判定ではない**（あくまで人間が優先的に見る順番を決めるためのラベル）。
  - `restriction` = Rグループ全部(RA/RD/RM/RO/RP/RR/RT)。
  - `flag` = WM(射撃/砲撃。**実データでは花火が多数混入**)・WE(演習)・WF(空中給油)・WR(放射性/有害物質)・WD/WH(爆破)。
  - `watch` = WU(無人機。**実データは民間ドローン届出が大半**)・WL(自由気球)・WC(係留気球/凧)、および未知のWサブタイプ(安全側に倒す)。
  - `leisure` = WA/WB/WG/WJ/WP/WV/WY/WZ(航空ショー・曲技・グライダー・バナー曳航・パラ系・編隊飛行・航空測量・模型飛行)。
  - `plaintext` = QXXXX等、Qコード上分類できず本文(E項)を読むしかないもの。
  - `admin` = R/W以外(空港施設・航法援助施設・運航方式等)。基本的に対象外。
  - 実データ(1,717件)での分布: `admin` 49.9% / `watch` 40.0% / `flag` 5.1% / `plaintext` 4.0% / `leisure` 0.8% / `restriction` 0.2%。**ビューアの既定表示は `admin` を除外**するだけでも見るべき件数が約半分に減る。
  - 次の判断材料が貯まったら、`flag`/`watch` の中を本文キーワード(FORBIDDEN/PROHIBITED/MILITARY等)でさらに絞れるか検討する（ユーザー方針: 今回は保留）。

### 5.4 ワークフロー
- **`notam-probe.yml`**: 手動実行。入力 `env`(prod/staging/fit)。Secrets: `NMS_CLIENT_ID`, `NMS_CLIENT_SECRET`。`probe_out/` をアーティファクトに7日保存。
- **`notam-collect.yml`**: `workflow_dispatch` のみ。`concurrency` で同時実行を防止。収集失敗(終了コード2)ならジョブが赤くなり**コミットしない**。変更があれば `notam_out/` を `[skip ci]` でコミットし、`git pull --rebase` + push を最大5回リトライ。Variables: `NMS_ENV`（未設定なら prod）。
- **cron-job.org 側**: `POST https://api.github.com/repos/<OWNER>/<REPO>/actions/workflows/notam-collect.yml/dispatches`、ヘッダ `Authorization: Bearer <PAT>` / `Accept: application/vnd.github+json` / `X-GitHub-Api-Version: 2022-11-28`、body `{"ref":"main"}`、間隔10分。PATの権限は既存MSA用と同様（Actions書き込み）。**PAT失効で静かに止まる問題は既存MSA側と同じ**（Actions側の失敗検知は今回の終了コード2で一部改善したが、cron側の失効は検知できない）。

---

## 6. テスト状況（32件、うち31件は合成データ・1件は実データを踏まえた回帰テスト）

`python dev/test_notam_cn_collect.py`（リポジトリのルートから）。カバー範囲: 対象抽出(米ARTCC・モンゴル・日本の除外、香港・マカオ・台湾の包含)、複数地点A)、ZXXX、図形の優先順位、E項多角形の度分秒パース(閉じたリング・複数エリア・分60超の無効・1点のみは面にしない)、Q項座標パース、**API座標未設定(Point[0,0])の除外**、冪等性、更新、置換/取消、取消通知が先に届くケース、即時Archive、Archive月＝失効月、Archive再登場でファイル不変、追記のみ、PERM、Q分類、実NOTAM2件（多角形採用・ずれ検出・時刻・分類）、APIモード（トークン・窓計算・408再試行・失敗時に meta 据え置き・認証エラー・認証情報なし・24h超ギャップ・bootstrap il(gz)・il解析不能・locations・dry-run・出力に秘密が混入しない）。
- **2026-09-24、staging環境の実データで検証済み**: probe実行(delta 164件)、`--fixture`での通し確認(40件)、`--bootstrap il`での初回投入(1,871件抽出)。いずれも正常終了。
- 開発中に見つかった実バグ: 初期ロードがJSONでない場合に例外で落ちる → 説明付きエラーに修正済み。
- **実データ検証で見つかった実バグ（修正済み）**: APIが座標未設定を`Point[0,0]`で返すケース（複数FIRにまたがるトリガーNOTAM等）を実座標として誤採用していた。`build_geometry()`を修正し、回帰テスト`test_zero_zero_point_rejected`を追加。

## 7. 設計判断とその理由（要点）

1. **クライアント側で絞る**: API にFIR/国フィルタが無いため、全世界の差分→中国系のみ抽出。差分が大きすぎる(408)場合は bootstrap で補う。
2. **4文字トークン判定**: 米ARTCC(3文字)との衝突回避＋A)複数地点対応。
3. **E項多角形を最優先**: Q項の中心は誤りうる（実測138NMずれ）。
4. **JSON+GeoJSONの二本立てで状態を持つ**: 真実の源は `state_active.json`（レコード）。`notam_cn.geojson` は派生物として毎回再生成。
5. **失敗は赤く**: 取得失敗は終了コード2、`last_success` 不進行。
6. **コミット抑制**: 変化なしなら書かない・meta は間引く。それでも変化があれば都度コミットされる（中国側の更新頻度次第で1日数十〜百コミットの可能性。増えすぎるなら data ブランチや Pages への分離を検討）。
7. **データ量**: NOTAM は年間400万件超発行される（FAA の説明。対象範囲の内訳は未確認）。中国分に絞れば有効側は小さいが、Archive は月別ファイルで増え続ける。リポジトリ肥大は運用しながら要監視。
8. **プッシュ配信(SWIM)は採用しない**: 常時接続が必要で cron 方式と相性が悪い。

---

## 8. 未確認・未確定事項

**2026-09-24、staging環境の実データ（probe + `--bootstrap il` 1,717件）で以下が確定した。**

| # | 事項 | 結果 |
|---|---|---|
| 1 | 認証情報の環境（fit / staging / prod） | **staging と確定**(`report.json`の`host`が`api-staging.cgifederal-aim.com`)。**prod での確認はまだ**。本番相当のデータかは未確認 |
| 2 | 中国系NOTAMが実際に返るか、件数、`classification` | 返る。直近差分(23h窓)で164件(CN 125 / TW 38 / HK 1)、**全件`classification: INTL`**。`--bootstrap il`(`classification=INTERNATIONAL`)でも中国系1,871件取得でき、**中国分はINTERNATIONALに含まれることを確認** |
| 3 | 複数地点A)の`icaoLocation`への入り方／`ZXXX` | 2回の実データ(probe 40件、il 1,871件)とも**該当ゼロ**。空白分割ロジックは無害だが未検証のまま |
| 4 | APIの`geometry`が面を含むか | **含まない**。probe(164件)・il(1,717件)とも `geometry_source: api`(面)は0件。E項テキストのパースが唯一の面情報源、という設計判断は正しかった |
| 5 | 差分の窓の上限（408） | 23hは通ることを確認(11秒)。**24hちょうどまで通るかは未検証**(probeは成功したら打ち切る仕様のため) |
| 6 | `--bootstrap il`のファイル構造・中国分がINTERNATIONALに入るか | **成功**。GEOJSON→gz解凍→JSONのパースは実データで問題なく動作。全世界45,504件中、中国系1,871件抽出 |
| 7 | 取消・置換の返り方、`cancelationDate`の意味 | 本文参照(`NOTAMC G4294/26`)からの`ref_number`抽出は実データで機能を確認。加えて元NOTAM側に`cancelationDate`が別途入る実例も確認（`effectiveEnd`とは異なる値）が、**発生タイミングの正確な意味はなお不確か** |
| 8 | `accountId`/`locationseries`によるサーバ側絞り込み | 差分取得では`accountId`(`ZBBBYNYX`等)が入り、`locationseries_cn_accountIds`も少数に集約（絞り込みに使えそう）。**ただし`--bootstrap il`側では`account_id`が全1,717件で`None`**（il固有。原因未確認＝ilのレスポンス自体に無いのか、収集側のパース漏れか）。`accountability`パラメータ自体はまだ未使用 |
| 9 | QコードのR/W分類・各コードの意味をICAO公式表で確認 | 未着手 |
| 10 | NMSデータの再配布条件（GitHub公開の可否） | 未着手。FAAへの照会が必要 |
| 11 | レート制限・利用条件（個人・非商用の可否） | 未確認 |
| 12 | 国内向けのみのNOTAM（中国）が見えるか | 中国系はprobe・ilとも**全件INTL**で、DOMESTIC分類は1件も出現せず。**§2のリスク1（国際配信分のみ見えている可能性）を裏付ける結果**。国内向けNOTAMが別途存在するとしても、このAPI経由では見えない可能性が高い |

**新たに判明した事項（実データで気づいたもの）**

| 事項 | 内容 |
|---|---|
| **座標(0,0)バグ（修正済み）** | APIが座標未設定を`Point[0,0]`で返すケース（複数FIRにまたがるトリガーNOTAM等）を、収集スクリプトが実座標として誤採用していた。probe 40件中9〜11件、il 1,717件中32件で発生。`build_geometry()`を修正し、`(0,0)`は`no_geometry`扱いに変更。回帰テスト追加済み（§6） |
| 台湾(TW)がCNより件数が多い理由 | `--bootstrap il`結果でTW 901件・CN 781件。中身を見ると`RCAA`(台北FIR)だけで356件。主因は**`QWLLW`(無人気球打ち上げ)の予告NOTAMが1ヶ月ごとに別NOTAM番号で反復発行**されているため。台湾側の発行慣行による水増しで、実際の事象数の多寡を意味しない |
| `ping_http: 400` | probeの`/v1/ping`が400を返す。認証・`/v1/notams`本体は正常に通るため致命的ではなさそうだが、原因不明のまま残る |
| 地点別個別問い合わせの件数感 | `per_location`は地点あたり数十〜300件超（`RCAA`375件等）。`--bootstrap locations`方式も現実的な選択肢と確認 |

---

## 9. 次のステップ（推奨順）

1. **残る未確認事項の確認**（§8の#9〜11）: Qコードの公式確認（ICAO Doc 8126）、NMSデータの再配布条件（FAA `NOTAMS@faa.gov` へ照会）、レート制限。再配布条件が分かるまでは **private リポジトリ運用を推奨**（§2）。
2. **prod環境かどうかの確認**: 今回(2026-09-24)検証したのは staging。`NMS_ENV=prod` で probe を再実行し、認証できるか・データが変わるかを確認。
3. **`account_id`がilで`None`になる件の追加調査**（任意）: 気になるようなら `--bootstrap locations` と比較するか、ilの生レスポンスを一度保存して構造を見る。
4. **初回投入**: `python notam_cn_collect.py --bootstrap il` → 出力(`notam_out/`)を確認してコミット。※2026-09-24にstagingで動作確認済み（1,871件抽出・1,717件をgeojson化）。
5. **collect ワークフローを手動実行**して赤/緑とコミットを確認 → cron-job.org に10分おきで登録。
6. **ビューア改修**（§10）。地点集計の気泡地図プロトタイプで表示イメージは確認済み（§14）。

## 10. ビューア(`geoplot-mil.html`)統合 — 2026-09-24 実装済み（v1.9.0）

MSA側バンドル(`geoplot-mil_bundle_2026-09-20.zip`)の `geoplot-mil.html` に、NOTAMレイヤーを追加した。
MSA(`warningsByUrl`/`msaLayer`)とは完全に別のデータモデル・別レイヤー(`notamLayer`)・別UIで、
`enterMode()` のモード切替(MAP自動更新/MAP手動)の対象外にしてある — **どちらのモードでも
ヘッダーの🛰ボタンからNOTAMパネルを開閉でき、地図上のNOTAM図形も常時表示され続ける**（ユーザー
要望「MAP(自動更新) MAP(手動) タブにNOTAMを追加」への対応）。

**実装したもの**:
- ヘッダーに🛰ボタン(未読件数バッジ付き)→クリックで開くパネル(`#notamPanel`、`.notif-panel`と
  同じ「ヘッダーボタン→ドロップダウン」の作り)。
- 接続: `notam_out/notam_cn.geojson` へのURL接続(10分ごと自動再読込、`notam-collect.yml`の
  収集間隔に合わせた)。MSA側が`?geojson=`で自動接続していれば、`/msa_out/`→`/notam_out/`の
  置換で `notam_cn.geojson` のURLを自動推測して接続する(`?notam=`で明示上書きも可)。
- タブ: Active / Upcoming / Archive / All。Archiveは年月(`<input type="month">`)を選んで
  `archive/notam_YYYY-MM.geojson` を都度読み込み(読み込んだ月ぶん`notamArchiveFeatures`に蓄積)。
- 絞り込み: 地域(CN/HK/MO/TW、既定全ON)と `focus_tag`(restriction/flag/watch/leisure/
  plaintext/admin。**既定でON: restriction/flag/watch、既定でOFF: leisure/plaintext/admin**
  — 実データでの分布・目的別分類の検討に基づく初期値)。
- 地図描画: GeoJSONをそのまま`L.geoJSON()`に渡すだけ(MSA側のような独自座標パースは不要、
  サーバー側で完成済みのため)。`geometry_source`が`qline-*`(Q項からの推定図形)のものは
  破線で描き分け。色は`focus_tag`ごと(restriction=赤/flag=琥珀/watch=シアン/その他=グレー)。
  Archiveは既定では地図に出さない(`ArchiveもMAPに表示`チェックボックスで任意にON)。
- ポップアップ: タイトル・地域・ICAO地点・Qコード・focus_tagバッジ・有効期間・(Q項推定図形
  なら)中心ずれ注記・本文全文。

**未着手・今後の検討事項**:
- Shapefile/PDF出力へのNOTAM反映（MSA側の警報と同様、今回は対象外のまま）。
- Archiveの月選択は都度1か月ずつ手動読み込み。複数月をまたいだ一括読み込みUIは無し。
- 動作確認はNode.jsでのJS構文チェックと、実データ(`--fixture samples/cn_sample.json`)で
  再生成した`notam_cn.geojson`のプロパティ形とJS側の参照キーの突き合わせまで。実ブラウザでの
  表示確認はまだ行っていない。

---

## 11. 既知の制約・残課題

- E項のパースは**座標列の多角形のみ**。「半径◯KMの円」「弧」「扇形」「回廊(線)」などの記述は未対応（1点だけの記述は面にせず、Q項の円/点にフォールバック）。
- 座標書式は `N/S…E/W…`（度分秒 or 度分）のみ。`DDMMSS N` 形式など別書式は未対応。
- 複数エリアの列挙は「先頭点に戻ったらリングを閉じる」規則。閉じない列挙は1つのリングになる。
- 反子午線(±180°)をまたぐ図形は考慮していない（対象地域では不要）。
- 高度は文字列のまま保持（`lower/upper`, `min_fl/max_fl`）。E項の「HEIGHT: 35,000M」等は解析していない。
- 開始・終了時刻はAPIの `effectiveStart/End` に依存。`schedule`(日次スケジュール)は `valid_raw` に文字列で入れるのみ。
- 種別 N/R/C は本文の `NOTAMN/R/C` を優先（合成データではAPIの `type` と本文が食い違うことがあった）。
- 対象判定は地名指標の接頭辞ベース。中国が管轄する空域を他国のコードで示すNOTAMは拾えない。

---

- **`--bootstrap il`では`account_id`が取得できない**（全件`None`）。差分取得では正常に入る。原因未確認（§8-8）。accountId前提の機能（accountabilityフィルタ等）は差分取得側のみで使うこと。
- 台湾(TW)の件数がFIR単位(`RCAA`)の反復発行(無人気球予告等)で大きく水増しされる。件数を「事象の多さ」と解釈しないよう注意（§8参照）。

## 12. 注意事項（重要）

- **認証情報**: `NMS-API-Pre-Prod-soapui-project_sample.xml`（ユーザー提供）に、**OAuthのクライアントID・シークレット・アクセストークンが平文で入っていた**。ユーザー自身の認証情報なら、このファイルをコミット・共有せず、FAAに再発行を相談すること。共通のサンプル値なら影響は小さい。本バンドルには含めていない。KEY/SECRET は今後もチャットに貼らず、**GitHub Secrets** に入れる。
- **public公開の可否は未確認**（§2, §8-10）。
- 本バンドルに入れていない再提供物（次のセッションで必要なら再アップロード）: `nms-api.yaml`(1.0.18・最新)、`FAQ_NMS-API.pdf`、`nms-api_curl_examples.txt`、サンプルXML/JSON。`nms-api-1_0_17.yaml` は旧版なので不要。SoapUI サンプルは上記の理由で再アップロードしないこと。
- MSA側のバンドル(`geoplot-mil_bundle_2026-09-20.zip`: `HANDOFF.md`, `geoplot-mil.html` v1.8.0, `msa_scraper.py`, `commit_state.py`, `state.json`, `military.geojson`)は別管理。NOTAM機能はMSA側のファイルを変更していない。

---

## 13. 新しいセッションでの再開手順

1. 本バンドル(zip)と、MSA側バンドルをアップロード。あれば `samples/`（§14の実データサンプル）も。
2. 「`HANDOFF_NOTAM.md` を読んで引き継いで」と伝える。
3. §9の次のステップ（1〜3）から進める。
4. 変更後は `python dev/test_notam_cn_collect.py` で32件（以降は追加分）が通ることを確認。

---

## 14. 2026-09-24 セッションログ（実データ検証）

このセッションで行ったこと。今後の参考・再現用に記録する。

1. **probe をstaging環境で実行**（ユーザーがGitHub Actionsで実行）。結果(`report.json`/`cn_sample.json`/`locationseries_cn.json`)を共有してもらい、§8の#1〜5・8・12を確定。
2. **`notam_cn_collect.py --fixture cn_sample.json`** で通し確認。冪等性・Archive振り分け・取消リンク(`ref_number`)の抽出が実データで動作することを確認。
3. **地図プロトタイプ**（Visualizerで作成・保存はしていない。チャット上で確認のみ）: D3 + world-atlas(110m)の基盤地図に、実データの座標を点/円としてプロット。ホバーで詳細表示。
4. **`--bootstrap il --out-dir ./notam_out_local`** をユーザーの手元で実行。全世界45,504件から中国系1,871件を抽出、1,717件をgeojson化。結果(`notam_cn.geojson`)を共有してもらい、§8の#6を確定。
5. **バグ発見・修正**: `notam_cn.geojson`を分析中、`geometry_source: api-point`のうち座標(0,0)のものが多数(40件サンプルで11件、1,717件中32件)見つかった。原因はAPIが座標未設定を`Point[0,0]`で返し、`build_geometry()`がそれを実座標として採用していたこと。修正し、回帰テスト`test_zero_zero_point_rejected`を追加(§6)。
6. **台湾(TW)901件の内訳確認**: `RCAA`(台北FIR)だけで356件、うち大半が無人気球打ち上げの月次反復NOTAM(`QWLLW`)と判明（§8参照）。
7. **地点集計の気泡地図**（120→114地点、座標(0,0)分は除外）を再度Visualizerで作成し、規模感を確認。
8. **成果物をzipにまとめて本セッションを終了**（このファイルを含む）。

**このセッションで得た実データサンプル**（`samples/`に同梱。再配布条件は未確認のため取り扱い注意・§8-10）
- `samples/report.json` — probe実行結果のサマリ（staging、23h窓、delta_cn_total=164）
- `samples/cn_sample.json` — probeで取得した中国系NOTAM生データ40件
- `samples/locationseries_cn.json` — locationseriesの中国系地点一覧
- `samples/notam_cn_il_bootstrap.geojson` — `--bootstrap il`の出力(1,717件、**修正前のバグ入り**。座標(0,0)の32件が混入したままなので、再利用する際は修正後の収集スクリプトに通し直すこと)
