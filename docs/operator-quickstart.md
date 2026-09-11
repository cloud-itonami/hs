# Operator quickstart

**ここに載っているコマンドは、2026-09-03 にこの repo の checkout に対して実際に
実行したものだけである。** 出力は貼った時点の実測値。踏めなかった手順は載せて
いない（`actor-manifest.test.ts` の vitest、manifest が宣言する `kubectl` /
graph クエリは、対象がこの repo に無いので**ここでは踏めない**。理由は
[README](../README.md) の「この repo に**無い**もの」を参照）。

所要は 5 分弱。JVM を起こすのは 2. と 3. だけで、**4. は nbb（Node）で走る**ので
`clojure` が無くても gate の検証はできる。

## 0. 前提

| 道具 | 用途 | 確認 |
|---|---|---|
| `git` | 1. clone | `git --version` |
| `clojure` | 2. test / 3. lint | `clojure --version` |
| `nbb` | 4. gate 検証（JVM 不要） | `nbb --version` |
| `jq` | 5. manifest 検査 | `jq --version` |
| `curl` / `dig` | 6. DID 検証 | 標準 |

`clojure` と `nbb` は独立している。**片方しか無くても 1〜6 のうち踏める分は踏める。**

## 1. clone

```bash
git clone git@github.com:cloud-itonami/hs.git
cd hs
```

この workspace の west checkout から使う場合は `orgs/cloud-itonami/hs`
（**remote 名は `origin` ではなく `cloud-itonami`**）。

## 2. テストを走らせる

```bash
kbb -M:test
```

```
Running tests in #{"test"}

Testing hs.murakumo-test

Ran 9 tests containing 213 assertions.
0 failures, 0 errors.
```

初回は依存（cognitect test-runner）の解決で 1〜2 分かかる。2 回目以降は数秒。

テストは `cell-specs` を**内省**して回るので、セルを足し引きしても書き換えずに
効く（15 セル × 契約で 213 assertion になっている）。

## 3. lint

```bash
kbb -M:lint
```

```
src/hs/murakumo.cljk:159:14: warning: unused binding input
linting took <N>ms, errors: 0, warnings: 1
```

（所要 ms は貼っていない —— このマシンでは並行セッションの負荷で大きく振れる。
見るのは `errors: 0, warnings: 1` の方。）

**warning 1 件は既知で、exit は 0 である**（`:lint` alias は `--fail-level error`）。
`records-for` が `:as input` を束縛して使っていない。**exit 1 になったら warning
ではなく error が増えている**ので、その差分を見ること。

## 4. gate が閉まることを自分で確かめる（JVM 不要）

この repo の主張は「attestation が揃わなければ effect を 1 つも出さない」である。
**それを信じずに、その場で両方向を出す。**

```bash
kbb --backend sci --classpath src -e '
(ns probe (:require [hs.murakumo :as m]))
(let [all (into {} (map (fn [g] [g true]) m/common-gates))
      one-short (dissoc all (first m/common-gates))]
  (println "cells declared:" (count m/cell-specs))
  (println "gates required:" (count m/common-gates))
  (println "no attestation   ->" (:status (m/cell-plan :health {}))
           "effects:" (count (:effects (m/cell-plan :health {}))))
  (println "one gate missing ->" (:status (m/cell-plan :health {:attestations one-short}))
           "effects:" (count (:effects (m/cell-plan :health {:attestations one-short}))))
  (println "all 7 attested   ->" (:status (m/cell-plan :health {:attestations all :request-id "req-1"}))
           "effects:" (count (:effects (m/cell-plan :health {:attestations all :request-id "req-1"}))))
  (println "fleet-wide, zero attestation -> total effects:"
           (reduce + (map #(count (:effects %)) (vals (m/all-cell-plans {}))))))'
```

```
cells declared: 15
gates required: 7
no attestation   -> :blocked effects: 0
one gate missing -> :blocked effects: 0
all 7 attested   -> :ready effects: 1
fleet-wide, zero attestation -> total effects: 0
```

**3 行目が `:ready` になり、1〜2 行目が `:blocked` になることの両方を見ること。**
`:blocked` しか出ない実行は、gate が閉まっている証拠ではなく、単に何も動いて
いない証拠かもしれない（classpath を間違えて別の名前空間を読んでいても 0 件は
出る）。**`cells declared: 15` が出ていることが、正しい名前空間を読んだ証拠**で、
**7 本目を 1 本抜いただけで止まる**ことが、この境界の主張そのもの。

`:ready` のときに返るのは effect の**記述**であって、書き込みではない:

```bash
kbb --backend sci --classpath src -e '
(ns probe (:require [hs.murakumo :as m]))
(let [all (into {} (map (fn [g] [g true]) m/common-gates))]
  (prn (first (:effects (m/cell-plan :health {:attestations all :request-id "req-1"})))))'
```

