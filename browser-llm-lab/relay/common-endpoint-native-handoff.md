# Common の署名付き公開接続先と Web discovery の接続手順

更新日: 2026-10-03。対象 Web は `https://ai-test.make-cph-great-again.community/`。

統合前の専用Commonの切替・公開反映・31分試験・未実施項目は [discovery-update.md](discovery-update.md) に記録した。この文書後半の単体試験結果と区別する。

Common 側の署名・配布と Web 側の取得・検証を実装した。以下は、両ディレクトリで作業を引き継ぐための実装契約と運用手順である。稼働切替や異なる地域・回線での試験は、ソースの実装・ローカル試験とは別に記録する。

## 1. 現在の実装範囲

| 対象 | 実装 |
| --- | --- |
| `/root/cypher` | 既存 native node key による公開 gateway origin の署名、同じ mesh boot ID、増加 sequence、30 秒更新、owner 限定 HTTP API |
| `/root/cypher/browser-llm-lab` | 正規の署名付き Common 広告・公開接続先の暗号検証、RAM 候補 cache、期限・巻戻し検査、遠隔 HTTPS/WSS 接続、複数 rendezvous によるブラウザ紹介・署名付き signaling |
| Web gateway | ローカル Common の署名 envelope を取得し、自分の directory へ格納。設定された bootstrap directory へ同じ envelope を自動広告 |
| native 認証 | 既存の Common 役割検査、ReservedNodes、network/genesis 確認、RLPx 認証を継続 |

Common は採掘・合意形成の鍵や native 秘密鍵をブラウザへ渡さない。ブラウザの自己署名レコードはブラウザの一時 identity を証明する。レコード内の `nodeId` だけで、そのブラウザと Common の所属関係が証明されるわけではない。

通常の Common P2P 参加に、この Web への登録は必要ない。ブラウザ用公開接続口を提供する運用者は、自分の Common と同じ所有者・同じホストの gateway を接続し、TLS と公開 origin を準備する。遠隔の Common の公開鍵を中央管理者が一台ずつ allowlist へ追加する運用は不要となる。

## 2. ネイティブ設定

統合後の通常接続元は `common-mine`。設定は `/root/cypher/config/browser-relay/common-mine.json`、解決後の owner socket は `/root/cypher/config/browser-relay/common-mine.sock`、通常の Linux binary は `/root/cypher/build/bin/cypher-linux-amd64` を使う。`mesh.publicGatewayOrigin` は公開サイトの HTTPS origin と一致させる。既存 `chaindbmine` と native identity を維持し、A/B の DB を合体・上書きしない。Web の `relay/start-managed.sh` は gateway と TURN のみを管理し、通常 Common は `/root/cypher/start-cyphermine.sh` 側で別途管理する。新配置・通常 Common での LIVE 検証結果は、以前の専用 A/B の結果と分けて記録する。

既存の owner 限定設定へ、任意の `mesh.publicGatewayOrigin` を追加した。

```json
{
  "enabled": true,
  "socketPath": "/var/lib/cypher-browser-relay/common-region-x/source.sock",
  "network": {
    "chainId": 10101919,
    "genesisHash": "0x001c8239f25a697933e2a54511a576205fb21cbb80dc974adb29894dc80250ad"
  },
  "sourceId": "common-region-x",
  "mesh": {
    "allowedOrigins": ["https://relay.example.org"],
    "publicGatewayOrigin": "https://relay.example.org"
  }
}
```

上記 genesis は研究ネットワークの値である。`relay.example.org` と socket path は配置例であり、現在稼働する接続先ではない。

- `publicGatewayOrigin` を省略した旧設定は、従来の mesh を継続し、`GET /relay/v1/mesh/endpoint` は 404 となる。追加機能を有効にするための別鍵は不要。
- origin は小文字 host の正規 HTTPS origin。末尾 `/`、path、query、fragment、資格情報、明示的な既定 port `:443`、port 0、private/reserved IP、localhost・`.local` などを拒否する。
- `sourceId` は gateway のローカル接続先 ID と一致させる。公開接続先を有効にする場合は `[A-Za-z0-9][A-Za-z0-9_-]{0,63}`。
- `allowedOrigins` は private API に到着する gateway の `nativeOrigin` に合わせる。実際の Web フロントエンドの許可 origin は、公開 gateway の `discovery.allowedOrigins` で別途設定する。
- owner 限定 config/socket、既存の Common 役割検査、datadir、鍵の所有権、要求数・接続数上限は維持する。既存 DB を別プロセスで重複して開かない。

