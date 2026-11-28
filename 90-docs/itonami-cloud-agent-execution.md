# Itonami Cloud の AI Agent 実行 CLI

設計提案 · 2026-11-28

既存の `itonami` を、手元の実行と Awai Node 上の実行を操作する共通 CLI にする。Itonami Agent が推論・ツール呼び出し・チェックポイントを扱い、接続先が実行要求と状態を永続化する。Murakumo Node が配置先での起動と権限を検査し、結果を署名付き receipt で返す。最初は単発の Bot 1 体をローカルと Awai Otehon の Node 1 台で動かし、実行契約を検証する。

## wasmCloud と SpinKube から採るもの

| 参考先 | Itonami Cloud で採る設計 |
| --- | --- |
| wasmCloud v2 | `WorkloadDeployment` の宣言と実行先を分ける。WIT の import/export とホストが提供する capability の組み合わせを、Wasm ツールの境界に使う。配置は Node の runtime class と capability を明示して決める。 |
| SpinKube | `SpinApp` と `SpinAppExecutor` の分離を参考に、実行要求と executor の選択を独立させる。executor の違いは CLI の `status` と receipt に表す。 |

wasmCloud の WADM／lattice は v1 の設計。v2 の Kubernetes 経路は `WorkloadDeployment`、operator、host group を使う。SpinKube の `SpinAppExecutor` は Kubernetes のリソースで、shim の RuntimeClass やコンテナ内の Spin を選べる。Itonami Cloud の初版は既存 Node に直接委譲する構成とし、Kubernetes operator はクラスタ運用が必要になった段階で検証する。[wasmCloud v2 の移行ガイド](https://wasmcloud.com/docs/migration/)、[wasmCloud のホストインターフェース](https://wasmcloud.com/docs/kubernetes-operator/host-interface-configuration/)、[SpinApp](https://www.spinkube.dev/docs/reference/spin-app/)、[SpinAppExecutor](https://www.spinkube.dev/docs/reference/spin-app-executor/)。

## 責務と既存機能

| 所有者 | 責務 |
| --- | --- |
| `itonami`（Itonami Cloud CLI） | 接続先の選択、入力検証、実行・停止要求、状態・イベント・receipt の表示。CLI のコマンド登録は既存の commands レジストリを通す。 |
| Itonami Cloud の実行制御サービス | 実行要求の認証、冪等受理、状態とイベントの保存、配置の依頼、権限範囲内での照会と停止。`local` も同じ意味の契約を使う。初版のリモート経路は Node 1 台に固定する。 |
| `itonami-agent` | Agent のターン、Bot profile、記憶、推論 rail、実行のチェックポイント。既存の headless 経路を executor 側から呼び出す。 |
| Murakumo Node／executor | Node の適格性と grant の再検査、artifact の検証、隔離・リソース制限、実行・停止、receipt への署名。Murakumo inference への接続は Agent の推論 rail が決める。 |
| kagi／状態基盤 | kagi が署名鍵と委譲の境界を持つ。実行状態の正本は各環境の台帳に置く。対象経路に既存の実行台帳がある場合は受理・fencing・照合をその台帳に委ねる。分散永続化を接続するときは Yataverse に bytes、Kotobase に参照と確定 head を置く。 |

現行 CLI には `/v1/runs`、SSE、停止・承認操作がある。対話用の契約とクライアントを維持し、Cloud 固有の配置・receipt を扱う操作は、既存の gateway と peer API の契約を棚卸ししてから接続する。CLI 用に Agent loop やスケジューラを複製する必要はない。[既存 CLI の実装](https://github.com/cloud-itonami/app-itonami-cloud-itonami-cli/blob/a1b236092fce385acd93f077a4f9c81d46207306/bin/itonami_main.cljk#L685)、[Itonami Agent](https://github.com/cloud-itonami/itonami-agent)、[Awai Network の構成](https://awai.network/network/anatomy/)。

`itonami-agent` は `~/.itonami` と `itonami.edn` の `:inference`、Hermes は `~/.hermes` と Hermes の設定を使う。Awai Otehon の Hermes が明示した Awai Node の `/v1` を使う既存の例外は、その契約に従う。初版はローカルと Node で同じ Otehon の `oneshot` を接続候補にする。Otehon と Kotoba／Amu Native は同じ受入シナリオで個別に検証し、経路の切替は接続先と executor の選択に明示する。Native は profile を強制できる headless 経路の受入後に接続する。[Otehon の CLI](https://github.com/cloud-itonami/itonami-agent/blob/7c937c15bb86c651d0612a9512562b10dd07993c/otehon/README.md)、[Native launcher の未対応ルート](https://github.com/cloud-itonami/itonami-agent/blob/7c937c15bb86c651d0612a9512562b10dd07993c/bin/itonami-agent#L397-L399)。

ブラウザ版を接続する場合、静的なアプリ配信元と実行制御サービスを分離する。ブラウザは共有 descriptor と broker に宣言したサービスへ接続し、送信直前に操作ごとの同意を得る。Bot の実行 grant は Bot のツール操作を別途制約する。[Kotoba end-user apps の契約](https://github.com/kotoba-lang/kotoba/blob/main/lib/kotoba.wasmWebcomponent/docs/app-environment-contract.md)。

## CLI の提案

```text
itonami agent plan ./run.edn --context local --json
itonami agent run ./run.edn --context local --wait
itonami agent run ./run.edn --context awai-otehon --idempotency-key report-42 --json
itonami agent status RUN_ID --context awai-otehon --json
itonami agent logs RUN_ID --context awai-otehon --after 17 --follow
itonami agent cancel RUN_ID --context awai-otehon --json
itonami agent receipt RUN_ID --context awai-otehon --verify --json
```

`plan` は宣言の解決結果、対象 Node、要求する権限と不足理由を表示する。計画時の許可判定は参考情報として扱い、実行時に再検査する。通常の `run` は受理時に Run ID を返し、`--wait` は終端状態まで待機する。待機を中断しても実行は続き、停止は `cancel` で要求する。サーバー応答を失った場合は同じ冪等キーで照会または再送する。`--json` は単一の JSON 応答、イベント配信は連番付きの NDJSON を使い、進捗は標準エラーに出す。終了コードは受理／成功、入力エラー、接続・認証エラー、実行失敗、結果不明を識別できるようにする。

## 配布、配置、実行を別の記録にする

| 記録 | 内容 |
| --- | --- |
| Bot profile と配布物 | 既存 profile と Awai の L5 manifest／OCI 配布契約を参照する。要求の受理時に profile リビジョン、manifest CID、OCI digest、entry を固定する。OCI tag を入力に使う場合も、起動前に digest まで解決する。Node は署名、digest、manifest の対応を検証する。 |
| Run request | 初版の `run.edn` はローカル入力形式の案。profile 参照、入力参照、runtime class、推論 rail、実行期限、ターン・トークン・金額・並列数の上限、grant 参照を持つ。秘密は参照で指定し、実行制御サービスと Node は caller の権限に応じて解決する。 |
| Run と attempt | Run ID が利用者の要求を識別し、attempt ID が個々の起動を識別する。Node ID、選択した executor、状態と理由、イベント連番を保持する。継続運用を足す段階でだけ `Deployment` に目標状態と `observedGeneration` を持たせ、スケジュールやイベントから同じ Run を発行する。 |
| 署名付き receipt | Run／attempt ID、Agent DID、Node ID、入力参照、固定した artifact・profile・policy のリビジョン、出力参照、使用量、結果と時刻を束ねる。検証側は署名と署名者の委譲、リビジョンと参照の対応を確認する。receipt が証明するのは署名者が記録した実行結果まで。 |

`run.edn` の入力例。参照の値と予算値は検証用の例とする。

```edn
{:version 1
 :profile-ref "profile:read-only-report"
 :input-ref "fixture:report-input-v1"
 :runtime-class :container
 :inference-rail :murakumo
 :limits {:timeout-seconds 120
          :max-turns 3
          :max-tokens 4000
          :max-cost {:currency "JPY" :minor-units 10}
          :max-concurrent-tools 1}
 :grant-ref "grant:report-read"}
```

制御サービスが持つ操作は、計画、実行受理、状態取得、連番からのイベント取得、停止要求、receipt 取得。受理時に確定したリビジョンは応答と状態取得で参照できるようにする。権限と利用予算の上限は、要求の指定を profile と grant が認める範囲まで絞る。初版では HTTP/JSON と既存の SSE クライアントを接続候補とし、peer wire の変更は独立に判断する。

Bot profile と L5 manifest は互換性を維持し、Run request は配置と実行の指定に絞る。Node の runtime class は Awai の `wasm` と `container` を継承する。rootless コンテナと microVM は `container` の backing にあたる。OCI digest と L5 manifest CID は別の内容を識別し、対応付けは検証済みの配布情報に従う。[OCI Descriptor](https://github.com/opencontainers/image-spec/blob/main/descriptor.md)。

## 認可、隔離、復旧

- **起動時の認可。** tsukasa の Node 宣言と runtime class、対象 Bot／Node／操作を束ねた期限つき grant、ホスト側ポリシーを検査する。配置可能という判定に加えて、Node が起動とツール実行の各境界で許可を確認する。Agent DID、操作者、Node の鍵を分離し、wallet や Agent の長期秘密鍵はコンポーネントへ渡さない。
- **ツールの権限。** WIT import を capability の宣言に使い、ホスト側で操作・接続先・データ範囲を絞る。コンテナ経路は専用作業領域、読み取り専用 rootfs、リソース上限、既定で閉じたネットワークを使い、許可済みの推論・ツールはホストの broker 経由で呼ぶ。信頼していないコードの実行は明示した VM 境界に分ける。初版のパイロットは読み取り中心の Bot とする。
- **予算と承認。** 推論とツールを呼ぶ前に期限・金額・トークン・回数を検査する。同時処理では枠を先に予約し、今回の消費上限が確定しないときは保守的に扱う。書き込みや支払いなどの操作には実行権限とは別に、宛先・対象・操作内容・期限に結び付いた承認を要求する。ログには操作の種類と結果を残し、秘密と会話本文は既定で収集対象から外す。
- **同じ要求を一度だけ作る。** 呼び出し元・接続先・冪等キーと要求のハッシュを、Run レコードと同じトランザクションで保存する。同じキーと同じ要求は同じ Run ID を返し、内容が変われば競合を返す。イベントは再配信され得るため、`run-id` と連番で重複を除き、最後に受け取った連番から再開する。
- **古い実行による確定を防ぐ。** attempt に更新可能な lease と単調増加する fencing token を与える。チェックポイント、出力 head、終端状態の確定時に現在の token を照合する。外部呼び出しの直前に operation ID と意図を記録し、外部が冪等キーを受ける場合はそれを伝える。応答喪失などで副作用の結果が不明な場合は照会か人の判断まで再送を止める。

状態遷移は `queued → running ⇄ waiting-approval → succeeded | failed` を基本とし、配置や認可の拒否には `rejected` と理由を返す。停止要求は `cancel-requested` として記録し、停止を確認して `cancelled` を確定する。許可の拒否・期限切れは状態と理由を記録する。lease 喪失や応答喪失で結果を確定できなければ `unknown` とし、自動再起動は副作用なしを確認できる処理か、冪等性を検証した処理に限る。再開時は固定リビジョン、grant の有効性、最後の確定チェックポイントを再検査する。Cell を使う経路では、Node を移しても mon CID と Agent 鍵に結び付く Cell の同一性を保つ。

## 実装順と受入条件

1. **ローカルの単発 Run。** 固定入力を使う Bot 1 体で `plan → run → status → logs → receipt` を通す。既存の対話 run も回帰確認する。重複要求、同じキーでの内容変更、ログ再接続、停止競合をテストする。
2. **Otehon の Node 1 台。** 同じ契約を明示したリモート接続先で通し、同じ fixture の意味上の結果と receipt 検証を比べる。許可なし、digest 不一致、無効な Node、未知の class、未宣言の外部操作、期限切れを拒否する。worker 停止を注入して古い attempt が結果を確定できないこと、結果不明の副作用が再送されないことを確認する。この検証は Native の受入と独立させる。既存の Itonami スケジューラを流用できるかを継続運用の前に確認する。
3. **Wasm ツールを 1 つ。** 外部アクセスを要しない文書の正規化などを WIT で切り出し、宣言済みの入出力だけで動くことを確認する。同じ入力と出力をローカルと Node で検証する。CLI のルーティング、Agent 本体、モデル推論はそれぞれの既存実行経路を使う。
4. **継続運用。** 実測に基づき、スケジュールを Run に変換する reconciler、複数 Node の配置、drain、更新時の段階配信を追加する。単発 Run と継続 Deployment の両方で rollback 対象のリビジョンを固定する。既に確定した外部操作の取り消しは、その操作固有の処理として扱う。

初回の計測では、受理から起動までの時間、executor の起動時間、初回出力までの時間、復旧時間、Run ごとの推論消費を記録する。ローカルと Otehon で同じ入力・モデル・計測条件を使い、試行数と環境を併記する。Kubernetes 対応は、運用対象が既存クラスタを必要とするか、測定したキュー滞留・復旧時間・運用工数が単一 Node の構成では目標を満たせない場合に比較する。候補は wasmCloud v2 の Wasm workload と SpinKube の SpinApp で、同じ権限と復旧の受入条件を課す。

同一の Wasm HTTP コンポーネントを使ったローカル kind での予備実測では、再デプロイから初回応答までの中央値は SpinKube 1.521 秒、wasmCloud v2 0.755 秒だった。Agent の実行経路は今後の受入で検証する。[2026-11-28 のローカル実測](itonami-cloud-runtime-benchmark-2026-11-28.md)
