# SpinKube と wasmCloud v2 のローカル実測

2026-11-28 JST。Apple M1 Max の macOS 26.4 上で、Lima 2.3.0 の arm64 Linux VM（5 vCPU、8 GiB）に単一ノードの kind クラスタを作った。Kubernetes 1.35.0、kind 0.30.0、containerd 2.2.0。両方の operator と測定用クライアントは同じクラスタで動かした。

| 指標 | SpinKube | wasmCloud v2 |
| --- | ---: | ---: |
| 再デプロイから最初の正常応答、中央値 | 1.521 s | 0.755 s |
| 同、最小–最大（各 10 回） | 1.261–1.914 s | 0.449–1.071 s |
| 定常 HTTP、中央値 | 0.238 ms | 0.217 ms |
| 定常 HTTP、p95（各 3,000 リクエスト） | 0.384 ms | 0.338 ms |
| 定常 HTTP、最大 | 41.809 ms | 0.827 ms |
| 定常 HTTP、期待した本文での HTTP 200 | 3,000 / 3,000 | 3,000 / 3,000 |

この構成では、wasmCloud v2 の再デプロイ中央値は SpinKube の約半分だった。Kubernetes の Service IP が見えるまでの中央値は SpinKube が 146 ms、wasmCloud v2 が 149 ms。SpinKube は試行ごとに新しい Wasm Pod を作り、wasmCloud は稼働中の同じ host group の Pod に workload を配置した。定常 HTTP の差は中央値で約 0.021 ms、p95 で約 0.047 ms。SpinKube の各バッチで約 41 ms の最大値が観測され、原因は未特定。単一ホストでの小さな HTTP workload の結果で、Agent のターン、推論、ツール、永続化、署名、並列負荷は測定範囲外。

## 実行した構成

- **SpinKube**: Spin Operator 0.6.1、cert-manager 1.20.0、containerd-shim-spin 0.27.2。`SpinAppExecutor` を workload と同じ namespace に作成し、`SpinApp` を 1 replica、`wasmtime-spin-v2` RuntimeClass で起動した。Service は operator が作成した。[SpinKube Quickstart](https://www.spinkube.dev/docs/install/quickstart/)
- **wasmCloud v2**: runtime-operator と `wash` host が 2.12.1。bundled NATS 2.12.8-alpine と 1 replica の `default` host group を使用。旧 gateway を無効化し、`WorkloadDeployment` と `ClusterIP` Service を作成した。ホストには `wasi:http/incoming-handler` を許可し、operator が EndpointSlice を登録した。[wasmCloud のインストール](https://wasmcloud.com/docs/installation/) · [WorkloadDeployment](https://wasmcloud.com/docs/quickstart/deploy-a-webassembly-workload/)
- **同一の Wasm**: どちらも SHA-256 が `65adfc98b08cc017ce3fe4d86c8cf924838f5a098faca1052f223e460a435879` の 224,241 bytes。wasmCloud は [公式 hello-world OCI artifact](https://github.com/wasmCloud/wasmCloud/blob/v2.12.1/templates/http-hello-world/manifests/workloaddeployment.yaml) を digest 指定で使用（`sha256:68cfa8249badb038abc5b8c91fe86063f47c47f9d9cf4f11627aa16f8f917b9d`）。Spin はその Wasm を `spin registry push --compose false` で包んでローカル OCI registry に置き、レイヤー digest が元の Wasm と一致することを確認した（包みの digest `sha256:e8f816d4d3e7b6309d8afc9b830adc82a0f2bb5f877b7ce25bdd52fe77b04a1a`）。[kind のローカル registry 手順](https://kind.sigs.k8s.io/docs/user/local-registry/)

## 測定手順と試行値

1. Operator と wasmCloud host を Ready にし、両方で `GET /` が `Hello from wasmCloud!\n` を返すことを確認する。最初の確認では Service 作成前に名前解決を開始したため、DNS の失敗後に約 5.1 秒の待ちを観測した。集計用の試行は Service の ClusterIP に直接接続した。
2. 各 workload を削除してから作り直す。試行ごとに両方式の順序を交互にした。Mac で `kubectl apply` を起動する直前から、クラスタ内の Python 3.12 クライアントが最初に HTTP 200 と正しい本文を受け取り、その結果が Mac に届くまでを測った。Service IP を取得後、失敗したプローブの次の試行まで 50 ms 待った。HTTP ソケットのタイムアウトは 250 ms とした。この値には Kubernetes の調整、プローブ間隔、結果を返す通信が含まれる。Operator のインストールと VM の起動は計測の前に済ませた。初回の registry pull は比較対象外。再デプロイ時のローカル cache は残した。
3. 1、5、10 回目のデプロイ後に同じクライアント Pod から各 50 回をウォームアップし、単一の HTTP/1.1 接続で直列に各 1,000 回測定。リクエスト開始からレスポンス本文を読み終えるまでをクライアント Pod 内で計測した。測定対象の 3,000 回で、両方式とも再接続は発生しなかった。p95 は昇順に並べたときの nearest-rank。

| 試行 | SpinKube の初回応答 (ms) | wasmCloud v2 の初回応答 (ms) |
| ---: | ---: | ---: |
| 1 | 1,369.7 | 1,071.4 |
| 2 | 1,304.7 | 459.1 |
| 3 | 1,510.5 | 754.1 |
| 4 | 1,261.1 | 822.6 |
| 5 | 1,516.4 | 1,062.9 |
| 6 | 1,681.1 | 449.2 |
| 7 | 1,525.5 | 752.2 |
| 8 | 1,557.6 | 759.3 |
| 9 | 1,810.3 | 754.5 |
| 10 | 1,914.1 | 755.5 |

測定ログ `measurements.jsonl` の SHA-256 は `d5d18cec77e8ddf08a274aa7326e0cb3c254d9ae151a4256dd6e4507ef4a6337`。集計前に、両方式の試行番号、応答本文、サンプル数、ウォームアップ数、接続の再利用を照合した。SpinKube で使った registry はローカル、wasmCloud で使った元の artifact の配布元は GHCR。両方式とも初回の確認後にキャッシュを維持したため、完全なコールドスタートや registry 性能の比較には使えない。

## Itonami Cloud の設計への反映

小さな Wasm workload を頻繁に配置する Kubernetes 経路を追加する際は、常駐 host group を持つ wasmCloud v2 を最初の比較候補にする。SpinKube の Pod ごとの起動と executor 分離も実機で確認できた。Itonami の初版は引き続き、ローカルと Awai Node の単発 Run を同じ契約で通し、Agent の処理時間、権限、チェックポイント、署名付き receipt を測る。この一連の値が揃った段階で、両 Kubernetes 経路を Agent workload で比較できる。