ノードはこの URL に対して DNS 解決や HTTP 接続を行わない。自分の公開接続先という claim に署名する。公開接続口の生存や、その WSS が同じ Common へ到達することは、Web 側の接続と native HELLO の一致で別途確認する。

## 3. 署名対象の固定契約

payload は次の **9 フィールドだけ** を持つ厳密 JSON とする。未知フィールド、重複した decoded key、null、不正整数を受け付けない。

```json
{
  "version": 1,
  "network": {"chainId": 10101919, "genesisHash": "0x...64桁の小文字hex..."},
  "enode": "enode://...既存native公開鍵...@...数値IP...:port",
  "sourceId": "common-region-x",
  "gatewayOrigin": "https://relay.example.org",
  "bootId": "32桁の小文字hex",
  "sequence": 1,
  "issuedAt": 1791020000000,
  "expiresAt": 1791020120000
}
```

上の読みやすい表示のフィールド数は、`network` を一つと数えて 9 個である。native の生成順は表示順と同じ。受信側は key の並び替えや JSON の再生成を行わず、envelope に入った元の UTF-8 bytes で検証する。

```text
domain = UTF8("cypher-browser-mesh-endpoint-v1\0")
digest = Keccak256(domain || exactPayloadBytes)
signature = crypto.Sign(digest, existingNativeNodeKey)
```

末尾 `\0` は文字列 `\\0` ではなく、1 byte の NUL。SHA3-256 と Keccak-256 は異なるため、Keccak-256 を使用する。

```json
{"payloadBase64":"...","signatureHex":"..."}
```

- payload は最大 4096 bytes、envelope は最大 8192 bytes。
- base64 は標準の canonical base64。
- `signatureHex` は 65 bytes の `R || S || V` を表す小文字 hex。V は 0 または 1、S は low-S。
- identity は enode の非圧縮公開鍵の prefix を除いた `X || Y` 64 bytes。`nodeId = hex(Keccak256(X || Y))`。
- `version` は 1、`sequence` は 1 以上かつ 9007199254740991 以下の整数。文字列形式の sequence は受け付けない。
- `issuedAt` / `expiresAt` は Unix epoch milliseconds の整数。TTL は最大 120000 ms。検証時刻より 10000 ms を超えて未来の issuedAt を拒否する。
- `bootId` は **既存 mesh 広告と同じ** `Mesh.bootID`。32 桁の小文字 hex。プロセス再起動で更新する。

既存の `cypher-browser-mesh/1` 広告にはフィールドを追加しない。既存 domain `cypher-browser-mesh-advertisement-v1\0` と新 endpoint domain を区別する。WSS/frame/RLPx の契約も維持する。

## 4. 生成・更新・停止

1. Common の mesh 開始時に、現在の native identity と既存 signer から最初の descriptor を作成する。sequence は 1。
2. 既存 mesh maintenance を利用し、30 秒を目安に再生成する。署名・自己検証の成功時だけ、envelope、sequence、発行時刻をまとめて更新する。
3. GET のたびに署名や sequence 増加を行わない。同じ世代の GET は同じ envelope を返す。
4. 署名失敗や identity 不在では古い bytes の期限を延長しない。最後の成功から 120 秒で提供を停止する。
5. clock の巻戻しや sequence の上限到達では新しい descriptor を生成しない。停止後に GET で復活しない。

Web の RAM cache は同じ boot の sequence 巻戻しと同 sequence の内容変更を拒否する。新 boot は前の descriptor より新しい issuedAt が必要。直前の boot の再送を拒否するため、期限付きの履歴を一時保持する。上限は 64 identities、各 identity の retired boot は最大 8。生きた候補を無条件に追い出して新規 identity flood を受け入れない。

