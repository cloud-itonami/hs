# hs — HS（統計品目番号）分類アクターの、実行しない計画境界

**名乗り**: `hs` は **Harmonized System**（HS 条約に基づく国際統一商品分類、
いわゆる HS コード / 統計品目番号）の略で、機能を示さない 2 文字名である。
扱う主題は**貿易財の関税分類**——部・類・項・号（section / chapter / heading /
subheading）の階層、GTIN・CPC・ISIC との対応付け、国別の関税・規制オーバーレイ。

この repo が実際に持っているのは、そのアクターの
**純粋な `.cljc` 計画境界 1 本だけ** —— `hs.murakumo`（`src/hs/murakumo.cljk`）。

**この repo に HS コードのデータは 1 件も無い。分類を行うコードも無い。**
ネットワークに触る関数も、グラフに問い合わせる関数も、PDS に書く関数も 1 つも
無い。`hs.murakumo` は入力を受けて **effect の *記述*** を返す純関数の集まりで、
その記述を誰かが実行するかどうかは、この repo の外側の話である。依存は
`clojure.string` だけ。

生成されるレコードは自分でそう申告している —— `:scaffold true` /
`:constitutionalStatus "attested-plan"` / `:actorBoundary "cljc-migration-scaffold"`。
各セルの `:ceiling` も `"Manifest-driven migration scaffold; explicit execution
stays in runtime methods"` と書いている。

## この repo に在るもの（`git ls-files`、2026-09-03 実測）

| path | bytes | 何か |
|---|---:|---|
| `src/hs/murakumo.cljk` | 8,414 | 唯一の実装。下記の計画境界。15 セル |
| `test/hs/murakumo_test.cljk` | 4,181 | その契約テスト（9 tests / 213 assertions） |
| `actor-manifest.jsonld` | 6,544 | アクター identity + governance + パイプラインの宣言 |
| `actor-manifest.test.ts` | 2,025 | 上の manifest の vitest 検査。**この repo では走らない**（下記） |
| `deps.edn` | 398 | `:test`（cognitect test-runner）/ `:lint`（clj-kondo） |
| `.well-known/did.json` | 670 | DID 文書の**リポジトリ側の写し**。配信中のものとは違う（下記） |
| `NOTICE` | 513 | Apache-2.0 + etzhayyim Charter Rider v3.1 |

上の 7 つに `.gitignore`（`.cpcache/` の 1 行）と空の `.nojekyll` を足した
**9 ファイルが tracked な全部**である。**README はこれが最初の 1 枚。**

## 計画境界がしていること

`cell-plan` は、15 セルのいずれかについて「7 つの gate が全部 attest されて
いるか」を見て、揃っていなければ `:blocked` と `:effects []` を、揃っていれば
`:ready` と `:mst/put-record` の**記述**を返す。書き込みはしない。

```
no attestation   -> :blocked  effects: 0
one gate missing -> :blocked  effects: 0     ← 7 本中 1 本欠けただけで止まる
all 7 attested   -> :ready    effects: 1
```

7 gate は `common-gates`（`src/hs/murakumo.cljk`、キーワードの綴りはこのとおり）:
`:council-charter-attestation` / `:no-platform-held-key-baseline` /
`:no-probing-baseline` / `:murakumo-only-inference-baseline` /
`:did-primary-baseline` / `:append-only-gate-baseline` /
`:kotoba-only-substrate-baseline`。**両方向を自分で出す手順**は
[docs/operator-quickstart.md](docs/operator-quickstart.md) の 4.。

## この repo に**無い**もの —— manifest の読み方

`actor-manifest.jsonld` は 6 パイプライン（cron 1 + XRPC 5）、`k8s-langserver`
ランタイム、`sveltekit-proxy` エッジ、`HsNode` / `HsConcordance` /
`HsPolicyOverlay` に対する Cypher クエリを宣言している。**そのどれ 1 つとして
この repo には無い。** 宣言された `complianceDocs` も両方欠けている（2026-09-03 実測）:

```
90-docs/rules/compliance/per-did-kyumei-shinka-autonomy.md   MISSING
90-docs/260415-hs-code-domain-coverage-design.md             MISSING
```

manifest が記述しているのは上流デプロイ（`etzhayyimcojp/20-actors` から移行）の
姿であって、**この repo を clone した人が動かせるもの**ではない。ここに在るのは
そのアクターの計画境界だけである。

`actor-manifest.test.ts` も同じ理由で走らない —— `package.json` も vitest も
この repo に無いので、**clone しただけでは実行できない**（2026-09-03 実測）。
manifest の中身を検査したいなら `jq` で直接引く（quickstart の 5.）。

## Identity —— 3 つの DID が食い違っている（2026-09-03 実測）

**コードが名乗る DID は解決しない。** 3 箇所が 3 通りのことを言っている:

| 出所 | 値 | 解決 |
|---|---|---|
| 配信中の did.json | `did:web:etzhayyim.com:actor:hs` | **200** |
| repo の `.well-known/did.json` | `did:web:etzhayyim.com:actor:hs` | 同上（ただし中身が配信版と違う） |
| `actor-manifest.jsonld` の `@id` | `did:web:hs.etzhayyim.com` | **NXDOMAIN** |
| `src/hs/murakumo.cljk` の `actor-did` | `did:web:hs.etzhayyim.com` | **NXDOMAIN** |

`hs.etzhayyim.com` は DNS レコードが無い。にもかかわらず `actor-did` は
**計画される全レコードの `:actorDid` に刻まれる**。`at://hs.etzhayyim.com` は
repo 側 did.json の `alsoKnownAs` に在るが、それは AT ハンドルであって
did:web ではない。

さらに **repo の `.well-known/did.json` は配信中のものと別物**である。一致するのは
`id` だけで、`@context`（jws-2020 / ed25519-2020）・PDS エンドポイント
（`pds.aozora.app` / `pds.etzhayyim.com`）・サービス（`#xrpc-libp2p` / `#aozora`）・
`alsoKnownAs`（空 / 4 件）が食い違う。**did:web の解決先は配信中の文書**なので、
repo のファイルを読んで identity を判断しない。

経緯は git に在る: `f498741`（2026-07-02、ADR-2606231200 addendum）が did.json を
`did:web:etzhayyim.github.io:com-etzhayyim-hs` から現在値へ移したが、**manifest と
コードは移していない**。旧値は 404 で、`alsoKnownAs` に保存されている。

**この repo はこの食い違いを直していない** —— アクターの公開 DID を変えるのは
identity の決定であって、README を書く作業の範囲ではない。ここでは測って記録する。

## 走らせる

[docs/operator-quickstart.md](docs/operator-quickstart.md)。所要 5 分弱、
JVM を使わずに済む経路も用意してある。

```bash
clojure -M:test    # 9 tests / 213 assertions
clojure -M:lint    # errors: 0, warnings: 1（既知）
```

## ライセンス

Apache-2.0 + etzhayyim Charter Compliance Rider v3.1。`NOTICE` を参照。
Rider 本文（`CHARTER-RIDER.md`）は**この repo には無い** —— `NOTICE` が
参照しているのは上流の文書である。
