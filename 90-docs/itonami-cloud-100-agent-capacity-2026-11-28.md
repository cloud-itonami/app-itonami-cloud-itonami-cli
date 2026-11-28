# 共有ホスト上の 100 Agent の RAM と費用

2026-11-28 JST。1 台の Linux ホスト上で、100 Agent を個別にデプロイする構成を比較する。SpinKube は Agent ごとに `SpinApp` と Pod、wasmCloud v2 は Agent ごとに `WorkloadDeployment` を作り、100 workload を同じ `wash` ホスト Pod に載せる。共通のアプリケーションプロセスで 100 Agent の実行を多重化する構成は別の比較になる。

## 実行基盤の RAM 実測

Apple M1 Max、Lima の Alpine Linux 3.24 arm64 VM 1 台（5 vCPU、設定 RAM 10 GiB、swap 無効）、kind 1 ノード、Kubernetes 1.35.0。Spin Operator 0.6.1 と containerd-shim-spin 0.27.2、wasmCloud 2.12.1 を同一クラスタで順に測定した。SpinKube 側は cert-manager、wasmCloud 側は bundled NATS を含む。比較対象外の側の Deployment は停止してから測定した。wasmCloud ホスト Pod のメモリ上限は 4 GiB。Pod 上限は 150 に設定した。計測用 VM のディスクは外付け SSD 上の専用 APFS イメージに置いた。[SpinKube Quickstart](https://www.spinkube.dev/docs/install/quickstart/) · [wasmCloud v2 の operator](https://wasmcloud.com/docs/kubernetes-operator/operator-manual/overview/)

| 指標 | SpinKube | wasmCloud v2 |
| --- | ---: | ---: |
| 共通基盤のみ（各方式の直前） | 1.43 GiB | 1.22 GiB |
| 各方式の基盤を起動、workload 0 | 1.53 GiB | 1.31 GiB |
| 100 workload 待機時、中央値 | 2.70 GiB | 1.54 GiB |
| 100 接続の負荷時、中央値 | 4.09 GiB | 1.58 GiB |
| 同、20 サンプルの最大 | 4.09 GiB | 1.58 GiB |
| 負荷試験で正しい本文の HTTP 200 | 7,200 / 7,200 | 7,200 / 7,200 |

100 接続の負荷時に wasmCloud v2 は VM 全体で **2.51 GiB（約 61%）少なかった**。基盤起動後から 100 workload 待機時までの増分を 100 で割ると、SpinKube は 12.0 MiB、wasmCloud v2 は 2.4 MiB。これは VM 全体の差分を workload 数で割った値。

指標は Linux VM 全体の `MemTotal − MemAvailable`。Kubernetes、ホスト OS、必要なコントローラー、Pod、shim、テストクライアントを含む。平常時は 1 秒間隔で 10 サンプル、負荷時は 20 サンプル。wasmCloud、SpinKube の順に 1 回ずつ実施した。ページキャッシュを消去する操作は行わず、両方式の直前に共通基盤だけの基準値を記録した。

先行した Ubuntu 26.04 VM の SpinKube 単独測定では、同じ 100 接続で中央値 3.09 GiB、サンプル最大 3.10 GiB、7,200 リクエスト成功だった。SSD 上の同一環境で比較した表には、その値を混ぜていない。SpinKube の約 1 GiB の差はゲスト OS、実行順序などが異なるため、要因は特定できていない。

両方式で、[前回と同じ 224,241-byte の Wasm HTTP コンポーネント](itonami-cloud-runtime-benchmark-2026-11-28.md)を使った。100 個のサービスが Ready であることを確認した後、各サービスに 1 本の持続接続を張り、500 ms ごとに 1 リクエストを送った。全 100 接続を約 36 秒並行で稼働させ、負荷時の RAM をサンプリングした。これは Agent の配置と応答の基盤測定で、モデル、プロンプト、会話履歴、ツールの実行プロセスは下の見積もりで別に扱う。

## ホスト費用

AWS EC2 東京リージョン、Linux Shared のオンデマンド、USD、月 730 時間の計算例。2026-11-24 公開の AWS 価格表 version `20261124231417` を使用。`m8g.large`（2 vCPU、8 GiB）は **$84.64/月**、`m8g.xlarge`（4 vCPU、16 GiB）は **$169.27/月**、`m8g.2xlarge`（8 vCPU、32 GiB）は **$338.54/月**。インスタンス料金のみで、ストレージ、転送、管理 Kubernetes、冗長構成、税は別。[AWS EC2 公式東京リージョン価格表 CSV](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonEC2/current/ap-northeast-1/index.csv) · [AWS オンデマンド料金](https://aws.amazon.com/ec2/pricing/on-demand/)

| Agent ごとの追加常駐 RAM | SpinKube の合計 → VM / 月額 | wasmCloud v2 の合計 → VM / 月額 |
| ---: | ---: | ---: |
| 0 MiB | 4.09 GiB → 8 GiB / $84.64 | 1.58 GiB → 8 GiB / $84.64 |
| 32 MiB | 7.22 GiB → 16 GiB / $169.27 | 4.70 GiB → 8 GiB / $84.64 |
| 40 MiB | 8.00 GiB → 16 GiB / $169.27 | 5.49 GiB → 8 GiB / $84.64 |
| 48 MiB | 8.78 GiB → 16 GiB / $169.27 | 6.27 GiB → 8 GiB / $84.64 |
| 64 MiB | 10.34 GiB → 16 GiB / $169.27 | 7.83 GiB → 16 GiB / $169.27 |
| 128 MiB | 16.59 GiB → 32 GiB / $338.54 | 14.08 GiB → 32 GiB / $338.54 |

8 GiB / 20% 予備枠での追加常駐 RAM の上限は、この測定に基づくと Agent 1 体あたり SpinKube 約 23 MiB、wasmCloud v2 約 49 MiB。SpinKube の先行測定値を使うと上限は約 33 MiB で、32 MiB の試算では両方式とも 8 GiB に収まる。40 MiB の試算では両測定の SpinKube が 16 GiB となり、ホスト費用の月差は約 $85。

上表は RAM だけで比較する。各方式の定常負荷中の最大値に、Agent 100 体の追加常駐メモリを加え、合計が公称 RAM の 80% 以内に収まる最小サイズを選んだ。追加メモリは設計上の仮定。Agent の CPU、稼働率、起動ピーク、ツールプロセス、Kubernetes の予約分は本番のサイジング時に検証する。EC2 の同じサイズを選ぶ限り、両方式の計算料金は同額になる。

## 推論の RAM と費用

- **別ホストや API の推論を共有する場合。** Agent ホストはモデルと KV キャッシュを保持しない。推論料金の例として、キャッシュなしの入力 2,000 token、出力 500 token、入力 $0.75・出力 $4.50／100 万 token なら 1 turn は $0.00375。100 体が各 20 turns/日で 30 日稼働すれば両方式とも **$225/月**。各 Agent が毎分 1 turn を 30 日連続で発行する仮定では **$16,200/月**。同時 Agent 数だけでは推論費用は決まらない。実際の推論 rail の単価と使用量を代入する。[トークン単価の例](https://developers.openai.com/api/docs/models/gpt-5.4-mini)
- **同じホストで推論する場合。** モデルの重みを共有しても、推論中の系列が持つ KV キャッシュは別に見積もる。32 層、KV 8 heads、head 次元 128、BF16 の例では `2（KとV） × 32 × 8 × 128 × 2 bytes = 128 KiB/token`。100 系列が同時に 4,096 tokens 分を保持すると **KV だけで 50 GiB**。モデルの重み、推論ワークスペース、Agent 実行基盤のメモリを加える。GPU を使う場合は推論側の VRAM を独立に計測し、コンテキスト長、バッチ、KV の精度、同時生成枠を決めてから費用を算定する。[vLLM の KV キャッシュ設定](https://docs.vllm.ai/en/latest/configuration/engine_args/)

## Itonami Cloud の設計への反映

独立した Wasm workload を 100 体載せるなら、Itonami Cloud の共有ホスト候補は wasmCloud v2。実 Agent のサイジングでは、ホスト側の Agent 実行枠と推論側の同時生成枠を別に持つ。100 体分の追加常駐 RAM を同じホスト Pod 内に置く場合は、ホスト Pod と guest memory の上限を別に設定する。今回のホスト Pod 上限 4 GiB は、40 MiB × 100 の追加常駐状態を検証した値ではない。[wasmCloud の接続・同時実行制限](https://wasmcloud.com/docs/kubernetes-operator/operator-manual/concurrency-and-connections/)

集計 JSON の SHA-256 は `f38ecab6bca533f0ec02c2c216f4271fa815127a48842348f10799493e0092ac`。