## 5. private API と gateway の取得

```text
GET /relay/v1/mesh/endpoint
Origin: <gateway.nativeOrigin>
body: なし
```

| 状態 | 応答 |
| --- | --- |
| 設定済み・有効期限内 | 200 と元の `{payloadBase64, signatureHex}` |
| 公開接続先を未設定 | 404 |
| mesh 停止、期限切れ、利用不能 | 503 |
| Origin 不一致 | 403 |
| body 付き GET | 400 |
| GET 以外 | 404 |
| 既存 mesh 要求数上限超過 | 429 |

`Cache-Control: no-store` を維持する。GET は browser session、circuit、native peer を作成しない。owner 限定 socket の権限を緩めず、RPC/IPC や任意 TCP 接続を公開する機能も追加しない。

Web gateway はこの API を 30 秒ごと、最大 2 並列で読み、署名・network・ローカル nodeId・sourceId・gateway origin を確認する。envelope を再署名しない。ローカル node の enode の IP 表記が更新されても、公開鍵 identity が同じなら扱える。古い native の 404 は、既存のローカル mesh 接続を停止させない。

## 6. 各地域 gateway の起動準備と自動広告

各地域の gateway は、自分が所有する Common の socket へのローカル接続情報を持つ。これは、遠隔の全 Common を一つの中央一覧へ登録することとは別の配線である。

gateway の既存設定へ、次を追加する。

```json
{
  "discovery": {
    "enabled": true,
    "bootstrapOrigins": ["https://bootstrap.example.org"],
    "allowedOrigins": ["https://ai-test.make-cph-great-again.community"]
  }
}
```

実際には既存 gateway config の他のフィールドを保持する。`origin` は当該地域の公開 origin、`nativeOrigin` は Common の `mesh.allowedOrigins` に含まれる origin とする。`bootstrapOrigins` は operator が選ぶ出会いの場所であり、Common 公開鍵の信頼一覧ではない。

公開 API は次のとおり。

```text
GET  /relay/v1/mesh/discovery
POST /relay/v1/mesh/discovery
WSS  /relay/v1/mesh/rendezvous
```

GET は `{version:1, network, endpoints:[署名envelope...], peers:[署名browserrecord...]}`。POST body は `{"endpoint":署名envelope}`。成功時は `202 {nodeId, expiresAt}`。

gateway はローカル Common の有効 descriptor を取得すると、設定済み bootstrap origin へ同じ POST を自動送信する。最大 2 並列、同じ相手への送信は最短 1 秒間隔、同じ endpoint の再広告は 30 秒間隔、要求 timeout は 2 秒、応答読取上限は 4 KiB。外部 directory から受け取った任意 URL をサーバー側の接続先へ転用しない。公開投稿側にも署名・TTL・Origin 形式・サイズ・速度・cache 上限がある。

ブラウザは Node ON で directory を読み、自己検証した Common へ接続する。ブラウザ紹介は最大 4 rendezvous 接続、Common/ブラウザ実接続上限は各 20。OFF・hidden・page 終了の世代管理と一時情報破棄を維持する。

公開 DNS、TLS、proxy の固定 API allowlist、CORS、WebSocket upgrade を、各地域 gateway に用意する必要がある。これらはソフトウェアが新しい Common を自動発見できることとは別の運用条件である。

## 7. 固定相互運用ベクター

Native の固定 vector:

```text
/root/cypher/node/browserrelay/testdata/mesh-endpoint-vector.json
```

公開されたテスト scalar 1 を使用する。実運用鍵として使わない。ファイルには exact UTF-8、envelope、nodeId、digest を収録する。

この vector の検証時刻は `1700000000000`、fixture genesis は `0x` と `22` の 32 回連結である。研究ネットワークの genesis とは区別する。現在時刻で vector の期限切れが拒否されることは正常であり、固定時刻を渡した相互運用テストで比較する。