```
{:op :mst/put-record, :actor "did:web:hs.etzhayyim.com",
 :collection "com.etzhayyim.hs.health", :rkey "req-1", :record {...}}
```

`:actor` に刻まれている DID は**解決しない**（6. を参照）。

## 5. manifest を検査する（vitest は走らないので jq で）

`actor-manifest.test.ts` は vitest 前提だが、この repo に `package.json` は
無い。宣言の中身はそのまま引ける:

```bash
jq -r '"pipelines: \(.pipelines|length)  xrpc: \([.pipelines[]|select(.trigger.type=="xrpc")]|length)  cron: \([.pipelines[]|select(.trigger.type=="cron")]|length)  actors: \(.actors|length)"' actor-manifest.jsonld
jq -r '.pipelines[]|select(.trigger.type=="xrpc")|.trigger.nsid' actor-manifest.jsonld
```

```
pipelines: 6  xrpc: 5  cron: 1  actors: 4
com.etzhayyim.apps.hs.getNode
com.etzhayyim.apps.hs.getChildren
com.etzhayyim.apps.hs.resolveConcordance
com.etzhayyim.apps.hs.getPolicyOverlay
com.etzhayyim.apps.hs.health
```

宣言された `complianceDocs` が在るかを確かめる。**両方欠けている**のが
2026-09-03 の実測値である:

```bash
for p in $(jq -r '.complianceDocs[]' actor-manifest.jsonld); do
  printf '%-62s %s\n' "$p" "$(test -e "$p" && echo PRESENT || echo MISSING)"
done
```

```
90-docs/rules/compliance/per-did-kyumei-shinka-autonomy.md      MISSING
90-docs/260415-hs-code-domain-coverage-design.md                MISSING
```

`PRESENT` が出るようになったら README の「無いもの」節を測り直すこと。

## 6. DID を検証する

**3 つの DID が食い違っている**（[README](../README.md) の Identity 節）。
信じずに 3 つとも引く:

```bash
for u in "https://etzhayyim.com/actor/hs/did.json" \
         "https://hs.etzhayyim.com/.well-known/did.json" \
         "https://etzhayyim.github.io/com-etzhayyim-hs/did.json"; do
  printf '%-58s %s\n' "$u" "$(curl -sS -o /dev/null -w '%{http_code}' --max-time 20 "$u")"
done
dig +short hs.etzhayyim.com
```

```
https://etzhayyim.com/actor/hs/did.json                    200   <- did.json の id
https://hs.etzhayyim.com/.well-known/did.json              000   <- コードの actor-did
https://etzhayyim.github.io/com-etzhayyim-hs/did.json      404   <- 旧値（alsoKnownAs）
                                     <- hs.etzhayyim.com は NXDOMAIN（空行）
```

2 行目の `000` は `curl: (6) Could not resolve host`。**`src/hs/murakumo.cljk` の
`actor-did` と `actor-manifest.jsonld` の `@id` が指す先は実在しない。**
`dig` が**何も返さない**のが現在の期待値で、ここに IP が出るようになったら
README の Identity 節を測り直すこと。

**repo の写しを配信中の文書と混同しない。** 一致するのは `id` だけである:

```bash
curl -sS --max-time 20 "https://etzhayyim.com/actor/hs/did.json" -o /tmp/hs-served.json
diff <(jq -S . /tmp/hs-served.json) <(jq -S . .well-known/did.json)
```

`@context` / `alsoKnownAs`（配信は空、repo は 4 件）/ PDS エンドポイント
（配信 `pds.aozora.app`、repo `pds.etzhayyim.com`）/ サービス
（配信 `#xrpc-libp2p`、repo `#aozora`）が違う。**did:web の解決先は配信中の
文書**なので、identity を判断するときに repo のファイルを読まない。

## 踏めないもの（なぜ載っていないか）

| やりたいこと | ここで踏めない理由 |
|---|---|
| `npx vitest actor-manifest.test.ts` | `package.json` も vitest もこの repo に無い（→ 5. で代替） |
| manifest の `graph.query`（Cypher）を流す | `HsNode` / `HsConcordance` / `HsPolicyOverlay` のグラフがこの repo に無い |
| XRPC 5 本を叩く | `k8s-langserver` ランタイムがこの repo に無い |
| cron パイプライン（6 時間毎の coverage 報告）を回す | 同上。`agent.chat` / `derive:social` の実装も無い |
| HS コードを実際に分類する | **HS のデータがこの repo に 1 件も無い** |

これらは上流デプロイの手順である。**この repo を clone しただけのオペレータには
実行できない**ので、ここには書かない。