```text
nodeId:
c0a6c424ac7157ae408398df7e5f4552091a69125d5dfcb7b8c2659029395bdf

Keccak digest:
813a1a293c43e353a78e5663aecea9cddcd97fb037b8806a95cdd083502c2226

signatureHex:
66f0bd687a5707851429158b2eb08d898e1075482ea54dd2f8fd8cbfa6295136268da3124b7d3856e0b394b82102f2f8f12ae668dea531eee8ee1c4c5f60710a01
```

Go の `meshSignEndpoint` がこの envelope と完全一致すること、ブラウザの noble 実装が同じ署名を生成・検証できることを、それぞれ別のテストで確認する。

## 8. 今回の検証結果と再実行

この文書の native 実装時点で確認した項目:

| 試験 | 結果 |
| --- | --- |
| Web の crypto/protocol テスト | 29 件 PASS |
| Go endpoint の domain/署名/identity/TTL/改ざん/重複 key/固定 vector | PASS |
| native endpoint の開始・更新・署名失敗・期限切れ・停止・新 boot | PASS |
| owner HTTP の 200/404/403/400/停止・セッション非生成 | PASS |
| optional config、旧設定、null/重複 key/危険 origin | PASS |
| 既存 `make cypher` の BLS・LevelDB・mesh/cmd/p2p テストと Linux build | PASS |
| `go test -race ./node/browserrelay ./cmd/cypher ./p2p`（既存 fork/build tag 適用） | 3 packages PASS |
| 本番へのバイナリー切替・実 Common の新 endpoint 取得 | 親作業で別途管理。最新の配信・検証記録を参照 |
| 異なる実端末・地域・回線・TURN・モバイル・30 分超 | この追加機能の native 単体試験では未実施 |

Web 側:

```sh
cd /root/cypher/browser-llm-lab
npm ci
npm run build:mesh-crypto
node --test tests/mesh-discovery.test.mjs tests/mesh-protocol.test.mjs
```

Native の通常 build は、既存 helper がチェックサム固定の bounded-storage fork と `cypher_bounded_storage` を適用する。今回と同じく稼働バイナリーを直接上書きしない場合:

```sh
cd /root/cypher
make cypher \
  BINDIR=/tmp/cypher-endpoint-build/bin \
  STAGE_ROOT=/tmp/cypher-endpoint-build/stage \
  JOBS=4
```

今回の隔離 build:

```text
/tmp/cypher-mesh-discovery-20261003/native-endpoint/bin/cypher
SHA-256: be38686525295758caecfad9e76ddda4fc9579e3521b3651662fbf464e07052c
```

この `/tmp` は試験成果物の位置であり、実行時の必須接続先ではない。`build/bin` の既存バイナリーは、この隔離 build では変更していない。

build では既存の Herumi / Duktape C 警告が出たが、必須テストと build は完了した。race 試験も helper が準備した同じ BLS ヘッダー・ライブラリーと bounded fork を使用した。元の `go.mod` / `go.sum` を書き換えて fork を固定する方法は採らない。

## 9. 次の Codex セッションへ渡す作業事項

> `/root/cypher` と `/root/cypher/browser-llm-lab` の現行作業ツリーを確認してください。Common の endpoint 署名機能は `node/browserrelay/mesh_protocol.go` / `mesh.go`、設定と private API は `cmd/cypher/browser_mesh.go` / `browser_public_relay.go` に追加済みです。`mesh.publicGatewayOrigin`、上記の固定 domain / envelope / 時刻 / sequence / bootId 契約を維持してください。既存の未コミット変更を保存し、署名機能を別実装で作り直さないでください。
>
> 各地域の Common と owner gateway を接続し、native `GET /relay/v1/mesh/endpoint` → gateway directory → bootstrap への自動広告 → ブラウザでの署名検証 → 同じ identity / boot の native HELLO までを確認してください。private socket や RPC/IPC を無制限公開せず、TLS / Origin / request / connection / bytes 制限を維持してください。
>
> 実機・異なる回線の試験は、同一ホストのテスト結果と分けて記録してください。バイナリー切替は対象 Common、datadir、起動方法、旧バイナリーを確認して行い、別 Common や委員会ノードを巻き込まないでください。古い descriptor の再生、Common 再起動、endpoint 失効、browser OFF / background と再参加で、古い session や circuit が残らないことを検証してください。
