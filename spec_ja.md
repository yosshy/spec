# Container Storage Interface (CSI)

著者:

* Jie Yu <<jie@mesosphere.io>> (@jieyu)
* Saad Ali <<saadali@google.com>> (@saad-ali)
* James DeFelice <<james@mesosphere.io>> (@jdef)
* <container-storage-interface-working-group@googlegroups.com>

## 表記法

キーワード "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", "OPTIONAL" は
 [RFC 2119](http://tools.ietf.org/html/rfc2119) (Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997)
 で記述された通りに解釈される。

キーワード "unspecified", "undefined", "implementation-defined" は
[rationale for the C99 standard](http://www.open-std.org/jtc1/sc22/wg14/www/C99RationaleV5.10.pdf#page=18)
で記述された通りに解釈される。

実装されるプロトコルの MUST, REQUIRED, SHALL 要求を１つ以上満たせなかった場合、その実装は仕様に準拠していない。

実装されるプロトコルの MUST, REQUIRED, SHALL 要求を全て満たしていた場合、その実装は仕様に準拠している。

## 用語集

| 用語              | 定義                                             |
|-------------------|--------------------------------------------------|
| ボリューム                | (CSI を介して) CO で管理されたコンテナの中で利用可能になるストレージの単位                        |
| ブロックボリューム        | コンテナ内でブロックデバイスとして現れるボリューム                                                |
| マウントボリューム        | コンテナ内で指定されたファイルシステムを使ってマウントされ、ディレクトリとして現れるボリューム    |
| CO                        | コンテナオーケストレーションシステム (CSI サービス RPC を使用するプラグインと通信する)            |
| SP                        | ストレージプロバイダ (CSI プラグイン実装のベンダ)                                                 |
| RPC                       | [リモートプロシージャコール](https://en.wikipedia.org/wiki/Remote_procedure_call).                |
| ノード                    | ユーザワークロードが実行されるホスト (プラグインの観点からノードIDで唯一に識別される)             |
| プラグイン                | 「プラグイン実装」として知られる (CSI サービスを実装した gRPCエンドポイント)                      |
| プラグインスーパーバイザ  | プラグインのライフサイクルを管轄するプロセス (COでも良い(MAY))                                    |
| ワークロード              | COによってスケジュールされた「作業」の原始的な単位。これはコンテナ又はコンテナの集合でも良い(MAY) |


## 目的

ストレージベンダー(SP)によるプラグイン開発を可能にし、数あるコンテナオーケストレーションシステム(CO)間で動作する工業標準「Container Storage Interface (CSI)」を定義する事。

### MVP のゴール

Container Storage Interface (CSI) は

* SP 開発者が CSI を実装する全 CO で「ただ動く」ための CSI 準拠プラグイン開発を可能にする
* 以下を可能にする API (RPC) を定義する:
  * ボリュームの動的プロビジョニング／デプロビジョニング
  * ノードにボリュームをアタッチ／デタッチ
  * ノードにボリュームをマウント／アンマウント
  * ブロック／マウント可能ボリューム両方の消費
  * ローカルストレージプロバイダ (例：device mapper, lvm).
  * スナップショットの作成／削除 (スナップショット元はボリューム)
  * スナップショットから新しいボリュームをプロビジョニング (スナップショットの復元 (元ボリューム中のデータは削除され、スナップショット中のデータで置換される)はスコープ外)
* プラグインプロトコルの「推奨」を定義する
  * スーパーバイザがプラグインを設定するプロセスを定義
  * コンテナデプロイの考慮 (CAP_SYS_ADMIN、マウントネームスペース、など)

### MVP の非ゴール

Container Storage Interface (CSI) は明示的に、以下のものを定義、提供、記述しない：

* プラグインスーパーバイザーがプラグインのライフサイクルを管理する特定の機構 (以下を含む):
  * 状態の管理方法 (例：何がアタッチ、マウントなどされているか)
  * プラグインのデプロイ、インストール、アップグレード、アンインストール、監視、(不測の停止の場合の)再起動方法
* 「ストレージのグレード」(いわゆる「ストレージクラス」)を表現する最初のクラスメッセージ構造/項目
* プロトコルレベルの認証・認可
* プラグインのパッケージ
* POSIX 準拠：CSI は提供されたボリュームが POSIX 準拠のファイルシステムである保証を提供しません。
  準拠がプラグイン実装(そしてそれが依存する何らかのバックエンドストレージシステム)によって決定される
  CSI はプラグインスーパーバイザーまたはCO が POSIX 準拠の方法でプラグインが管理するボリュームとの相互作用を遮断すべきではない (SHALL NOT)。

## 解決方法の概要

本使用は、CSI準拠のプラグインを実装するための、最小の操作上のインターフェースとストレージプロバイダ(SP)用のパッケージ化の勧告を定義する。
インターフェースはプラグインが公開しなければならない(MUST) RPC を定義する: これは CSI 仕様の**最初の焦点**である。
いずれの操作上・パッケージ化勧告は CO 間の準拠を促進する為の追加のガイダンスを提案します。

### アーキテクチャ

本仕様の最初の焦点は、CO と Plugin 間の**プロトコル**です。
これは、様々なデプロイメントアーキテクチャ用の CO 間準拠プラグインを送り出す事を可能にすべきです(SHOULD)。
CO は中央集権型とヘッドレス型プラグインの両方(分割コンポーネントと統合プラグインと同様に)を扱う能力を持っているべきである(SHOULD)。
これらの可能性の幾つかは下記に図示されています。

```
                            CO マスターホスト
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    | Controller |  |
|  |            +----------->   Plugin   |  |
|  +------------+           +------------+  |
|                                           |
+-------------------------------------------+

                              CO ノードホスト
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    |    Node    |  |
|  |            +----------->   Plugin   |  |
|  +------------+           +------------+  |
|                                           |
+-------------------------------------------+

図1: プラグインはクラスタ内の全ノード上で実行されます: 中央集権型のコントローラープラグインは CO マスターホスト上で利用可能で、ノードプラグインは全ての CO ノード上で利用可能です。

```

```
                              CO ノードホスト
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    | Controller |  |
|  |            +--+-------->   Plugin   |  |
|  +------------+  |        +------------+  |
|                  |                        |
|                  |                        |
|                  |        +------------+  |
|                  |        |    Node    |  |
|                  +-------->   Plugin   |  |
|                           +------------+  |
|                                           |
+-------------------------------------------+

図2: ヘッドレスプラグイン環境構築 (COノードホストのみプラグインを実行します)。別々の分離コンポーネントプラグインはそれぞれ、コントローラサービスとノードサービスを提供します。
```

```
                              CO ノードホスト
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    | Controller |  |
|  |            +----------->    Node    |  |
|  +------------+           |   Plugin   |  |
|                           +------------+  |
|                                           |
+-------------------------------------------+

図3: ヘッドレスプラグイン環境構築 (COノードホストのみプラグインを実行します)。統一プラグインコンポーネントはコントローラサービスとノードサービス両方を提供します。
```

``` 
                              CO ノードホスト
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    |    Node    |  |
|  |            +----------->   Plugin   |  |
|  +------------+           +------------+  |
|                                           |
+-------------------------------------------+

図4: ヘッドレスプラグイン環境構築 (COノードホストのみプラグインを実行します)。ノードのみのプラグインコンポーネントがノードサービスのみを提供します。これの GetPluginCapabilities RPC は CONTROLLER_SERVICE ケーパビリティを報告しません。
```

### ボリュームライフサイクル

```
   CreateVolume +------------+ DeleteVolume
 +------------->|  CREATED   +--------------+
 |              +---+----^---+              |
 |       Controller |    | Controller       v
+++         Publish |    | Unpublish       +++
|X|          Volume |    | Volume          | |
+-+             +---v----+---+             +-+
                | NODE_READY |
                +---+----^---+
               Node |    | Node
            Publish |    | Unpublish
             Volume |    | Volume
                +---v----+---+
                | PUBLISHED  |
                +------------+

図5: 動的に作成されたボリュームのライフサイクル (作成～削除)
```

```
   CreateVolume +------------+ DeleteVolume
 +------------->|  CREATED   +--------------+
 |              +---+----^---+              |
 |       Controller |    | Controller       v
+++         Publish |    | Unpublish       +++
|X|          Volume |    | Volume          | |
+-+             +---v----+---+             +-+
                | NODE_READY |
                +---+----^---+
               Node |    | Node
              Stage |    | Unstage
             Volume |    | Volume
                +---v----+---+
                |  VOL_READY |
                +---+----^---+
               Node |    | Node
            Publish |    | Unpublish
             Volume |    | Volume
                +---v----+---+
                | PUBLISHED  |
                +------------+

図6: ノードプラグインが STAGE_UNSTAGE_VOLUME ケーパビリティを宣言する場合の動的に作成されたボリュームのライフサイクル (作成～削除)
```

```
    Controller                  Controller
       Publish                  Unpublish
        Volume  +------------+  Volume
 +------------->+ NODE_READY +--------------+
 |              +---+----^---+              |
 |             Node |    | Node             v
+++         Publish |    | Unpublish       +++
|X| <-+      Volume |    | Volume          | |
+++   |         +---v----+---+             +-+
 |    |         | PUBLISHED  |
 |    |         +------------+
 +----+
   Validate
   Volume
   Capabilities

図7: あるノードに公開される(NodePublishVolume)前に、ノードに公開する(`ControllerPublishVolume`)ようコントローラに要求する、事前作成されたボリュームのライフサイクル。
```

```
       +-+  +-+
       |X|  | |
       +++  +^+
        |    |
   Node |    | Node
Publish |    | Unpublish
 Volume |    | Volume
    +---v----+---+
    | PUBLISHED  |
    +------------+

図8: プラグインはケーパビリティ API 経由で禁忌を犯す事で他のライフサイクルステップに先んじても良い(MAY)。このようなプラグインのボリュームとの相互作用は `NodePublishVolume` と `NodeUnpublishVolume` 呼び出しの2つに集約されます。
```

上記の図は、本仕様で示された API を経由して CO がどのようにボリュームのライフサイクルを管理してもよい(MAY)かに関する一般的な想定を図示しています。
プラグインは１つのインターフェース用の全 RPC を公開すべきです(SHOULD): コントローラプラグインは `Controller` サービス用の全 RPC を実装すべきです(SHOULD)。
未サポートの RPC は、未サポートである事が判るように示された適切なエラーコード(例: `CALL_NOT_IMPLEMENTED`)を返すべきです(SHOULD)。
プラグインのケーパビリティの完全な一覧は `ControllerGetCapabilities` と `NodeGetCapabilities` RPC でドキュメント化されています。

## コンテナーストレージインターフェース(Container Storage Interface)

本章は CO とプラグイン間のインターフェースについて説明します。

### RPC インターフェース

RPC を介したプラグインとの CO インターフェース。
各 SP は以下を提供しなければならない(MUST):

* **ノードプラグイン**: SP がプロビジョニングしたボリュームが公開される場所であるノード上で実行しなければならない(MUST) CSI RPC をサービスする gRPC エンドポイント。
* **コントローラープラグイン**: どこで実行してもよい(MAY) CSI RPC をサービスする gRPC エンドポイント。
* 幾つかの状況で、単一 gRPC エンドポイントが全 CSI RPC をサービスしても良い(MAY) ([Architecture](#architecture)の図3参照)。

```protobuf
syntax = "proto3";
package csi.v1;

import "google/protobuf/descriptor.proto";
import "google/protobuf/timestamp.proto";
import "google/protobuf/wrappers.proto";

option go_package = "csi";

extend google.protobuf.EnumOptions {
  // Indicates that this enum is OPTIONAL and part of an experimental
  // API that may be deprecated and eventually removed between minor
  // releases.
  bool alpha_enum = 1060;
}
extend google.protobuf.EnumValueOptions {
  // Indicates that this enum value is OPTIONAL and part of an
  // experimental API that may be deprecated and eventually removed
  // between minor releases.
  bool alpha_enum_value = 1060;
}
extend google.protobuf.FieldOptions {
  // Indicates that a field MAY contain information that is sensitive
  // and MUST be treated as such (e.g. not logged).
  bool csi_secret = 1059;

  // Indicates that this field is OPTIONAL and part of an experimental
  // API that may be deprecated and eventually removed between minor
  // releases.
  bool alpha_field = 1060;
}
extend google.protobuf.MessageOptions {
  // Indicates that this message is OPTIONAL and part of an experimental
  // API that may be deprecated and eventually removed between minor
  // releases.
  bool alpha_message = 1060;
}
extend google.protobuf.MethodOptions {
  // Indicates that this method is OPTIONAL and part of an experimental
  // API that may be deprecated and eventually removed between minor
  // releases.
  bool alpha_method = 1060;
}
extend google.protobuf.ServiceOptions {
  // Indicates that this service is OPTIONAL and part of an experimental
  // API that may be deprecated and eventually removed between minor
  // releases.
  bool alpha_service = 1060;
}
```

RPC のセットが3つある:

* **アイデンティティサービス**: ノードプラグインとコントローラープラグインはこの RPC セットを実装しなければならない(MUST)。
* **コントローラーサービス**: コントローラープラグインはこの RPC セットを実装しなければならない(MUST)。
* **ノードサービス**: ノードプラグインはこの RPC セットを実装しなければならない(MUST)。

```protobuf
service Identity {
  rpc GetPluginInfo(GetPluginInfoRequest)
    returns (GetPluginInfoResponse) {}

  rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
    returns (GetPluginCapabilitiesResponse) {}

  rpc Probe (ProbeRequest)
    returns (ProbeResponse) {}
}

service Controller {
  rpc CreateVolume (CreateVolumeRequest)
    returns (CreateVolumeResponse) {}

  rpc DeleteVolume (DeleteVolumeRequest)
    returns (DeleteVolumeResponse) {}

  rpc ControllerPublishVolume (ControllerPublishVolumeRequest)
    returns (ControllerPublishVolumeResponse) {}

  rpc ControllerUnpublishVolume (ControllerUnpublishVolumeRequest)
    returns (ControllerUnpublishVolumeResponse) {}

  rpc ValidateVolumeCapabilities (ValidateVolumeCapabilitiesRequest)
    returns (ValidateVolumeCapabilitiesResponse) {}

  rpc ListVolumes (ListVolumesRequest)
    returns (ListVolumesResponse) {}

  rpc GetCapacity (GetCapacityRequest)
    returns (GetCapacityResponse) {}

  rpc ControllerGetCapabilities (ControllerGetCapabilitiesRequest)
    returns (ControllerGetCapabilitiesResponse) {}

  rpc CreateSnapshot (CreateSnapshotRequest)
    returns (CreateSnapshotResponse) {}

  rpc DeleteSnapshot (DeleteSnapshotRequest)
    returns (DeleteSnapshotResponse) {}

  rpc ListSnapshots (ListSnapshotsRequest)
    returns (ListSnapshotsResponse) {}

  rpc ControllerExpandVolume (ControllerExpandVolumeRequest)
    returns (ControllerExpandVolumeResponse) {}

  rpc ControllerGetVolume (ControllerGetVolumeRequest)
    returns (ControllerGetVolumeResponse) {
        option (alpha_method) = true;
    }
}

service Node {
  rpc NodeStageVolume (NodeStageVolumeRequest)
    returns (NodeStageVolumeResponse) {}

  rpc NodeUnstageVolume (NodeUnstageVolumeRequest)
    returns (NodeUnstageVolumeResponse) {}

  rpc NodePublishVolume (NodePublishVolumeRequest)
    returns (NodePublishVolumeResponse) {}

  rpc NodeUnpublishVolume (NodeUnpublishVolumeRequest)
    returns (NodeUnpublishVolumeResponse) {}

  rpc NodeGetVolumeStats (NodeGetVolumeStatsRequest)
    returns (NodeGetVolumeStatsResponse) {}


  rpc NodeExpandVolume(NodeExpandVolumeRequest)
    returns (NodeExpandVolumeResponse) {}


  rpc NodeGetCapabilities (NodeGetCapabilitiesRequest)
    returns (NodeGetCapabilitiesResponse) {}

  rpc NodeGetInfo (NodeGetInfoRequest)
    returns (NodeGetInfoResponse) {}
}
```

#### 同時性

一般に、クラスタオーケストレーター(CO)は所定の時間にボリューム毎に「フライト中(in-flight)」複数のコールが無い事を保証する責任を持つ。
しかし、幾つかの状況(例えば、CO がクラッシュして再起動する際)では CO はステートを失っても良く(MAY) 、同じボリュームに対して同時に複数のコールを発行しても良い(MAY)。

プラグインは可能な限り優雅に(gracefully)これを扱うべきである(SHOULD)。
この場合、プラグインがエラーコード `ABORTED` を返しても良い(MAY)。(詳細は[Error Scheme](#error-scheme)セクション参照)

#### 項目要求

ここの中で文書化された要求は、(別途記載されていない限り)本仕様で定義された全ての protobuf メッセージタイプに対して等しく例外なしに適用されます。
これらの要求の違反は、全ての CO、プラグイン、あるいは CSI ミドルウェア実装と互換性のない RPC メッセージデータ中の結果となり得ます(MAY)。

##### サイズ制限

CSI は様々なタイプの項目に対して全般サイズ制限を定義しています(以下テーブル参照)。

特定項目の全般サイズ制限はその項目の説明で異なるサイズ制限を明示する事で上書きできます(MAY)。

他で明示されない限り、項目はここで定義された制限を超過すべきではない(SHALL NOT)。
これらの制限は、CO・プラグイン両方で生成されたメッセージに適用されます。

| サイズ     | 項目タイプ    |
|------------|---------------------|
| 128 bytes  | string              |
| 4 KiB      | map<string, string> |

##### `必須(REQUIRED)` vs. `オプション(OPTIONAL)`

* `必須(REQUIRED)`と記述された項目は指定され、RPC 毎の警告の対象にならなければなりません(MUST)。※警告は稀であるべきです(SHOULD)。
* `必須(REQUIRED)`と記述された`繰り返し(repeated)`または`マップ(map)`項目は少なくとも1要素を含んでなければなりません(MUST)。
* `オプション(OPTIONAL)`と記述された項目は指定してもよく、仕様はデフォルト(このような項目の0値)として想定された挙動を明確に定義されるべきです(SHALL)。

スカラー項目(REQUIREDな値ですら)は未指定時にデフォルト化され、デフォルト値にセットされる任意の項目は [proto3](https://developers.google.com/protocol-buffers/docs/proto3#default) 通りに通信上でシリアル化されません。

#### タイムアウト

本仕様で定義された任意の RPC はタイムアウトしても良く(MAY)、再試行しても良い(MAY)。
CO は１コールの待機を厭わない最大時間(再試行の間にどれぐらい長く待つか)と何回再試行するか(これらの値はプラグインとCO間で交渉されない)を選択できる(MAY)。

冪等性要件は、再試行時にコールを止めた所で、同じ項目を持つ再試行コールが続く事を保証する。
コールをキャンセルする唯一の方法は、(もしあれば)「打ち消し(negation)」コールを発行する事である。
例えば、保留された `ControllerPublishVolume` 操作のキャンセルの為の `ControllerUnpublishVolume` コール発行、等。
幾つかの場合、「打ち消し(negation)」コール実行の為に保留された操作の結果に依存する為、CO は保留された操作をキャンセルできないかもしれない(MAY NOT)。

例えば、`CreateVolume` コールが絶対完了しないのであれば、CO は `DeleteVolume` をコールする為に使用する `volume_id` が得られない可能性がある(MAY NOT)。

### エラースキーム

本仕様中の全 CSI API コールは [標準 gRPC ステータス](https://github.com/grpc/grpc/blob/master/src/proto/grpc/status/status.proto) を返却しなければならない(MUST)。
ほとんどの gRPC ライブラリはこのステータス項目を設定・読み取るヘルパーメソッドを提供する。

ステータス`コード`は[canonical error code](https://github.com/grpc/grpc-go/blob/master/codes/codes.go)を含んでいなければならない(MUST)。CO は全ての有効なエラーコードを扱わなければならない(MUST)。指定された状態に遭遇した際、各 RPC はプラグインによって返却されなければならない(MUST) gRPC エラーコードのセットを定義する。これらに加え、下記で定義された状態に遭遇した際、プラグインは関連する gRPC エラーコードを返却しなければならない(MUST)。

| 状態 | gRPC コード | 説明 | 復旧挙動 |
|-----------|-----------|-------------|-------------------|
| 必須項目欠如 | 3 INVALID_ARGUMENT | 要求から必須項目がかけている事を示す。`status.message`でより人間が読みやすい情報が提供されても良い(MAY)。| 呼び出し元は、再試行前に不足している必須項目を追加して要求を修正しないといけない(MUST)。 |
| 要求中に不正あるいは未対応の項目 | 3 INVALID_ARGUMENT | 本項目(訳注:おそらく本要求の間違い)中に１つ又は複数の項目がプラグインによって許可されていないか、不正な値があるかのどちらかである事を示す。`status.message`でより人間が読みやすい情報が提供されても良い(MAY)。 | 呼び出し元は再試行前に項目を修正しなければならない(MUST)。 |
| 権限不足 | 7 PERMISSION_DENIED | プラグインは 1 RPC 中に存在するシークレットからアイデンティティを得るあるいは推測できるが、アイデンティティがその RPC を実行する権限を持っていない。 | システム管理者は、必須の権限が付与されているか確認すべきであり(SHOULD)、その後呼び出し元は試行した RPC を再試行する事ができる(MAY)。 |
| ボリューム操作保留 | 10 ABORTED | 指定されたボリューム用の保留中の操作が既にある事を示す。一般に、クラスタオーケストレーター(CO)は与えられた時間で１ボリュームあたりの「飛行中の(in-flight)」が複数ない事を保証する責任があります。しかし、幾つかのケースで(例えば、COがクラッシュして再起動する際)、CO は状態を失うかもしれず(MAY)、同じボリュームに対して同時に複数のコールを発行するかもしれません(MAY)。プラグインは可能な限りこれを穏便に扱うべき(SHOULD)で、2つめのコールを拒否する為にエラーコードを返す事ができる(MAY)。 | 呼び出し元は指定されたボリュームに対する保留された他のコールがない事を保証し、その後指数関数的後退でリトライするべきです(SHOULD)。 |
| コール未実装 | 12 UNIMPLEMENTED | 実行された RPC はプラグインで実装されていないか、操作中のプラグインの現在のモードで無効化されている。 | 呼び出し元は再試行すべきではない(MUST NOT)。プラグインの機能を検出するため、呼び出し元は `GetPluginCapabilities`、`ControllerGetCapabilities`、`NodeGetCapabilities` を呼び出す事ができます(MAY)。 |
| 未認証 | 16 UNAUTHENTICATED | 実行された RPC が認証用の有効なシークレットを持っていない。 |  呼び出し元は RPC で提供されたシークレットを修正するか、または試行された RPC に対し、プラグインによって認証が通過するようなシークレットを再取得して使用するかどちらかを行うべきである(SHOULD)。 |

ステータス `コード` が `OK` でない場合、ステータス `メッセージ` は人間が読めるエラーの説明を含まなければならない(MUST)。
この文字列は CO からエンドユーザーに対して表面化されても良い(MAY)。

ステータス `詳細` は空でなければならない(MUST)。将来的に、よりスマートなエラーハンドリングと障害解決を実装する為、本仕様はステータス `コード` が `OK` でない場合に機械が解読可能な protobuf メッセージを返却する為に `詳細` を要求するかもしれない(MAY)。

### シークレット要件

RPC リクエストを完了する為、プラグインがシークレットを要求しても良い(MAY)。
シークレットは、キーがシークレットの名前を示し(例："username", "password")、値がシークレットデータを含む(例: "bob", "abc123")文字列→文字列のマップである。
各キーは英数字、'-', '_', '.' で構成されなければならない(MUST)。
各値は有効な文字列を含まなければならない(MUST)。
SP は、base64 のようなバイナリからテキストに変換するエンコーディングスキームを使用してバイナリデータ(非文字列)を受け入れても良い(MAY)。
SP は要求するシークレットのキーと値に関する要件をドキュメント中で告知すべきである(SHALL)。
CO は要求したシークレットのパススルーを許可すべきである(SHALL)。
CO は同じシークレットを全 RPC に渡しても良い(MAY)が、しかしながら SP が期待する全てのユニークシークレット用のキーは全 CSI 操作を通して単一でなければならない(MUST)。
この情報はセンシティブで、(ログに残さないなど) CO によってそのように扱われなければならない(MUST)。

### アイデンティティサービス RPC

アイデンティティサービス RPC は CO がプラグインに機能、健全性、他のメタデータを問い合わせる事を可能にする。
The general flow of the success case MAY be as follows (protos illustrated in YAML for brevity):
成功ケースの一般的なフローは以下 (簡潔に YAML で図示されたプロトコル) のようになりうる (MAY)。

1. CO はアイデンティティ RPC 経由でメタデータを問い合わせる。

```
   # CO --(GetPluginInfo)--> Plugin
   request:
   response:
      name: org.foo.whizbang.super-plugin
      vendor_version: blue-green
      manifest:
        baz: qaz
```

2. CO はプラグインで利用可能な機能を問い合わせる。

```
   # CO --(GetPluginCapabilities)--> Plugin
   request:
   response:
     capabilities:
       - service:
           type: CONTROLLER_SERVICE
```

3. CO はプラグインの準備状況を問い合わせる。

```
   # CO --(Probe)--> Plugin
   request:
   response: {}
```

#### `GetPluginInfo`

```protobuf
message GetPluginInfoRequest {
  // Intentionally empty.
}

message GetPluginInfoResponse {
  // The name MUST follow domain name notation format
  // (https://tools.ietf.org/html/rfc1035#section-2.3.1). It SHOULD
  // include the plugin's host company name and the plugin name,
  // to minimize the possibility of collisions. It MUST be 63
  // characters or less, beginning and ending with an alphanumeric
  // character ([a-z0-9A-Z]) with dashes (-), dots (.), and
  // alphanumerics between. This field is REQUIRED.
  string name = 1;

  // This field is REQUIRED. Value of this field is opaque to the CO.
  string vendor_version = 2;

  // This field is OPTIONAL. Values are opaque to the CO.
  map<string, string> manifest = 3;
}
```

##### GetPluginInfo エラー

プラグインが GetPluginInfo コールを無事に完了できない場合、gRPC ステータス中に non-ok gRPC コードを返却しなければならない(MUST)。

#### `GetPluginCapabilities`

この 必須 RPC は CO がプラグインがサポートする機能を「総じて」問い合わせる事を可能にする: これは、(デプロイ用であるため) プラグインソフトウェアの全インスタンスの全機能の総計である。
プラグインの同じバージョンの全インスタンス (`GetPluginInfoResponse` の `vendor_version` 参照)は
(a)クラスタ上でインスタンスがデプロイされる場所と同様に (b)インスタンスが提供する RPC の種類の両方の機能と準備状況の同じセットを返却するべきである(SHALL)。

```protobuf
message GetPluginCapabilitiesRequest {
  // Intentionally empty.
}

message GetPluginCapabilitiesResponse {
  // All the capabilities that the controller service supports. This
  // field is OPTIONAL.
  repeated PluginCapability capabilities = 1;
}

// Specifies a capability of the plugin.
message PluginCapability {
  message Service {
    enum Type {
      UNKNOWN = 0;
      // CONTROLLER_SERVICE indicates that the Plugin provides RPCs for
      // the ControllerService. Plugins SHOULD provide this capability.
      // In rare cases certain plugins MAY wish to omit the
      // ControllerService entirely from their implementation, but such
      // SHOULD NOT be the common case.
      // The presence of this capability determines whether the CO will
      // attempt to invoke the REQUIRED ControllerService RPCs, as well
      // as specific RPCs as indicated by ControllerGetCapabilities.
      CONTROLLER_SERVICE = 1;

      // VOLUME_ACCESSIBILITY_CONSTRAINTS indicates that the volumes for
      // this plugin MAY NOT be equally accessible by all nodes in the
      // cluster. The CO MUST use the topology information returned by
      // CreateVolumeRequest along with the topology information
      // returned by NodeGetInfo to ensure that a given volume is
      // accessible from a given node when scheduling workloads.
      VOLUME_ACCESSIBILITY_CONSTRAINTS = 2;
    }
    Type type = 1;
  }

  message VolumeExpansion {
    enum Type {
      UNKNOWN = 0;

      // ONLINE indicates that volumes may be expanded when published to
      // a node. When a Plugin implements this capability it MUST
      // implement either the EXPAND_VOLUME controller capability or the
      // EXPAND_VOLUME node capability or both. When a plugin supports
      // ONLINE volume expansion and also has the EXPAND_VOLUME
      // controller capability then the plugin MUST support expansion of
      // volumes currently published and available on a node. When a
      // plugin supports ONLINE volume expansion and also has the
      // EXPAND_VOLUME node capability then the plugin MAY support
      // expansion of node-published volume via NodeExpandVolume.
      //
      // Example 1: Given a shared filesystem volume (e.g. GlusterFs),
      //   the Plugin may set the ONLINE volume expansion capability and
      //   implement ControllerExpandVolume but not NodeExpandVolume.
      //
      // Example 2: Given a block storage volume type (e.g. EBS), the
      //   Plugin may set the ONLINE volume expansion capability and
      //   implement both ControllerExpandVolume and NodeExpandVolume.
      //
      // Example 3: Given a Plugin that supports volume expansion only
      //   upon a node, the Plugin may set the ONLINE volume
      //   expansion capability and implement NodeExpandVolume but not
      //   ControllerExpandVolume.
      ONLINE = 1;

      // OFFLINE indicates that volumes currently published and
      // available on a node SHALL NOT be expanded via
      // ControllerExpandVolume. When a plugin supports OFFLINE volume
      // expansion it MUST implement either the EXPAND_VOLUME controller
      // capability or both the EXPAND_VOLUME controller capability and
      // the EXPAND_VOLUME node capability.
      //
      // Example 1: Given a block storage volume type (e.g. Azure Disk)
      //   that does not support expansion of "node-attached" (i.e.
      //   controller-published) volumes, the Plugin may indicate
      //   OFFLINE volume expansion support and implement both
      //   ControllerExpandVolume and NodeExpandVolume.
      OFFLINE = 2;
    }
    Type type = 1;
  }

  oneof type {
    // Service that the plugin supports.
    Service service = 1;
    VolumeExpansion volume_expansion = 2;
  }
}
```

##### GetPluginCapabilities エラー

プラグインが GetPluginCapabilities コールを無事に完了できない場合、gRPC ステータス中で non-ok gRPC コードを返却すべきである(MUST)。

#### `Probe`

プラグインはこの RPC コールを実装しなければならない(MUST)。
Probe RPC の第一の用途は、プラグインが健全で準備完了な状態である事を検証することである。
(不成功応答を経由して) 不健全状態が報告された場合、CO はプラグインを健全な状態にする為にアクションを起こしても良い(MAY)。
このようなアクションは以下を含んでも良い(MAY)が、限定されるべきではない(SHALL NOT):

* プラグインコンテナの再起動
* プラグインスーパーバイザーへの通知

プラグインは、稼働の為に正しい設定、デバイス、依存関係、ドライバがある事を検証し、検証が無事完了した場合には成功を返却しても良い(MAY)。
CO はこの RPC をいつでも実行して良い(MAY)。
CO は、プラグインの実装が些細なものではない可能性があり(MAY NOT)、このような連続コールによりオーバーヘッドを生じるかもしれない(MAY)事を理解しつつ、このコールを複数回呼び出しても良い(MAY)。
SP は特定のプラグインの本 RPC の実装に関してガイダンスと既知の制限事項を文書化すべきである(SHALL)。
例えば、SP は、その Probe 実装がコールされるべき(SHOULD)最大頻度について文書化しても良い(MAY)。

```protobuf
message ProbeRequest {
  // Intentionally empty.
}

message ProbeResponse {
  // Readiness allows a plugin to report its initialization status back
  // to the CO. Initialization for some plugins MAY be time consuming
  // and it is important for a CO to distinguish between the following
  // cases:
  //
  // 1) The plugin is in an unhealthy state and MAY need restarting. In
  //    this case a gRPC error code SHALL be returned.
  // 2) The plugin is still initializing, but is otherwise perfectly
  //    healthy. In this case a successful response SHALL be returned
  //    with a readiness value of `false`. Calls to the plugin's
  //    Controller and/or Node services MAY fail due to an incomplete
  //    initialization state.
  // 3) The plugin has finished initializing and is ready to service
  //    calls to its Controller and/or Node services. A successful
  //    response is returned with a readiness value of `true`.
  //
  // This field is OPTIONAL. If not present, the caller SHALL assume
  // that the plugin is in a ready state and is accepting calls to its
  // Controller and/or Node services (according to the plugin's reported
  // capabilities).
  .google.protobuf.BoolValue ready = 1;
}
```

##### Probe エラー

プラグインが Probe コールを無事に完了できない場合、gRPC ステータス中で non-ok gRPC コードを返却すべきである(MUST)。

もし以下で定義された条件に遭遇した場合、プラグインは指定された gRPC エラーコードを返却しなければならない(MUST)。
CO はこの gRPC エラーコードに遭遇した場合、指定されたエラーリカバリー動作を実装しなければならない(MUST)。

| 条件 | gRPC コード | 説明 | リカバリー動作 |
|-----------|-----------|-------------|-------------------|
| プラグインが不健全 | 9 FAILED_PRECONDITION | プラグインが健全/準備完了状態にない事を示す。 | 呼び出し元は、プラグインが健全でなく、それゆえ今後の RPC がこの状態の為に失敗するかもしれない(MAY)ことを想定すべきである(SHALL)。 |
| 必要な依存性が不足 | 9 FAILED_PRECONDITION | プラグインが１つ以上の必要な依存性を失っている事を示す。 | 呼び出し元は、プラグインが不健全である事を想定しなければならない(MUST)。 |


### コントローラーサービス RPC

#### `CreateVolume`

`CREATE_DELETE_VOLUME` コントローラー機能を持つ場合、コントローラープラグインは本 RPC を実装しなければならない(MUST)。
本 RPC は、ユーザの代わりに (ブロックデバイスまたはマウントされたファイルシステムのどちらかとして使用される) 新しいボリュームを用意するため、CO によってコールされる。

本操作は冪等性を持っていなければならない(MUST)。
もし、指定されたボリューム`名`を関連付けられたボリュームが既に存在し、`accessibility_requirements` からアクセス可能で、`CreateVolumeRequest` 中で指定された `capacity_range`、`volume_capabilities`、`parameters` と互換性がある場合、プラグインは関連する `CreateVolumeResponse` で `0 OK` を応答しなければならない(MUST)。

プラグインは３種類のボリュームを作成する可能性がある(MAY):

- 空ボリューム。プラグインが `CREATE_DELETE_VOLUME` オプション機能をサポートする場合。
- 既存スナップショットから。 プラグインが `CREATE_DELETE_VOLUME`、`CREATE_DELETE_SNAPSHOT` オプション機能をサポートする場合。
- 既存ボリュームから。プラグインがボリューム複製(cloning)をサポートし、`CREATE_DELETE_VOLUME`、`CLONE_VOLUME` オプション機能を報告する場合。

CO が既存のスナップショットまたはボリュームからボリュームを作成し、要求されたボリュームサイズが元のスナップショットされたボリューム(又は複製されたボリューム)より大きかった場合、プラグインは `OUT_OF_RANGE` エラーでこのような要求を拒否するか、(`NodePublish` コールによりワークロードが示されている場合)要求された(より大きな)サイズでスナップショット(又は元ボリューム)からのデータの両方を持つボリュームを用意しなければならない(MUST)。

明らかに、(ボリュームが `VolumeCapability` アクセスタイプ `MountVolume` を持ち、要求された容量を用意する為にファイルシステムリサイズが要求された場合)`NodePublish` コール時（又はそれ以前）に新規作成されたボリュームのファイルシステムをリサイズする責任はプラグインにあります。

```protobuf
message CreateVolumeRequest {
  // The suggested name for the storage space. This field is REQUIRED.
  // It serves two purposes:
  // 1) Idempotency - This name is generated by the CO to achieve
  //    idempotency.  The Plugin SHOULD ensure that multiple
  //    `CreateVolume` calls for the same name do not result in more
  //    than one piece of storage provisioned corresponding to that
  //    name. If a Plugin is unable to enforce idempotency, the CO's
  //    error recovery logic could result in multiple (unused) volumes
  //    being provisioned.
  //    In the case of error, the CO MUST handle the gRPC error codes
  //    per the recovery behavior defined in the "CreateVolume Errors"
  //    section below.
  //    The CO is responsible for cleaning up volumes it provisioned
  //    that it no longer needs. If the CO is uncertain whether a volume
  //    was provisioned or not when a `CreateVolume` call fails, the CO
  //    MAY call `CreateVolume` again, with the same name, to ensure the
  //    volume exists and to retrieve the volume's `volume_id` (unless
  //    otherwise prohibited by "CreateVolume Errors").
  // 2) Suggested name - Some storage systems allow callers to specify
  //    an identifier by which to refer to the newly provisioned
  //    storage. If a storage system supports this, it can optionally
  //    use this name as the identifier for the new volume.
  // Any Unicode string that conforms to the length limit is allowed
  // except those containing the following banned characters:
  // U+0000-U+0008, U+000B, U+000C, U+000E-U+001F, U+007F-U+009F.
  // (These are control characters other than commonly used whitespace.)
  string name = 1;

  // This field is OPTIONAL. This allows the CO to specify the capacity
  // requirement of the volume to be provisioned. If not specified, the
  // Plugin MAY choose an implementation-defined capacity range. If
  // specified it MUST always be honored, even when creating volumes
  // from a source; which MAY force some backends to internally extend
  // the volume after creating it.
  CapacityRange capacity_range = 2;

  // The capabilities that the provisioned volume MUST have. SP MUST
  // provision a volume that will satisfy ALL of the capabilities
  // specified in this list. Otherwise SP MUST return the appropriate
  // gRPC error code.
  // The Plugin MUST assume that the CO MAY use the provisioned volume
  // with ANY of the capabilities specified in this list.
  // For example, a CO MAY specify two volume capabilities: one with
  // access mode SINGLE_NODE_WRITER and another with access mode
  // MULTI_NODE_READER_ONLY. In this case, the SP MUST verify that the
  // provisioned volume can be used in either mode.
  // This also enables the CO to do early validation: If ANY of the
  // specified volume capabilities are not supported by the SP, the call
  // MUST return the appropriate gRPC error code.
  // This field is REQUIRED.
  repeated VolumeCapability volume_capabilities = 3;

  // Plugin specific parameters passed in as opaque key-value pairs.
  // This field is OPTIONAL. The Plugin is responsible for parsing and
  // validating these parameters. COs will treat these as opaque.
  map<string, string> parameters = 4;

  // Secrets required by plugin to complete volume creation request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 5 [(csi_secret) = true];

  // If specified, the new volume will be pre-populated with data from
  // this source. This field is OPTIONAL.
  VolumeContentSource volume_content_source = 6;

  // Specifies where (regions, zones, racks, etc.) the provisioned
  // volume MUST be accessible from.
  // An SP SHALL advertise the requirements for topological
  // accessibility information in documentation. COs SHALL only specify
  // topological accessibility information supported by the SP.
  // This field is OPTIONAL.
  // This field SHALL NOT be specified unless the SP has the
  // VOLUME_ACCESSIBILITY_CONSTRAINTS plugin capability.
  // If this field is not specified and the SP has the
  // VOLUME_ACCESSIBILITY_CONSTRAINTS plugin capability, the SP MAY
  // choose where the provisioned volume is accessible from.
  TopologyRequirement accessibility_requirements = 7;
}

// Specifies what source the volume will be created from. One of the
// type fields MUST be specified.
message VolumeContentSource {
  message SnapshotSource {
    // Contains identity information for the existing source snapshot.
    // This field is REQUIRED. Plugin is REQUIRED to support creating
    // volume from snapshot if it supports the capability
    // CREATE_DELETE_SNAPSHOT.
    string snapshot_id = 1;
  }

  message VolumeSource {
    // Contains identity information for the existing source volume.
    // This field is REQUIRED. Plugins reporting CLONE_VOLUME
    // capability MUST support creating a volume from another volume.
    string volume_id = 1;
  }

  oneof type {
    SnapshotSource snapshot = 1;
    VolumeSource volume = 2;
  }
}

message CreateVolumeResponse {
  // Contains all attributes of the newly created volume that are
  // relevant to the CO along with information required by the Plugin
  // to uniquely identify the volume. This field is REQUIRED.
  Volume volume = 1;
}

// Specify a capability of a volume.
message VolumeCapability {
  // Indicate that the volume will be accessed via the block device API.
  message BlockVolume {
    // Intentionally empty, for now.
  }

  // Indicate that the volume will be accessed via the filesystem API.
  message MountVolume {
    // The filesystem type. This field is OPTIONAL.
    // An empty string is equal to an unspecified field value.
    string fs_type = 1;

    // The mount options that can be used for the volume. This field is
    // OPTIONAL. `mount_flags` MAY contain sensitive information.
    // Therefore, the CO and the Plugin MUST NOT leak this information
    // to untrusted entities. The total size of this repeated field
    // SHALL NOT exceed 4 KiB.
    repeated string mount_flags = 2;

    // If SP has VOLUME_MOUNT_GROUP node capability and CO provides
    // this field then SP MUST ensure that the volume_mount_group
    // parameter is passed as the group identifier to the underlying
    // operating system mount system call, with the understanding
    // that the set of available mount call parameters and/or
    // mount implementations may vary across operating systems.
    // Additionally, new file and/or directory entries written to
    // the underlying filesystem SHOULD be permission-labeled in such a
    // manner, unless otherwise modified by a workload, that they are
    // both readable and writable by said mount group identifier.
    // This is an OPTIONAL field.
    string volume_mount_group = 3;
  }

  // Specify how a volume can be accessed.
  message AccessMode {
    enum Mode {
      UNKNOWN = 0;

      // Can only be published once as read/write on a single node, at
      // any given time.
      SINGLE_NODE_WRITER = 1;

      // Can only be published once as readonly on a single node, at
      // any given time.
      SINGLE_NODE_READER_ONLY = 2;

      // Can be published as readonly at multiple nodes simultaneously.
      MULTI_NODE_READER_ONLY = 3;

      // Can be published at multiple nodes simultaneously. Only one of
      // the node can be used as read/write. The rest will be readonly.
      MULTI_NODE_SINGLE_WRITER = 4;

      // Can be published as read/write at multiple nodes
      // simultaneously.
      MULTI_NODE_MULTI_WRITER = 5;

      // Can only be published once as read/write at a single workload
      // on a single node, at any given time. SHOULD be used instead of
      // SINGLE_NODE_WRITER for COs using the experimental
      // SINGLE_NODE_MULTI_WRITER capability.
      SINGLE_NODE_SINGLE_WRITER = 6 [(alpha_enum_value) = true];

      // Can be published as read/write at multiple workloads on a
      // single node simultaneously. SHOULD be used instead of
      // SINGLE_NODE_WRITER for COs using the experimental
      // SINGLE_NODE_MULTI_WRITER capability.
      SINGLE_NODE_MULTI_WRITER = 7 [(alpha_enum_value) = true];
    }

    // This field is REQUIRED.
    Mode mode = 1;
  }

  // Specifies what API the volume will be accessed using. One of the
  // following fields MUST be specified.
  oneof access_type {
    BlockVolume block = 1;
    MountVolume mount = 2;
  }

  // This is a REQUIRED field.
  AccessMode access_mode = 3;
}

// The capacity of the storage space in bytes. To specify an exact size,
// `required_bytes` and `limit_bytes` SHALL be set to the same value. At
// least one of the these fields MUST be specified.
message CapacityRange {
  // Volume MUST be at least this big. This field is OPTIONAL.
  // A value of 0 is equal to an unspecified field value.
  // The value of this field MUST NOT be negative.
  int64 required_bytes = 1;

  // Volume MUST not be bigger than this. This field is OPTIONAL.
  // A value of 0 is equal to an unspecified field value.
  // The value of this field MUST NOT be negative.
  int64 limit_bytes = 2;
}

// Information about a specific volume.
message Volume {
  // The capacity of the volume in bytes. This field is OPTIONAL. If not
  // set (value of 0), it indicates that the capacity of the volume is
  // unknown (e.g., NFS share).
  // The value of this field MUST NOT be negative.
  int64 capacity_bytes = 1;

  // The identifier for this volume, generated by the plugin.
  // This field is REQUIRED.
  // This field MUST contain enough information to uniquely identify
  // this specific volume vs all other volumes supported by this plugin.
  // This field SHALL be used by the CO in subsequent calls to refer to
  // this volume.
  // The SP is NOT responsible for global uniqueness of volume_id across
  // multiple SPs.
  string volume_id = 2;

  // Opaque static properties of the volume. SP MAY use this field to
  // ensure subsequent volume validation and publishing calls have
  // contextual information.
  // The contents of this field SHALL be opaque to a CO.
  // The contents of this field SHALL NOT be mutable.
  // The contents of this field SHALL be safe for the CO to cache.
  // The contents of this field SHOULD NOT contain sensitive
  // information.
  // The contents of this field SHOULD NOT be used for uniquely
  // identifying a volume. The `volume_id` alone SHOULD be sufficient to
  // identify the volume.
  // A volume uniquely identified by `volume_id` SHALL always report the
  // same volume_context.
  // This field is OPTIONAL and when present MUST be passed to volume
  // validation and publishing calls.
  map<string, string> volume_context = 3;

  // If specified, indicates that the volume is not empty and is
  // pre-populated with data from the specified source.
  // This field is OPTIONAL.
  VolumeContentSource content_source = 4;

  // Specifies where (regions, zones, racks, etc.) the provisioned
  // volume is accessible from.
  // A plugin that returns this field MUST also set the
  // VOLUME_ACCESSIBILITY_CONSTRAINTS plugin capability.
  // An SP MAY specify multiple topologies to indicate the volume is
  // accessible from multiple locations.
  // COs MAY use this information along with the topology information
  // returned by NodeGetInfo to ensure that a given volume is accessible
  // from a given node when scheduling workloads.
  // This field is OPTIONAL. If it is not specified, the CO MAY assume
  // the volume is equally accessible from all nodes in the cluster and
  // MAY schedule workloads referencing the volume on any available
  // node.
  //
  // Example 1:
  //   accessible_topology = {"region": "R1", "zone": "Z2"}
  // Indicates a volume accessible only from the "region" "R1" and the
  // "zone" "Z2".
  //
  // Example 2:
  //   accessible_topology =
  //     {"region": "R1", "zone": "Z2"},
  //     {"region": "R1", "zone": "Z3"}
  // Indicates a volume accessible from both "zone" "Z2" and "zone" "Z3"
  // in the "region" "R1".
  repeated Topology accessible_topology = 5;
}

message TopologyRequirement {
  // Specifies the list of topologies the provisioned volume MUST be
  // accessible from.
  // This field is OPTIONAL. If TopologyRequirement is specified either
  // requisite or preferred or both MUST be specified.
  //
  // If requisite is specified, the provisioned volume MUST be
  // accessible from at least one of the requisite topologies.
  //
  // Given
  //   x = number of topologies provisioned volume is accessible from
  //   n = number of requisite topologies
  // The CO MUST ensure n >= 1. The SP MUST ensure x >= 1
  // If x==n, then the SP MUST make the provisioned volume available to
  // all topologies from the list of requisite topologies. If it is
  // unable to do so, the SP MUST fail the CreateVolume call.
  // For example, if a volume should be accessible from a single zone,
  // and requisite =
  //   {"region": "R1", "zone": "Z2"}
  // then the provisioned volume MUST be accessible from the "region"
  // "R1" and the "zone" "Z2".
  // Similarly, if a volume should be accessible from two zones, and
  // requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"}
  // then the provisioned volume MUST be accessible from the "region"
  // "R1" and both "zone" "Z2" and "zone" "Z3".
  //
  // If x<n, then the SP SHALL choose x unique topologies from the list
  // of requisite topologies. If it is unable to do so, the SP MUST fail
  // the CreateVolume call.
  // For example, if a volume should be accessible from a single zone,
  // and requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"}
  // then the SP may choose to make the provisioned volume available in
  // either the "zone" "Z2" or the "zone" "Z3" in the "region" "R1".
  // Similarly, if a volume should be accessible from two zones, and
  // requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"},
  //   {"region": "R1", "zone": "Z4"}
  // then the provisioned volume MUST be accessible from any combination
  // of two unique topologies: e.g. "R1/Z2" and "R1/Z3", or "R1/Z2" and
  //  "R1/Z4", or "R1/Z3" and "R1/Z4".
  //
  // If x>n, then the SP MUST make the provisioned volume available from
  // all topologies from the list of requisite topologies and MAY choose
  // the remaining x-n unique topologies from the list of all possible
  // topologies. If it is unable to do so, the SP MUST fail the
  // CreateVolume call.
  // For example, if a volume should be accessible from two zones, and
  // requisite =
  //   {"region": "R1", "zone": "Z2"}
  // then the provisioned volume MUST be accessible from the "region"
  // "R1" and the "zone" "Z2" and the SP may select the second zone
  // independently, e.g. "R1/Z4".
  repeated Topology requisite = 1;

  // Specifies the list of topologies the CO would prefer the volume to
  // be provisioned in.
  //
  // This field is OPTIONAL. If TopologyRequirement is specified either
  // requisite or preferred or both MUST be specified.
  //
  // An SP MUST attempt to make the provisioned volume available using
  // the preferred topologies in order from first to last.
  //
  // If requisite is specified, all topologies in preferred list MUST
  // also be present in the list of requisite topologies.
  //
  // If the SP is unable to to make the provisioned volume available
  // from any of the preferred topologies, the SP MAY choose a topology
  // from the list of requisite topologies.
  // If the list of requisite topologies is not specified, then the SP
  // MAY choose from the list of all possible topologies.
  // If the list of requisite topologies is specified and the SP is
  // unable to to make the provisioned volume available from any of the
  // requisite topologies it MUST fail the CreateVolume call.
  //
  // Example 1:
  // Given a volume should be accessible from a single zone, and
  // requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"}
  // preferred =
  //   {"region": "R1", "zone": "Z3"}
  // then the SP SHOULD first attempt to make the provisioned volume
  // available from "zone" "Z3" in the "region" "R1" and fall back to
  // "zone" "Z2" in the "region" "R1" if that is not possible.
  //
  // Example 2:
  // Given a volume should be accessible from a single zone, and
  // requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"},
  //   {"region": "R1", "zone": "Z4"},
  //   {"region": "R1", "zone": "Z5"}
  // preferred =
  //   {"region": "R1", "zone": "Z4"},
  //   {"region": "R1", "zone": "Z2"}
  // then the SP SHOULD first attempt to make the provisioned volume
  // accessible from "zone" "Z4" in the "region" "R1" and fall back to
  // "zone" "Z2" in the "region" "R1" if that is not possible. If that
  // is not possible, the SP may choose between either the "zone"
  // "Z3" or "Z5" in the "region" "R1".
  //
  // Example 3:
  // Given a volume should be accessible from TWO zones (because an
  // opaque parameter in CreateVolumeRequest, for example, specifies
  // the volume is accessible from two zones, aka synchronously
  // replicated), and
  // requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"},
  //   {"region": "R1", "zone": "Z4"},
  //   {"region": "R1", "zone": "Z5"}
  // preferred =
  //   {"region": "R1", "zone": "Z5"},
  //   {"region": "R1", "zone": "Z3"}
  // then the SP SHOULD first attempt to make the provisioned volume
  // accessible from the combination of the two "zones" "Z5" and "Z3" in
  // the "region" "R1". If that's not possible, it should fall back to
  // a combination of "Z5" and other possibilities from the list of
  // requisite. If that's not possible, it should fall back  to a
  // combination of "Z3" and other possibilities from the list of
  // requisite. If that's not possible, it should fall back  to a
  // combination of other possibilities from the list of requisite.
  repeated Topology preferred = 2;
}

// Topology is a map of topological domains to topological segments.
// A topological domain is a sub-division of a cluster, like "region",
// "zone", "rack", etc.
// A topological segment is a specific instance of a topological domain,
// like "zone3", "rack3", etc.
// For example {"com.company/zone": "Z1", "com.company/rack": "R3"}
// Valid keys have two segments: an OPTIONAL prefix and name, separated
// by a slash (/), for example: "com.company.example/zone".
// The key name segment is REQUIRED. The prefix is OPTIONAL.
// The key name MUST be 63 characters or less, begin and end with an
// alphanumeric character ([a-z0-9A-Z]), and contain only dashes (-),
// underscores (_), dots (.), or alphanumerics in between, for example
// "zone".
// The key prefix MUST be 63 characters or less, begin and end with a
// lower-case alphanumeric character ([a-z0-9]), contain only
// dashes (-), dots (.), or lower-case alphanumerics in between, and
// follow domain name notation format
// (https://tools.ietf.org/html/rfc1035#section-2.3.1).
// The key prefix SHOULD include the plugin's host company name and/or
// the plugin name, to minimize the possibility of collisions with keys
// from other plugins.
// If a key prefix is specified, it MUST be identical across all
// topology keys returned by the SP (across all RPCs).
// Keys MUST be case-insensitive. Meaning the keys "Zone" and "zone"
// MUST not both exist.
// Each value (topological segment) MUST contain 1 or more strings.
// Each string MUST be 63 characters or less and begin and end with an
// alphanumeric character with '-', '_', '.', or alphanumerics in
// between.
message Topology {
  map<string, string> segments = 1;
}
```

##### CreateVolume エラー

プラグインが無事 CreateVolume コールを完了する事ができない場合、プラグインは gRPC ステータス中で non-ok gRPC コードを返却しなければならない(MUST)。
以下で定義された状況に遭遇した場合、プラグインは指定された gRPC エラーコードを返却しなければならない(MUST)。
CO は、この gRPC エラーコードに遭遇した場合に指定されたエラー復旧動作を実装しなければならない(MUST)。

| 状況 | gRPC コード | 説明 | 復旧動作 |
|-----------|-----------|-------------|-------------------|
| ソースが非互換か未サポート | 3 INVALID_ARGUMENT | 一般的なケース以外にも、本コードは CREATE_DELETE_VOLUME をサポートするプラグインが要求されたソース(`SnapshotSource` 又は `VolumeSource`)からボリュームを作成できない場合を示す為に使用されなければならない(MUST)。障害はソースをサポートしていない(CO がそのソースを提供してこなかったべきでない(SHOULD NOT))、あるいはソースからの `parameters` と新しいボリューム用に要求されたパラメータの間に互換性がない事に起因する可能性がある(MAY)。より人間が読みやすい情報は、問題がソースなのであれば gRPC `status.message` フィールド内で提供されるべきである(SHOULD)。 | ソース関連の問題の場合、呼び出し元は異なるパラメータ、異なるソース、あるいはソースなしを使用しなければならない(MUST)。 |
| ソースが存在しない | 5 NOT_FOUND | 指定されたソースが存在しない事を示す。 | 呼び出し元は、急激に時間間隔を伸ばす再試行を行う前に、`volume_content_source` が正しい事、ソースがアクセス可能である事、ソースがまだ削除されていない事を確認しなければならない(MUST)。 |
| ボリュームは既に存在するが非互換 | 6 ALREADY_EXISTS | 指定されたボリューム`name`に関連するボリュームが既に存在するが、指定された `capacity_range`、`volume_capabilities`、`parameters`、`accessibility_requirements`、`volume_content_source` と互換性がない事を示す。 | 呼び出し元は再試行前に引数を修正するか、異なる`name`を使用しなければならない(MUST)。 |
| `accessible_topology` 中で用意できない | 8 RESOURCE_EXHAUSTED | `accessible_topology` フィールドが有効であるにも関わらず、指定されたトポロジ成約を用いて新しいボリュームを用意できない事を示す。gRPC `status.message` フィールド内でより人間が読みやすい情報が提供されても良い(MAY)。 | 呼び出し元は、急激に時間間隔を伸ばす再試行を行う前に、指定された場所内でボリュームを阻害するあらゆるもの(例：クォータ問題)が対処されている事を確認しなければならない(MUST)。 |
| `capacity_range` が未サポート | 11 OUT_OF_RANGE | 容量範囲がプラグインで許可されていない事 (例：ソースのスナップショットより小さいボリューム作成を試みた際、又はプラグインがソースのスナップショットより大きいボリューム作成をサポートしない場合)を示す。gRPC `status.message` フィールド内でより人間が読みやすい情報が提供されても良い(MAY)。 | 呼び出し元は再試行前に容量範囲を修正しなければならない(MUST)。 |


#### `DeleteVolume`

`CREATE_DELETE_VOLUME` 機能がある場合、コントローラープラグインは本 RPC コールを実装しなければならない(MUST)。
ボリュームを削除する為、本 RPC は CO によリコールされます。

本操作は冪等性がなければならない(MUST)。
指定された `volume_id` に関連するボリュームが存在しないか、ボリューに関連するアーティファクトが存在しない場合、プラグインは `0 OK` を返却しなければならない(MUST)。

CSI プラグインはスナップショットから独立してボリュームを扱うべきである(SHOULD)。

コントローラープラグインが既存のスナップショットに影響しないボリューム削除をサポートする場合、一旦ボリュームが削除された後、`ListSnapshot` コールで表示されるこれらのスナップショットは依然完全に機能し、新しいボリューム用のソースとして許容されなければならない(MUST)。

コントローラープラグインが既存のスナップショットに影響しないボリューム削除をサポートする場合、(削除された)ボリュームは要求によるあらゆる手段で変更されるべきでなく(MUST NOT)、操作は `FAILED_PRECONDITION` エラーコードを返却しなければならなず(must)、意味のある人間が読みやすい情報が `status.message` フィールド中に含まれても良い(MAY)。

```protobuf
message DeleteVolumeRequest {
  // The ID of the volume to be deprovisioned.
  // This field is REQUIRED.
  string volume_id = 1;

  // Secrets required by plugin to complete volume deletion request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 2 [(csi_secret) = true];
}

message DeleteVolumeResponse {
  // Intentionally empty.
}
```

##### DeleteVolume エラー

プラグインが DeleteVolume コールを無事完了できない場合、プラグインは gRPC ステータス中で non-ok gRPC コードを返却しなければならない(MUST)。
下記で定義された状況に遭遇した場合、プラグインは指定された gRPC エラーコードを返却しなければならない(MUST)。
CO は、gRPC エラーコードに遭遇した際の指定されたエラー復旧動作を実装しなければならない(MUST)。

| 状況 | gRPC コード | 説明 | 復旧動作 |
|-----------|-----------|-------------|-------------------|
| ボリュームが使用中 | 9 FAILED_PRECONDITION | 指定された `volume_id` に関連するボリュームが別のリソースで使用中か、スナップショットがありプラグインがボリュームとスナップショットを独立して扱わない為に、プラグインがそのボリュームを削除できない事を示す。 | 呼び出し元は、ボリュームを使用している他のリソースがない事、スナップショットがない事を保証し、その後急激に時間間隔を伸ばす再試行を行うべきである(SHOULD)。 |


#### `ControllerPublishVolume`

コントローラープラグインは、`PUBLISH_UNPUBLISH_VOLUME` コントローラー機能を持つ場合、この RPC コール実装しなければならない(MUST)。
ボリュームを使用するワークロードをノード上に配置する際、CO によってこの RPC は コールされる。
プラグインは、与えられたノード上で利用可能なボリュームを作成する為に必要な作業を実行するべきである(SHOULD)。
プラグインは、本 RPC がボリュームが使用されるノード上で実行される事を前提にしてはならない(MUST NOT)。

本操作は冪等性を持たなければならない(MUST)。
`volume_id` に関連付けられたボリュームが `node_id` に関連付けられたノード上で既に公開されて(published)おり、指定された `volume_capability` と `readonly` フラグに互換性がある場合、プラグインは `0 OK` を返却しなければならない(MUST)。

操作が失敗するか、CO は操作が失敗したかしなかったかが分からない場合、CO は `ControllerPublishVolume` を再度コールする事を選択するか、`ControllerUnpublishVolume` をコールする事を選択しても良い(MAY)。

CO は、ボリュームが `MULTI_NODE` 機能(つまり、`MULTI_NODE_READER_ONLY`、`MULTI_NODE_SINGLE_WRITER`、`MULTI_NODE_MULTI_WRITER`)を持つ場合、複数ノードにボリュームを公開する為に本 RPC をコールしても良い(MAY)。

```protobuf
message ControllerPublishVolumeRequest {
  // The ID of the volume to be used on a node.
  // This field is REQUIRED.
  string volume_id = 1;

  // The ID of the node. This field is REQUIRED. The CO SHALL set this
  // field to match the node ID returned by `NodeGetInfo`.
  string node_id = 2;

  // Volume capability describing how the CO intends to use this volume.
  // SP MUST ensure the CO can use the published volume as described.
  // Otherwise SP MUST return the appropriate gRPC error code.
  // This is a REQUIRED field.
  VolumeCapability volume_capability = 3;

  // Indicates SP MUST publish the volume in readonly mode.
  // CO MUST set this field to false if SP does not have the
  // PUBLISH_READONLY controller capability.
  // This is a REQUIRED field.
  bool readonly = 4;

  // Secrets required by plugin to complete controller publish volume
  // request. This field is OPTIONAL. Refer to the
  // `Secrets Requirements` section on how to use this field.
  map<string, string> secrets = 5 [(csi_secret) = true];

  // Volume context as returned by SP in
  // CreateVolumeResponse.Volume.volume_context.
  // This field is OPTIONAL and MUST match the volume_context of the
  // volume identified by `volume_id`.
  map<string, string> volume_context = 6;
}

message ControllerPublishVolumeResponse {
  // Opaque static publish properties of the volume. SP MAY use this
  // field to ensure subsequent `NodeStageVolume` or `NodePublishVolume`
  // calls calls have contextual information.
  // The contents of this field SHALL be opaque to a CO.
  // The contents of this field SHALL NOT be mutable.
  // The contents of this field SHALL be safe for the CO to cache.
  // The contents of this field SHOULD NOT contain sensitive
  // information.
  // The contents of this field SHOULD NOT be used for uniquely
  // identifying a volume. The `volume_id` alone SHOULD be sufficient to
  // identify the volume.
  // This field is OPTIONAL and when present MUST be passed to
  // subsequent `NodeStageVolume` or `NodePublishVolume` calls
  map<string, string> publish_context = 1;
}
```

##### ControllerPublishVolume エラー

プラグインが ControllerPublishVolume コールを無事完了できない場合、プラグインは gRPC ステータス中で non-ok gRPC コードを返却しなければならない(MUST)。
下記で定義された状況に遭遇した場合、プラグインは指定された gRPC エラーコードを返却しなければならない(MUST)。
CO は、gRPC エラーコードに遭遇した際の指定されたエラー復旧動作を実装しなければならない(MUST)。

| 状況 | gRPC コード | 説明 | 復旧動作 |
|-----------|-----------|-------------|-------------------|
| ボリュームが存在しない | 5 NOT_FOUND | 指定された `volume_id` に関連付けられたボリュームが存在しない事を示す。 | 呼び出し元は、急激に時間間隔を伸ばす再試行を行う前に `volume_id` が正しい事、ボリュームがアクセス可能で、ボリュームが削除されていない事を確認しなければならない(MUST)。 |
| ノードが存在しない | 5 NOT_FOUND | 指定された `node_id` に関連付けられたノードが存在しない事を示す。 | 呼び出し元は、急激に時間間隔を伸ばす再試行を行う前に、`node_id` が正しい事、ノードが利用可能で、ノードが停止又は削除されていない事を確認しなければならない(MUST)。 |
| ボリュームは公開されているが互換性がない | 6 ALREADY_EXISTS | 指定された `volume_id` が関連付けられたボリュームが指定された `node_id` が関連付けられたノードに対し既に公開されている(published)が、指定された `volume_capability` あるいは `readonly` フラグと互換性がない事を示す。 | 呼び出し元は再試行前に引数を修正しなければならない(MUST)。 |
| ボリュームは別ノードに公開されている | 9 FAILED_PRECONDITION | 指定された `volume_id` に関連付けられたボリュームが別ノードに対して既に公開されて(published)おり、ボリュームが MULTI_NODE ボリューム機能を持たない事を示す。本エラーコードが返却された場合、プラグインは gRPC `status.message` の一部として、ボリュームが公開されているノードの `node_id` を指定すべきである(SHOULD)。 | 急激に時間間隔を伸ばす再試行を行う前に、呼び出し元は指定されたボリュームは他のどのノードに対しても公開されていない事を確認しなければならない(SHOULD)。 |
| 最大アタッチボリューム数 | 8 RESOURCE_EXHAUSTED | 特定ノードにアタッチ可能なボリューム最大数が既にアタッチ済みである事を示す。それゆえ、少なくとも１つの既存アタッチ済みボリュームがノードからデタッチされるまで本操作は失敗する。| 急激に時間間隔を伸ばす再試行を行う前に、呼び出し元はノードにアタッチ済みのボリューム数がボリューム最大数より少ない事を確認しなければならない(MUST)。 |

#### `ControllerUnpublishVolume`

`PUBLISH_UNPUBLISH_VOLUME` コントローラー機能を持つ場合、コントローラープラグインは本 RPC コールを実装しなければならない(MUST)
本 RPC は `ControllerPublishVolume` の逆操作である。
本 RPC は、当該ボリュームにおける全ての  `NodeUnstageVolume` と `NodeUnpublishVolume` がコールされ、成功した後に呼ばれなければならない(MUST)。
プラグインは、当該ボリュームが別ノードで使用可能状態にする為に必要な作業を実行すべきである(SHOULD)。
プラグインは、当該ボリュームが以前使用されていたノード上で本 RPC が実行されたとみなすべきではない(MUST NOT)。

本 RPC は、通常当該ボリュームを使用するワークロードが異なるノード上に移動される際、あるいはノード上で当該ボリュームを使用する全ワークロードが終了する際に CO によりコールされる。

本操作は冪等性を持つべきである(MUST)。
`volume_id` に関連付けられた当該ボリュームが `node_id` に関連付けられたノードにアタッチされていない場合、プラグインは `0 OK` を返却しなければならない(MUST)。
`volume_id` に関連付けられたボリュームあるいは `node_id` に関連付けられたノードがプラグインにより見つからず、ボリュームがノード上で安全に ControllerUnpublished されたと見なされる場合、プラグインは `0 OK` を返却すべきである(SHOULD)。
本操作が失敗した、あるいは本操作が失敗したかどうかを CO が把握していない場合、CO は `ControllerUnpublishVolume` を再コールしても良い。

```protobuf
message ControllerUnpublishVolumeRequest {
  // The ID of the volume. This field is REQUIRED.
  string volume_id = 1;

  // The ID of the node. This field is OPTIONAL. The CO SHOULD set this
  // field to match the node ID returned by `NodeGetInfo` or leave it
  // unset. If the value is set, the SP MUST unpublish the volume from
  // the specified node. If the value is unset, the SP MUST unpublish
  // the volume from all nodes it is published to.
  string node_id = 2;

  // Secrets required by plugin to complete controller unpublish volume
  // request. This SHOULD be the same secrets passed to the
  // ControllerPublishVolume call for the specified volume.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 3 [(csi_secret) = true];
}

message ControllerUnpublishVolumeResponse {
  // Intentionally empty.
}
```

##### ControllerUnpublishVolume エラー

プラグインが ControllerUnpublishVolume コールを無事完了できない場合、プラグインは gRPC ステータス中で non-ok gRPC コードを返却しなければならない(MUST)。
下記で定義された状況に遭遇した場合、プラグインは指定された gRPC エラーコードを返却しなければならない(MUST)。
CO は、gRPC エラーコードに遭遇した際の指定されたエラー復旧動作を実装しなければならない(MUST)。


| 状況 | gRPC コード | 説明 | 復旧動作 |
|-----------|-----------|-------------|-------------------|
| ボリュームが存在せず、ボリュームがノードから ControllerUnpublished されたと見なせない | 5 NOT_FOUND | 指定された `volume_id` に関連付けられたボリュームが存在せず、指定された `node_id` に関連付けられたノードから ControllerUnpublished されたとも見なせない事を示す。 | 急激に時間間隔を伸ばす再試行を行う前に、呼び出し元は `volume_id` が正しく、当該ボリュームがアクセス可能で削除されていない事を検証すべきである(SHOULD)。 |
| ノードが存在せず、ボリュームがノードから ControllerUnpublished されたと見なせない | 5 NOT_FOUND | 指定された `node_id` に関連付けられたノードが存在せず、指定された `volume_id` に関連付けられたボリュームがノードから ControllerUnpublished されたとも見なせない事を示す。 | 急激に時間間隔を伸ばす再試行を行う前に、呼び出し元は `node_id` が正しく、当該ノードが利用可能で停止・削除されていない事を検証すべきである(SHOULD)。 |


#### `ValidateVolumeCapabilities`

コントローラープラグインは本 RPC コールを実装しなければならない(MUST)。
本 RPC は事前にプロビジョンされたボリュームが CO の求める全ケーパビリティを持っているかどうか確認する為に CO によりコールされる。
本 RPC コールは、要求で指定されたのボリュームケーパビリティが全てサポートされる場合のみ `confirmed` を返却すべきである(SHALL)(下記警告参照)。
本操作は冪等性を持つべきである(MUST)。

注意: 古いプラグインは、CSI protobuf のより新しい後方互換バージョンを用いて通信している CO により送信された機能検証メッセージ(とサブメッセージ)中に存在しても良い(MAY)新しい「process」フィールドをパースはするがおそらく「処理」はしないだろう。
したがって、CO は元々要求されたケーパビリティと検証されたケーパビリティを比較する事で、成功したケーパビリティ検証応答をすべきである(SHALL)。

```protobuf
message ValidateVolumeCapabilitiesRequest {
  // The ID of the volume to check. This field is REQUIRED.
  string volume_id = 1;

  // Volume context as returned by SP in
  // CreateVolumeResponse.Volume.volume_context.
  // This field is OPTIONAL and MUST match the volume_context of the
  // volume identified by `volume_id`.
  map<string, string> volume_context = 2;

  // The capabilities that the CO wants to check for the volume. This
  // call SHALL return "confirmed" only if all the volume capabilities
  // specified below are supported. This field is REQUIRED.
  repeated VolumeCapability volume_capabilities = 3;

  // See CreateVolumeRequest.parameters.
  // This field is OPTIONAL.
  map<string, string> parameters = 4;

  // Secrets required by plugin to complete volume validation request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 5 [(csi_secret) = true];
}

message ValidateVolumeCapabilitiesResponse {
  message Confirmed {
    // Volume context validated by the plugin.
    // This field is OPTIONAL.
    map<string, string> volume_context = 1;

    // Volume capabilities supported by the plugin.
    // This field is REQUIRED.
    repeated VolumeCapability volume_capabilities = 2;

    // The volume creation parameters validated by the plugin.
    // This field is OPTIONAL.
    map<string, string> parameters = 3;
  }

  // Confirmed indicates to the CO the set of capabilities that the
  // plugin has validated. This field SHALL only be set to a non-empty
  // value for successful validation responses.
  // For successful validation responses, the CO SHALL compare the
  // fields of this message to the originally requested capabilities in
  // order to guard against an older plugin reporting "valid" for newer
  // capability fields that it does not yet understand.
  // This field is OPTIONAL.
  Confirmed confirmed = 1;

  // Message to the CO if `confirmed` above is empty. This field is
  // OPTIONAL.
  // An empty string is equal to an unspecified field value.
  string message = 2;
}
```

##### ValidateVolumeCapabilities エラー

プラグインが ValidateVolumeCapabilities コールを無事完了できない場合、プラグインは gRPC ステータス中で non-ok gRPC コードを返却しなければならない(MUST)。
下記で定義された状況に遭遇した場合、プラグインは指定された gRPC エラーコードを返却しなければならない(MUST)。
CO は、gRPC エラーコードに遭遇した際の指定されたエラー復旧動作を実装しなければならない(MUST)。

| 状況 | gRPC コード | 説明 | 復旧動作 |
|-----------|-----------|-------------|-------------------|
| ボリュームが存在しない | 5 NOT_FOUND | 指定された `volume_id` に関連付けられたボリュームが存在しない事を示す。 | 急激に時間間隔を伸ばす再試行を行う前に、呼び出し元は `volume_id` が正しく、ボリュームがアクセス可能で削除されていない事を検証しなければならない(MUST)。 |


#### `ListVolumes`

`LIST_VOLUMES` ケーパビリティを持つコントローラープラグインは本 RPC コールを実装しなければならない。
プラグインは自身が把握している全ボリュームに関する情報を返却すべきである(SHALL)。
CO が並行して `ListVolumes` の結果をページ単位にしている間にボリュームが作成/削除された場合、CO はリスト中に重複したボリュームを証言するか、既存のボリュームを証言しないか、その両方を行うかのいずれでも良い(MAY)。
`ListVolumes` の複数コールを介してボリューム一覧をページングする際、CO は全ボリュームの一貫した「ビュー」を想定すべきではない(SHALL NOT)。

```protobuf
message ListVolumesRequest {
  // If specified (non-zero value), the Plugin MUST NOT return more
  // entries than this number in the response. If the actual number of
  // entries is more than this number, the Plugin MUST set `next_token`
  // in the response which can be used to get the next page of entries
  // in the subsequent `ListVolumes` call. This field is OPTIONAL. If
  // not specified (zero value), it means there is no restriction on the
  // number of entries that can be returned.
  // The value of this field MUST NOT be negative.
  int32 max_entries = 1;

  // A token to specify where to start paginating. Set this field to
  // `next_token` returned by a previous `ListVolumes` call to get the
  // next page of entries. This field is OPTIONAL.
  // An empty string is equal to an unspecified field value.
  string starting_token = 2;
}

message ListVolumesResponse {
  message VolumeStatus{
    // A list of all `node_id` of nodes that the volume in this entry
    // is controller published on.
    // This field is OPTIONAL. If it is not specified and the SP has
    // the LIST_VOLUMES_PUBLISHED_NODES controller capability, the CO
    // MAY assume the volume is not controller published to any nodes.
    // If the field is not specified and the SP does not have the
    // LIST_VOLUMES_PUBLISHED_NODES controller capability, the CO MUST
    // not interpret this field.
    // published_node_ids MAY include nodes not published to or
    // reported by the SP. The CO MUST be resilient to that.
    repeated string published_node_ids = 1;

    // Information about the current condition of the volume.
    // This field is OPTIONAL.
    // This field MUST be specified if the
    // VOLUME_CONDITION controller capability is supported.
    VolumeCondition volume_condition = 2 [(alpha_field) = true];
  }

  message Entry {
    // This field is REQUIRED
    Volume volume = 1;

    // This field is OPTIONAL. This field MUST be specified if the
    // LIST_VOLUMES_PUBLISHED_NODES controller capability is
    // supported.
    VolumeStatus status = 2;
  }

  repeated Entry entries = 1;

  // This token allows you to get the next page of entries for
  // `ListVolumes` request. If the number of entries is larger than
  // `max_entries`, use the `next_token` as a value for the
  // `starting_token` field in the next `ListVolumes` request. This
  // field is OPTIONAL.
  // An empty string is equal to an unspecified field value.
  string next_token = 2;
}
```

##### ListVolumes エラー

プラグインが ListVolumes コールを無事完了できない場合、プラグインは gRPC ステータス中で non-ok gRPC コードを返却しなければならない(MUST)。
下記で定義された状況に遭遇した場合、プラグインは指定された gRPC エラーコードを返却しなければならない(MUST)。
CO は、gRPC エラーコードに遭遇した際の指定されたエラー復旧動作を実装しなければならない(MUST)。

| 状況 | gRPC コード | 説明 | 復旧動作 |
|-----------|-----------|-------------|-------------------|
| `starting_token` 無効 | 10 ABORTED | `starting_token` が有効でない事を示す。 | 呼び出し元は空の `starting_token` を用いて `ListVolumes` 操作を再び実行すべきである(SHOULD)。 |

#### `ControllerGetVolume`

**α版機能**

オプションの本 RPC は、1ボリュームの現在の情報を取得するために CO によってコールされる。

`GET_VOLUME` ケーパビリティを持つ場合、コントローラープラグインはこの `ControllerGetVolume` RPC コールを実装しなければならない(MUST)。

`VOLUME_CONDITION` ケーパビリティを保つ場合、コントローラープラグインは `ControllerGetVolumeResponse` 中の空でない `volume_condition` フィールドを提供しなければならない(MUST)。

ボリュームが存在する場合、`ControllerGetVolumeResponse` はボリュームの現在の情報を含むべきである(SHOULD)。
ボリュームが最早存在しない場合、 `ControllerGetVolume` は gRPC エラーコード `NOT_FOUND` を返却すべきである(should)。

```protobuf
message ControllerGetVolumeRequest {
  option (alpha_message) = true;

  // The ID of the volume to fetch current volume information for.
  // This field is REQUIRED.
  string volume_id = 1;
}

message ControllerGetVolumeResponse {
  option (alpha_message) = true;

  message VolumeStatus{
    // A list of all the `node_id` of nodes that this volume is
    // controller published on.
    // This field is OPTIONAL.
    // This field MUST be specified if the LIST_VOLUMES_PUBLISHED_NODES
    // controller capability is supported.
    // published_node_ids MAY include nodes not published to or
    // reported by the SP. The CO MUST be resilient to that.
    repeated string published_node_ids = 1;

    // Information about the current condition of the volume.
    // This field is OPTIONAL.
    // This field MUST be specified if the
    // VOLUME_CONDITION controller capability is supported.
    VolumeCondition volume_condition = 2;
  }

  // This field is REQUIRED
  Volume volume = 1;

  // This field is REQUIRED.
  VolumeStatus status = 2;
}
```

##### ControllerGetVolume エラー

プラグインが ControllerGetVolume コールを無事完了できない場合、プラグインは gRPC ステータス中で non-ok gRPC コードを返却しなければならない(MUST)。
下記で定義された状況に遭遇した場合、プラグインは指定された gRPC エラーコードを返却しなければならない(MUST)。
CO は、gRPC エラーコードに遭遇した際の指定されたエラー復旧動作を実装しなければならない(MUST)。

| 状況 | gRPC コード | 説明 | 復旧動作 |
|-----------|-----------|-------------|-------------------|
| ボリュームが存在しない | 5 NOT_FOUND | 指定された `volume_id` に関連付けられたボリュームが存在しない事を示す。 | 急激に時間間隔を伸ばす再試行を行う前に、呼び出し元は `volume_id` が正しい事、ボリュームがアクセス可能で削除されていない事を検証しなければならない(MUST)。 |

#### `GetCapacity`

`GET_CAPACITY` コントローラーケーパビリティを保つ場合、コントローラープラグインは本 RPC コールを実装しなければならない。
本 RPC は、コントローラーがボリュームをプロビジョンするストレージプールの容量を CO が問い合わせられるようにする。

```protobuf
message GetCapacityRequest {
  // If specified, the Plugin SHALL report the capacity of the storage
  // that can be used to provision volumes that satisfy ALL of the
  // specified `volume_capabilities`. These are the same
  // `volume_capabilities` the CO will use in `CreateVolumeRequest`.
  // This field is OPTIONAL.
  repeated VolumeCapability volume_capabilities = 1;

  // If specified, the Plugin SHALL report the capacity of the storage
  // that can be used to provision volumes with the given Plugin
  // specific `parameters`. These are the same `parameters` the CO will
  // use in `CreateVolumeRequest`. This field is OPTIONAL.
  map<string, string> parameters = 2;

  // If specified, the Plugin SHALL report the capacity of the storage
  // that can be used to provision volumes that in the specified
  // `accessible_topology`. This is the same as the
  // `accessible_topology` the CO returns in a `CreateVolumeResponse`.
  // This field is OPTIONAL. This field SHALL NOT be set unless the
  // plugin advertises the VOLUME_ACCESSIBILITY_CONSTRAINTS capability.
  Topology accessible_topology = 3;
}

message GetCapacityResponse {
  // The available capacity, in bytes, of the storage that can be used
  // to provision volumes. If `volume_capabilities` or `parameters` is
  // specified in the request, the Plugin SHALL take those into
  // consideration when calculating the available capacity of the
  // storage. This field is REQUIRED.
  // The value of this field MUST NOT be negative.
  int64 available_capacity = 1;

  // The largest size that may be used in a
  // CreateVolumeRequest.capacity_range.required_bytes field
  // to create a volume with the same parameters as those in
  // GetCapacityRequest.
  //
  // If `volume_capabilities` or `parameters` is
  // specified in the request, the Plugin SHALL take those into
  // consideration when calculating the minimum volume size of the
  // storage.
  //
  // This field is OPTIONAL. MUST NOT be negative.
  // The Plugin SHOULD provide a value for this field if it has
  // a maximum size for individual volumes and leave it unset
  // otherwise. COs MAY use it to make decision about
  // where to create volumes.
  google.protobuf.Int64Value maximum_volume_size = 2;

  // The smallest size that may be used in a
  // CreateVolumeRequest.capacity_range.limit_bytes field
  // to create a volume with the same parameters as those in
  // GetCapacityRequest.
  //
  // If `volume_capabilities` or `parameters` is
  // specified in the request, the Plugin SHALL take those into
  // consideration when calculating the maximum volume size of the
  // storage.
  //
  // This field is OPTIONAL. MUST NOT be negative.
  // The Plugin SHOULD provide a value for this field if it has
  // a minimum size for individual volumes and leave it unset
  // otherwise. COs MAY use it to make decision about
  // where to create volumes.
  google.protobuf.Int64Value minimum_volume_size = 3
    [(alpha_field) = true];
}
```

##### GetCapacity エラー

プラグインが GetCapacity コールを無事完了できない場合、プラグインは gRPC ステータス中で non-ok gRPC コードを返却しなければならない(MUST)。

#### `ControllerGetCapabilities`

コントローラープラグインは本 RPC コールを実装しなければならない(MUST)。本 RPC は、CO がプラグインによって提供されるコントローラーサービスがサポートするケーパビリティをチェックできるようにする。

```protobuf
message ControllerGetCapabilitiesRequest {
  // Intentionally empty.
}

message ControllerGetCapabilitiesResponse {
  // All the capabilities that the controller service supports. This
  // field is OPTIONAL.
  repeated ControllerServiceCapability capabilities = 1;
}

// Specifies a capability of the controller service.
message ControllerServiceCapability {
  message RPC {
    enum Type {
      UNKNOWN = 0;
      CREATE_DELETE_VOLUME = 1;
      PUBLISH_UNPUBLISH_VOLUME = 2;
      LIST_VOLUMES = 3;
      GET_CAPACITY = 4;
      // Currently the only way to consume a snapshot is to create
      // a volume from it. Therefore plugins supporting
      // CREATE_DELETE_SNAPSHOT MUST support creating volume from
      // snapshot.
      CREATE_DELETE_SNAPSHOT = 5;
      LIST_SNAPSHOTS = 6;

      // Plugins supporting volume cloning at the storage level MAY
      // report this capability. The source volume MUST be managed by
      // the same plugin. Not all volume sources and parameters
      // combinations MAY work.
      CLONE_VOLUME = 7;

      // Indicates the SP supports ControllerPublishVolume.readonly
      // field.
      PUBLISH_READONLY = 8;

      // See VolumeExpansion for details.
      EXPAND_VOLUME = 9;

      // Indicates the SP supports the
      // ListVolumesResponse.entry.published_node_ids field and the
      // ControllerGetVolumeResponse.published_node_ids field.
      // The SP MUST also support PUBLISH_UNPUBLISH_VOLUME.
      LIST_VOLUMES_PUBLISHED_NODES = 10;

      // Indicates that the Controller service can report volume
      // conditions.
      // An SP MAY implement `VolumeCondition` in only the Controller
      // Plugin, only the Node Plugin, or both.
      // If `VolumeCondition` is implemented in both the Controller and
      // Node Plugins, it SHALL report from different perspectives.
      // If for some reason Controller and Node Plugins report
      // misaligned volume conditions, CO SHALL assume the worst case
      // is the truth.
      // Note that, for alpha, `VolumeCondition` is intended be
      // informative for humans only, not for automation.
      VOLUME_CONDITION = 11 [(alpha_enum_value) = true];

      // Indicates the SP supports the ControllerGetVolume RPC.
      // This enables COs to, for example, fetch per volume
      // condition after a volume is provisioned.
      GET_VOLUME = 12 [(alpha_enum_value) = true];

      // Indicates the SP supports the SINGLE_NODE_SINGLE_WRITER and/or
      // SINGLE_NODE_MULTI_WRITER access modes.
      // These access modes are intended to replace the
      // SINGLE_NODE_WRITER access mode to clarify the number of writers
      // for a volume on a single node. Plugins MUST accept and allow
      // use of the SINGLE_NODE_WRITER access mode when either
      // SINGLE_NODE_SINGLE_WRITER and/or SINGLE_NODE_MULTI_WRITER are
      // supported, in order to permit older COs to continue working.
      SINGLE_NODE_MULTI_WRITER = 13 [(alpha_enum_value) = true];
    }

    Type type = 1;
  }

  oneof type {
    // RPC that the controller supports.
    RPC rpc = 1;
  }
}
```

##### ControllerGetCapabilities Errors

プラグインが ControllerGetCapabilities コールを無事完了できない場合、プラグインは gRPC ステータス中で non-ok gRPC コードを返却しなければならない(MUST)。

#### `CreateSnapshot`

`CREATE_DELETE_SNAPSHOT` コントローラーケーパビリティを持つ場合、コントローラープラグインは本 RPC コールを実装しなければならない(MUST)。
ユーザの代わりに CO が元ボリュームからスナップショットを作成する為、本 RPC は CO によりコールされる。

本操作は冪等性を持たなければならない(MUST)。
指定したスナップショット `name` に関連付けられたスナップショットが無事作成され使用可能な場合(つまり、`CreateVolumeRequest` 中の `volume_content_source` で指定できる(MAY))、プラグインは関連する `CreateSnapshotResponse` で `0 OK` を返却しなければならない(MUST)。

スナップショットが作成される前にエラーが発生した場合、`CreateSnapshot` はエラー状況を反映した関連 gRPC エラーコードを返却すべきである(SHOULD)。

アップロードのようなスナップショットの後工程をサポートするプラグインの為、`CreateSnapshot` は `0 OK` を返却し(SHOULD)、スナップショット作成後に依然処理中だった場合は `ready_to_use` を `false` にセットすべきである(SHOULD)。
スナップショット作成が「処理済み」であり、新しいボリューム作成に使用可能である事を示す `ready_to_use` が `true` に切り替わるまで、CO は `CreateSnapshotRequest` を定期的に再実行すべきである(SHOULD)
処理中にエラーが発生した場合、`CreateSnapshot` はエラー状況を反映した関連 gRPC エラーコードを返却すべきである(SHOULD)。

スナップショットは新しいボリュームを生成する為のソースとして使用できる(MAY)。
CreateVolumeRequest メッセージは省略可能(OPTIONAL)な元スナップショットパラメータを指定しても良い(MAY)。
スナップショットを戻す(元ボリューム中のデータを削除し、スナップショット中のデータで置き換える)事は先進的な機能であり、全てのストレージシステムがサポートするものではなく、それゆえ本機能は現在スコープ外です。

##### ready_to_use パラメーター

スナップショット作成後にスナップショットをどこかにアップロードするなど、スナップショット作成後にいくつかの SP はスナップショットを「処理」できる(MAY)。
作成後処理は数時間かかるような長時間処理になりうる(MAY)。
CO はスナップショットを取得する前に元ボリュームを使用するアプリケーションをフリーズさせる可能性がある(MAY)。
`freeze` の目的はアプリケーションデータが一貫性ある状態である事を保証する事です。
`freeze` 実行の際、コンテナは一旦停止し、アプリケーションも一旦停止します。
`thaw` 実行の際、コンテナとアプリケーションは再び実行し始めます。
スナップショット作成フェーズの間、スナップショットが既に作成されていたら、`thaw` 操作は実行可能で、そのため処理が完了するのを待つ事なくアプリケーションは実行開始できます。
処理完了後、スナップショットの `ready_to_use` パラメータは `true` になります。

スナップショット作成後に追加処理を行わない SP の為、スナップショット作成後 `ready_to_use` は `true` になるべきである(SHOULD)。
この場合、`ready_to_use` パラメータが `true` になった時、`thaw` は実行できる。

スナップショット中の処理で CO がアプリケーションを「温める(thaw)」事が出来る際に、`ready_to_use` パラメータは CO にガイダンスを与える。
スナップショット作成後にクラウドプロバイダあるいはストレージシステムがスナップショットを処理する必要がある場合、CreateSnapshot により返却される `ready_to_use` パラメータは `false` であるべきである(SHALL)。
`ready_to_use` が `true` になるまで処理完了を待つ間、CO は　CreateSnapshot をコールし続けても良い(MAY)。
スナップショット作成御、CreateSnapshot は最早ブロックしないことに注意。

スナップショット処理のいずれの段階中でエラーが発生した場合、gRPC エラーコードが返却されるべきである(SHALL)。
エラー発生時、CO は明確にスナップショットを削除すべきである(SHOULD)。

この情報に基づき、CO は CreateSnapshot のコールを繰り返し(冪等性)発行し、応答を監視し、判断できる(can)。
CreateSnapshot は同期コールであり、スナップショットが作成されるまでブロックされなければならない(MUST)事に注意。

```protobuf
message CreateSnapshotRequest {
  // The ID of the source volume to be snapshotted.
  // This field is REQUIRED.
  string source_volume_id = 1;

  // The suggested name for the snapshot. This field is REQUIRED for
  // idempotency.
  // Any Unicode string that conforms to the length limit is allowed
  // except those containing the following banned characters:
  // U+0000-U+0008, U+000B, U+000C, U+000E-U+001F, U+007F-U+009F.
  // (These are control characters other than commonly used whitespace.)
  string name = 2;

  // Secrets required by plugin to complete snapshot creation request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 3 [(csi_secret) = true];

  // Plugin specific parameters passed in as opaque key-value pairs.
  // This field is OPTIONAL. The Plugin is responsible for parsing and
  // validating these parameters. COs will treat these as opaque.
  // Use cases for opaque parameters:
  // - Specify a policy to automatically clean up the snapshot.
  // - Specify an expiration date for the snapshot.
  // - Specify whether the snapshot is readonly or read/write.
  // - Specify if the snapshot should be replicated to some place.
  // - Specify primary or secondary for replication systems that
  //   support snapshotting only on primary.
  map<string, string> parameters = 4;
}

message CreateSnapshotResponse {
  // Contains all attributes of the newly created snapshot that are
  // relevant to the CO along with information required by the Plugin
  // to uniquely identify the snapshot. This field is REQUIRED.
  Snapshot snapshot = 1;
}

// Information about a specific snapshot.
message Snapshot {
  // This is the complete size of the snapshot in bytes. The purpose of
  // this field is to give CO guidance on how much space is needed to
  // create a volume from this snapshot. The size of the volume MUST NOT
  // be less than the size of the source snapshot. This field is
  // OPTIONAL. If this field is not set, it indicates that this size is
  // unknown. The value of this field MUST NOT be negative and a size of
  // zero means it is unspecified.
  int64 size_bytes = 1;

  // The identifier for this snapshot, generated by the plugin.
  // This field is REQUIRED.
  // This field MUST contain enough information to uniquely identify
  // this specific snapshot vs all other snapshots supported by this
  // plugin.
  // This field SHALL be used by the CO in subsequent calls to refer to
  // this snapshot.
  // The SP is NOT responsible for global uniqueness of snapshot_id
  // across multiple SPs.
  string snapshot_id = 2;

  // Identity information for the source volume. Note that creating a
  // snapshot from a snapshot is not supported here so the source has to
  // be a volume. This field is REQUIRED.
  string source_volume_id = 3;

  // Timestamp when the point-in-time snapshot is taken on the storage
  // system. This field is REQUIRED.
  .google.protobuf.Timestamp creation_time = 4;

  // Indicates if a snapshot is ready to use as a
  // `volume_content_source` in a `CreateVolumeRequest`. The default
  // value is false. This field is REQUIRED.
  bool ready_to_use = 5;
}
```

##### CreateSnapshot エラー

プラグインが CreateSnapshot コールを無事完了できない場合、プラグインは gRPC ステータス中で non-ok gRPC コードを返却しなければならない(MUST)。
下記で定義された状況に遭遇した場合、プラグインは指定された gRPC エラーコードを返却しなければならない(MUST)。
CO は、gRPC エラーコードに遭遇した際の指定されたエラー復旧動作を実装しなければならない(MUST)。

| 状況 | gRPC コード | 説明 | 復旧動作 |
|-----------|-----------|-------------|-------------------|
| スナップショットは既に存在するが非互換である | 6 ALREADY_EXISTS | 指定されたスナップショット `name` に関連付けられたスナップショット既に存在するが、指定された `volume_id` と一致しない。 | 呼び出し元は再試行前に引数を修正するか異なる `name` を使用しなければならない(MUST)。 |
| スナップショットの操作が保留中 | 10 ABORTED | 指定されたスナップショットの保留中の操作が既にある事を示す。
一般に、CO はある瞬間にスナップショットに対して「実行中の」コールが複数ない事を保証する責任を持つ。

しかし、(例えば、CO がクラッシュして再起動した場合など) 状況によっては CO が状態を失う可能背があり(MAY)、同じスナップショットに対して同時に複数の呼び出しを発行する可能性がある(MAY)。
プラグイン は可能な限り穏便にこの問題を扱うべきであり(SHOULD)、2番目の呼び出しを拒否する為にこのエラーコードを返却する事ができる(MAY)。 | 
呼び出し元は特定のスナップショットに対して他の保留中の呼び出しがない事を保証し、その後指数バックオフで再試行すべきである(SHOULD)。 |
| スナップショット作成のための十分なスペースがない | 13 RESOURCE_EXHAUSTED | スナップショット作成要求を処理する為の、ストレージシステム上の十分なスペースがない。 | 呼び出し元はこの要求を失敗とすべきである(SHOULD)。今後スペースが開放された場合、CreateSnapshot 呼び出しは成功する可能性がある(MAY)。 |


#### `DeleteSnapshot`

A Controller Plugin MUST implement this RPC call if it has `CREATE_DELETE_SNAPSHOT` capability.
This RPC will be called by the CO to delete a snapshot.

This operation MUST be idempotent.
If a snapshot corresponding to the specified `snapshot_id` does not exist or the artifacts associated with the snapshot do not exist anymore, the Plugin MUST reply `0 OK`.

```protobuf
message DeleteSnapshotRequest {
  // The ID of the snapshot to be deleted.
  // This field is REQUIRED.
  string snapshot_id = 1;

  // Secrets required by plugin to complete snapshot deletion request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 2 [(csi_secret) = true];
}

message DeleteSnapshotResponse {}
```

##### DeleteSnapshot Errors

If the plugin is unable to complete the DeleteSnapshot call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Snapshot in use | 9 FAILED_PRECONDITION | Indicates that the snapshot corresponding to the specified `snapshot_id` could not be deleted because it is in use by another resource. | Caller SHOULD ensure that there are no other resources using the snapshot, and then retry with exponential back off. |
| Operation pending for snapshot | 10 ABORTED | Indicates that there is already an operation pending for the specified snapshot. In general the Cluster Orchestrator (CO) is responsible for ensuring that there is no more than one call "in-flight" per snapshot at a given time. However, in some circumstances, the CO MAY lose state (for example when the CO crashes and restarts), and MAY issue multiple calls simultaneously for the same snapshot. The Plugin, SHOULD handle this as gracefully as possible, and MAY return this error code to reject secondary calls. | Caller SHOULD ensure that there are no other calls pending for the specified snapshot, and then retry with exponential back off. |


#### `ListSnapshots`

A Controller Plugin MUST implement this RPC call if it has `LIST_SNAPSHOTS` capability.
The Plugin SHALL return the information about all snapshots on the storage system within the given parameters regardless of how they were created.
`ListSnapshots` SHALL NOT list a snapshot that is being created but has not been cut successfully yet.
If snapshots are created and/or deleted while the CO is concurrently paging through `ListSnapshots` results then it is possible that the CO MAY either witness duplicate snapshots in the list, not witness existing snapshots, or both.
The CO SHALL NOT expect a consistent "view" of all snapshots when paging through the snapshot list via multiple calls to `ListSnapshots`.

```protobuf
// List all snapshots on the storage system regardless of how they were
// created.
message ListSnapshotsRequest {
  // If specified (non-zero value), the Plugin MUST NOT return more
  // entries than this number in the response. If the actual number of
  // entries is more than this number, the Plugin MUST set `next_token`
  // in the response which can be used to get the next page of entries
  // in the subsequent `ListSnapshots` call. This field is OPTIONAL. If
  // not specified (zero value), it means there is no restriction on the
  // number of entries that can be returned.
  // The value of this field MUST NOT be negative.
  int32 max_entries = 1;

  // A token to specify where to start paginating. Set this field to
  // `next_token` returned by a previous `ListSnapshots` call to get the
  // next page of entries. This field is OPTIONAL.
  // An empty string is equal to an unspecified field value.
  string starting_token = 2;

  // Identity information for the source volume. This field is OPTIONAL.
  // It can be used to list snapshots by volume.
  string source_volume_id = 3;

  // Identity information for a specific snapshot. This field is
  // OPTIONAL. It can be used to list only a specific snapshot.
  // ListSnapshots will return with current snapshot information
  // and will not block if the snapshot is being processed after
  // it is cut.
  string snapshot_id = 4;

  // Secrets required by plugin to complete ListSnapshot request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 5 [(csi_secret) = true];
}

message ListSnapshotsResponse {
  message Entry {
    Snapshot snapshot = 1;
  }

  repeated Entry entries = 1;

  // This token allows you to get the next page of entries for
  // `ListSnapshots` request. If the number of entries is larger than
  // `max_entries`, use the `next_token` as a value for the
  // `starting_token` field in the next `ListSnapshots` request. This
  // field is OPTIONAL.
  // An empty string is equal to an unspecified field value.
  string next_token = 2;
}
```

##### ListSnapshots Errors

If the plugin is unable to complete the ListSnapshots call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Invalid `starting_token` | 10 ABORTED | Indicates that `starting_token` is not valid. | Caller SHOULD start the `ListSnapshots` operation again with an empty `starting_token`. |


#### `ControllerExpandVolume`

A Controller plugin MUST implement this RPC call if plugin has `EXPAND_VOLUME` controller capability.
This RPC allows the CO to expand the size of a volume.

This operation MUST be idempotent.
If a volume corresponding to the specified volume ID is already larger than or equal to the target capacity of the expansion request, the plugin SHOULD reply 0 OK.

This call MAY be made by the CO during any time in the lifecycle of the volume after creation if plugin has `VolumeExpansion.ONLINE` capability.
If plugin has `EXPAND_VOLUME` node capability, then `NodeExpandVolume` MUST be called after successful `ControllerExpandVolume` and `node_expansion_required` in `ControllerExpandVolumeResponse` is `true`.

If specified, the `volume_capability` in `ControllerExpandVolumeRequest` should be same as what CO would pass in `ControllerPublishVolumeRequest`.

If the plugin has only `VolumeExpansion.OFFLINE` expansion capability and volume is currently published or available on a node then `ControllerExpandVolume` MUST be called ONLY after either:
- The plugin has controller `PUBLISH_UNPUBLISH_VOLUME` capability and `ControllerUnpublishVolume` has been invoked successfully.

OR ELSE

- The plugin does NOT have controller `PUBLISH_UNPUBLISH_VOLUME` capability, the plugin has node `STAGE_UNSTAGE_VOLUME` capability, and `NodeUnstageVolume` has been completed successfully.

OR ELSE

- The plugin does NOT have controller `PUBLISH_UNPUBLISH_VOLUME` capability, nor node `STAGE_UNSTAGE_VOLUME` capability, and `NodeUnpublishVolume` has completed successfully.

Examples:
* Offline Volume Expansion:
  Given an ElasticSearch process that runs on Azure Disk and needs more space.
  - The administrator takes the Elasticsearch server offline by stopping the workload and CO calls `ControllerUnpublishVolume`.
  - The administrator requests more space for the volume from CO.
  - The CO in turn first makes `ControllerExpandVolume` RPC call which results in requesting more space from Azure cloud provider for volume ID that was being used by ElasticSearch.
  - Once `ControllerExpandVolume` is completed and successful, the CO will inform administrator about it and administrator will resume the ElasticSearch workload.
  - On the node where the ElasticSearch workload is scheduled, the CO calls `NodeExpandVolume` after calling `NodeStageVolume`.
  - Calling `NodeExpandVolume` on volume results in expanding the underlying file system and added space becomes available to workload when it starts up.
* Online Volume Expansion:
  Given a Mysql server running on Openstack Cinder and needs more space.
  - The administrator requests more space for volume from the CO.
  - The CO in turn first makes `ControllerExpandVolume` RPC call which results in requesting more space from Openstack Cinder for given volume.
  - On the node where the mysql workload is running, the CO calls `NodeExpandVolume` while volume is in-use using the path where the volume is staged.
  - Calling `NodeExpandVolume` on volume results in expanding the underlying file system and added space automatically becomes available to mysql workload without any downtime.


```protobuf
message ControllerExpandVolumeRequest {
  // The ID of the volume to expand. This field is REQUIRED.
  string volume_id = 1;

  // This allows CO to specify the capacity requirements of the volume
  // after expansion. This field is REQUIRED.
  CapacityRange capacity_range = 2;

  // Secrets required by the plugin for expanding the volume.
  // This field is OPTIONAL.
  map<string, string> secrets = 3 [(csi_secret) = true];

  // Volume capability describing how the CO intends to use this volume.
  // This allows SP to determine if volume is being used as a block
  // device or mounted file system. For example - if volume is
  // being used as a block device - the SP MAY set
  // node_expansion_required to false in ControllerExpandVolumeResponse
  // to skip invocation of NodeExpandVolume on the node by the CO.
  // This is an OPTIONAL field.
  VolumeCapability volume_capability = 4;
}

message ControllerExpandVolumeResponse {
  // Capacity of volume after expansion. This field is REQUIRED.
  int64 capacity_bytes = 1;

  // Whether node expansion is required for the volume. When true
  // the CO MUST make NodeExpandVolume RPC call on the node. This field
  // is REQUIRED.
  bool node_expansion_required = 2;
}
```

##### ControllerExpandVolume Errors

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Exceeds capabilities | 3 INVALID_ARGUMENT | Indicates that the CO has specified capabilities not supported by the volume. | Caller MAY verify volume capabilities by calling ValidateVolumeCapabilities and retry with matching capabilities. |
| Volume does not exist | 5 NOT FOUND | Indicates that a volume corresponding to the specified volume_id does not exist. | Caller MUST verify that the volume_id is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |
| Volume in use | 9 FAILED_PRECONDITION | Indicates that the volume corresponding to the specified `volume_id` could not be expanded because it is currently published on a node but the plugin does not have ONLINE expansion capability. | Caller SHOULD ensure that volume is not published and retry with exponential back off. |
| Unsupported `capacity_range` | 11 OUT_OF_RANGE | Indicates that the capacity range is not allowed by the Plugin. More human-readable information MAY be provided in the gRPC `status.message` field. | Caller MUST fix the capacity range before retrying. |

#### RPC Interactions

##### `CreateVolume`, `DeleteVolume`, `ListVolumes`

It is worth noting that the plugin-generated `volume_id` is a REQUIRED field for the `DeleteVolume` RPC, as opposed to the CO-generated volume `name` that is REQUIRED for the `CreateVolume` RPC: these fields MAY NOT contain the same value.
If a `CreateVolume` operation times out, leaving the CO without an ID with which to reference a volume, and the CO *also* decides that it no longer needs/wants the volume in question then the CO MAY choose one of the following paths:

1. Replay the `CreateVolume` RPC that timed out; upon success execute `DeleteVolume` using the known volume ID (from the response to `CreateVolume`).
2. Execute the `ListVolumes` RPC to possibly obtain a volume ID that may be used to execute a `DeleteVolume` RPC; upon success execute `DeleteVolume`.
3. The CO takes no further action regarding the timed out RPC, a volume is possibly leaked and the operator/user is expected to clean up.

It is NOT REQUIRED for a controller plugin to implement the `LIST_VOLUMES` capability if it supports the `CREATE_DELETE_VOLUME` capability: the onus is upon the CO to take into consideration the full range of plugin capabilities before deciding how to proceed in the above scenario.

##### `CreateSnapshot`, `DeleteSnapshot`, `ListSnapshots`

The plugin-generated `snapshot_id` is a REQUIRED field for the `DeleteSnapshot` RPC, as opposed to the CO-generated snapshot `name` that is REQUIRED for the `CreateSnapshot` RPC.
A `CreateSnapshot` operation SHOULD return with a `snapshot_id` when the snapshot is cut successfully.
If a `CreateSnapshot` operation times out before the snapshot is cut, leaving the CO without an ID with which to reference a snapshot, and the CO also decides that it no longer needs/wants the snapshot in question then the CO MAY choose one of the following paths:

1. Execute the `ListSnapshots` RPC to possibly obtain a snapshot ID that may be used to execute a `DeleteSnapshot` RPC; upon success execute `DeleteSnapshot`.
2. The CO takes no further action regarding the timed out RPC, a snapshot is possibly leaked and the operator/user is expected to clean up.

It is NOT REQUIRED for a controller plugin to implement the `LIST_SNAPSHOTS` capability if it supports the `CREATE_DELETE_SNAPSHOT` capability: the onus is upon the CO to take into consideration the full range of plugin capabilities before deciding how to proceed in the above scenario.

ListSnapshots SHALL return with current information regarding the snapshots on the storage system.
When processing is complete, the `ready_to_use` parameter of the snapshot from ListSnapshots SHALL become `true`.
The downside of calling ListSnapshots is that ListSnapshots will not return a gRPC error code if an error occurs during the processing. So calling CreateSnapshot repeatedly is the preferred way to check if the processing is complete.

### Node Service RPC

#### `NodeStageVolume`

A Node Plugin MUST implement this RPC call if it has `STAGE_UNSTAGE_VOLUME` node capability.

This RPC is called by the CO prior to the volume being consumed by any workloads on the node by `NodePublishVolume`.
The Plugin SHALL assume that this RPC will be executed on the node where the volume will be used.
This RPC SHOULD be called by the CO when a workload that wants to use the specified volume is placed (scheduled) on the specified node for the first time or for the first time since a `NodeUnstageVolume` call for the specified volume was called and returned success on that node.

If the corresponding Controller Plugin has `PUBLISH_UNPUBLISH_VOLUME` controller capability and the Node Plugin has `STAGE_UNSTAGE_VOLUME` capability, then the CO MUST guarantee that this RPC is called after `ControllerPublishVolume` is called for the given volume on the given node and returns a success.
The CO MUST guarantee that this RPC is called and returns a success before any `NodePublishVolume` is called for the given volume on the given node.

This operation MUST be idempotent.
If the volume corresponding to the `volume_id` is already staged to the `staging_target_path`, and is identical to the specified `volume_capability` the Plugin MUST reply `0 OK`.

If this RPC failed, or the CO does not know if it failed or not, it MAY choose to call `NodeStageVolume` again, or choose to call `NodeUnstageVolume`.

```protobuf
message NodeStageVolumeRequest {
  // The ID of the volume to publish. This field is REQUIRED.
  string volume_id = 1;

  // The CO SHALL set this field to the value returned by
  // `ControllerPublishVolume` if the corresponding Controller Plugin
  // has `PUBLISH_UNPUBLISH_VOLUME` controller capability, and SHALL be
  // left unset if the corresponding Controller Plugin does not have
  // this capability. This is an OPTIONAL field.
  map<string, string> publish_context = 2;

  // The path to which the volume MAY be staged. It MUST be an
  // absolute path in the root filesystem of the process serving this
  // request, and MUST be a directory. The CO SHALL ensure that there
  // is only one `staging_target_path` per volume. The CO SHALL ensure
  // that the path is directory and that the process serving the
  // request has `read` and `write` permission to that directory. The
  // CO SHALL be responsible for creating the directory if it does not
  // exist.
  // This is a REQUIRED field.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string staging_target_path = 3;

  // Volume capability describing how the CO intends to use this volume.
  // SP MUST ensure the CO can use the staged volume as described.
  // Otherwise SP MUST return the appropriate gRPC error code.
  // This is a REQUIRED field.
  VolumeCapability volume_capability = 4;

  // Secrets required by plugin to complete node stage volume request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 5 [(csi_secret) = true];

  // Volume context as returned by SP in
  // CreateVolumeResponse.Volume.volume_context.
  // This field is OPTIONAL and MUST match the volume_context of the
  // volume identified by `volume_id`.
  map<string, string> volume_context = 6;
}

message NodeStageVolumeResponse {
  // Intentionally empty.
}
```

#### NodeStageVolume Errors

If the plugin is unable to complete the NodeStageVolume call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist. | Caller MUST verify that the `volume_id` is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |
| Volume published but is incompatible | 6 ALREADY_EXISTS | Indicates that a volume corresponding to the specified `volume_id` has already been published at the specified `staging_target_path` but is incompatible with the specified `volume_capability` flag. | Caller MUST fix the arguments before retrying. |
| Exceeds capabilities | 9 FAILED_PRECONDITION | Indicates that the CO has specified capabilities not supported by the volume. | Caller MAY choose to call `ValidateVolumeCapabilities` to validate the volume capabilities, or wait for the volume to be unpublished on the node. |

#### `NodeUnstageVolume`

A Node Plugin MUST implement this RPC call if it has `STAGE_UNSTAGE_VOLUME` node capability.

This RPC is a reverse operation of `NodeStageVolume`.
This RPC MUST undo the work by the corresponding `NodeStageVolume`.
This RPC SHALL be called by the CO once for each `staging_target_path` that was successfully setup via `NodeStageVolume`.

If the corresponding Controller Plugin has `PUBLISH_UNPUBLISH_VOLUME` controller capability and the Node Plugin has `STAGE_UNSTAGE_VOLUME` capability, the CO MUST guarantee that this RPC is called and returns success before calling `ControllerUnpublishVolume` for the given node and the given volume.
The CO MUST guarantee that this RPC is called after all `NodeUnpublishVolume` have been called and returned success for the given volume on the given node.

The Plugin SHALL assume that this RPC will be executed on the node where the volume is being used.

This RPC MAY be called by the CO when the workload using the volume is being moved to a different node, or all the workloads using the volume on a node have finished.

This operation MUST be idempotent.
If the volume corresponding to the `volume_id` is not staged to the `staging_target_path`,  the Plugin MUST reply `0 OK`.

If this RPC failed, or the CO does not know if it failed or not, it MAY choose to call `NodeUnstageVolume` again.

```protobuf
message NodeUnstageVolumeRequest {
  // The ID of the volume. This field is REQUIRED.
  string volume_id = 1;

  // The path at which the volume was staged. It MUST be an absolute
  // path in the root filesystem of the process serving this request.
  // This is a REQUIRED field.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string staging_target_path = 2;
}

message NodeUnstageVolumeResponse {
  // Intentionally empty.
}
```

#### NodeUnstageVolume Errors

If the plugin is unable to complete the NodeUnstageVolume call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist. | Caller MUST verify that the `volume_id` is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |

#### RPC Interactions and Reference Counting
`NodeStageVolume`, `NodeUnstageVolume`, `NodePublishVolume`, `NodeUnpublishVolume`

The following interaction semantics ARE REQUIRED if the plugin advertises the `STAGE_UNSTAGE_VOLUME` capability.
`NodeStageVolume` MUST be called and return success once per volume per node before any `NodePublishVolume` MAY be called for the volume.
All `NodeUnpublishVolume` MUST be called and return success for a volume before `NodeUnstageVolume` MAY be called for the volume.

Note that this requires that all COs MUST support reference counting of volumes so that if `STAGE_UNSTAGE_VOLUME` is advertised by the SP, the CO MUST fulfill the above interaction semantics.

#### `NodePublishVolume`

This RPC is called by the CO when a workload that wants to use the specified volume is placed (scheduled) on a node.
The Plugin SHALL assume that this RPC will be executed on the node where the volume will be used.

If the corresponding Controller Plugin has `PUBLISH_UNPUBLISH_VOLUME` controller capability, the CO MUST guarantee that this RPC is called after `ControllerPublishVolume` is called for the given volume on the given node and returns a success.

This operation MUST be idempotent.
If the volume corresponding to the `volume_id` has already been published at the specified `target_path`, and is compatible with the specified `volume_capability` and `readonly` flag, the Plugin MUST reply `0 OK`.

If this RPC failed, or the CO does not know if it failed or not, it MAY choose to call `NodePublishVolume` again, or choose to call `NodeUnpublishVolume`.

This RPC MAY be called by the CO multiple times on the same node for the same volume with a possibly different `target_path` and/or other arguments if the volume supports either `MULTI_NODE_...` or `SINGLE_NODE_MULTI_WRITER` access modes (see second table).
The possible `MULTI_NODE_...` access modes are `MULTI_NODE_READER_ONLY`, `MULTI_NODE_SINGLE_WRITER` or `MULTI_NODE_MULTI_WRITER`.
COs SHOULD NOT call `NodePublishVolume` a second time with a different `volume_capability`.
If this happens, the Plugin SHOULD return `FAILED_PRECONDITION`.

The following table shows what the Plugin SHOULD return when receiving a second `NodePublishVolume` on the same volume on the same node:

(**T<sub>n</sub>**: target path of the n<sup>th</sup> `NodePublishVolume`; **P<sub>n</sub>**: other arguments of the n<sup>th</sup> `NodePublishVolume` except `secrets`)

|                    | T1=T2, P1=P2    | T1=T2, P1!=P2  | T1!=T2, P1=P2       | T1!=T2, P1!=P2     |
|--------------------|-----------------|----------------|---------------------|--------------------|
| MULTI_NODE_...     | OK (idempotent) | ALREADY_EXISTS | OK                  | OK                 |
| Non MULTI_NODE_... | OK (idempotent) | ALREADY_EXISTS | FAILED_PRECONDITION | FAILED_PRECONDITION|

NOTE: If the Plugin supports the `SINGLE_NODE_MULTI_WRITER` capability, use the following table instead for what the Plugin SHOULD return when receiving a second `NodePublishVolume` on the same volume on the same node:

|                                       | T1=T2, P1=P2    | T1=T2, P1!=P2  | T1!=T2, P1=P2       | T1!=T2, P1!=P2      |
|---------------------------------------|-----------------|----------------|---------------------|---------------------|
| SINGLE_NODE_SINGLE_WRITER             | OK (idempotent) | ALREADY_EXISTS | FAILED_PRECONDITION | FAILED_PRECONDITION |
| SINGLE_NODE_MULTI_WRITER              | OK (idempotent) | ALREADY_EXISTS | OK                  | OK                  |
| MULTI_NODE_...                        | OK (idempotent) | ALREADY_EXISTS | OK                  | OK                  |
| Non MULTI_NODE_...                    | OK (idempotent) | ALREADY_EXISTS | FAILED_PRECONDITION | FAILED_PRECONDITION |

The `SINGLE_NODE_SINGLE_WRITER` and `SINGLE_NODE_MULTI_WRITER` access modes are intended to replace the `SINGLE_NODE_WRITER` access mode to clarify the number of writers for a volume on a single node.
Plugins MUST accept and allow use of the `SINGLE_NODE_WRITER` access mode (subject to the processing rules above), when either `SINGLE_NODE_SINGLE_WRITER` and/or `SINGLE_NODE_MULTI_WRITER` are supported, in order to permit older COs to continue working.


```protobuf
message NodePublishVolumeRequest {
  // The ID of the volume to publish. This field is REQUIRED.
  string volume_id = 1;

  // The CO SHALL set this field to the value returned by
  // `ControllerPublishVolume` if the corresponding Controller Plugin
  // has `PUBLISH_UNPUBLISH_VOLUME` controller capability, and SHALL be
  // left unset if the corresponding Controller Plugin does not have
  // this capability. This is an OPTIONAL field.
  map<string, string> publish_context = 2;

  // The path to which the volume was staged by `NodeStageVolume`.
  // It MUST be an absolute path in the root filesystem of the process
  // serving this request.
  // It MUST be set if the Node Plugin implements the
  // `STAGE_UNSTAGE_VOLUME` node capability.
  // This is an OPTIONAL field.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string staging_target_path = 3;

  // The path to which the volume will be published. It MUST be an
  // absolute path in the root filesystem of the process serving this
  // request. The CO SHALL ensure uniqueness of target_path per volume.
  // The CO SHALL ensure that the parent directory of this path exists
  // and that the process serving the request has `read` and `write`
  // permissions to that parent directory.
  // For volumes with an access type of block, the SP SHALL place the
  // block device at target_path.
  // For volumes with an access type of mount, the SP SHALL place the
  // mounted directory at target_path.
  // Creation of target_path is the responsibility of the SP.
  // This is a REQUIRED field.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string target_path = 4;

  // Volume capability describing how the CO intends to use this volume.
  // SP MUST ensure the CO can use the published volume as described.
  // Otherwise SP MUST return the appropriate gRPC error code.
  // This is a REQUIRED field.
  VolumeCapability volume_capability = 5;

  // Indicates SP MUST publish the volume in readonly mode.
  // This field is REQUIRED.
  bool readonly = 6;

  // Secrets required by plugin to complete node publish volume request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 7 [(csi_secret) = true];

  // Volume context as returned by SP in
  // CreateVolumeResponse.Volume.volume_context.
  // This field is OPTIONAL and MUST match the volume_context of the
  // volume identified by `volume_id`.
  map<string, string> volume_context = 8;
}

message NodePublishVolumeResponse {
  // Intentionally empty.
}
```

##### NodePublishVolume Errors

If the plugin is unable to complete the NodePublishVolume call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist. | Caller MUST verify that the `volume_id` is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |
| Volume published but is incompatible | 6 ALREADY_EXISTS | Indicates that a volume corresponding to the specified `volume_id` has already been published at the specified `target_path` but is incompatible with the specified `volume_capability` or `readonly` flag. | Caller MUST fix the arguments before retrying. |
| Exceeds capabilities | 9 FAILED_PRECONDITION | Indicates that the CO has specified capabilities not supported by the volume. | Caller MAY choose to call `ValidateVolumeCapabilities` to validate the volume capabilities, or wait for the volume to be unpublished on the node. |
| Staging target path not set | 9 FAILED_PRECONDITION | Indicates that `STAGE_UNSTAGE_VOLUME` capability is set but no `staging_target_path` was set. | Caller MUST make sure call to `NodeStageVolume` is made and returns success before retrying with valid `staging_target_path`. |



#### `NodeUnpublishVolume`

A Node Plugin MUST implement this RPC call.
This RPC is a reverse operation of `NodePublishVolume`.
This RPC MUST undo the work by the corresponding `NodePublishVolume`.
This RPC SHALL be called by the CO at least once for each `target_path` that was successfully setup via `NodePublishVolume`.
If the corresponding Controller Plugin has `PUBLISH_UNPUBLISH_VOLUME` controller capability, the CO SHOULD issue all `NodeUnpublishVolume` (as specified above) before calling `ControllerUnpublishVolume` for the given node and the given volume.
The Plugin SHALL assume that this RPC will be executed on the node where the volume is being used.

This RPC is typically called by the CO when the workload using the volume is being moved to a different node, or all the workload using the volume on a node has finished.

This operation MUST be idempotent.
If this RPC failed, or the CO does not know if it failed or not, it can choose to call `NodeUnpublishVolume` again.

```protobuf
message NodeUnpublishVolumeRequest {
  // The ID of the volume. This field is REQUIRED.
  string volume_id = 1;

  // The path at which the volume was published. It MUST be an absolute
  // path in the root filesystem of the process serving this request.
  // The SP MUST delete the file or directory it created at this path.
  // This is a REQUIRED field.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string target_path = 2;
}

message NodeUnpublishVolumeResponse {
  // Intentionally empty.
}
```

##### NodeUnpublishVolume Errors

If the plugin is unable to complete the NodeUnpublishVolume call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist. | Caller MUST verify that the `volume_id` is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |


#### `NodeGetVolumeStats`

A Node plugin MUST implement this RPC call if it has GET_VOLUME_STATS node capability or VOLUME_CONDITION node capability.
`NodeGetVolumeStats` RPC call returns the volume capacity statistics available for the volume.

If the volume is being used in `BlockVolume` mode then `used` and `available` MAY be omitted from `usage` field of `NodeGetVolumeStatsResponse`.
Similarly, inode information MAY be omitted from `NodeGetVolumeStatsResponse` when unavailable.

The `staging_target_path` field is not required, for backwards compatibility, but the CO SHOULD supply it.
Plugins can use this field to determine if `volume_path` is where the volume is published or staged,
and setting this field to non-empty allows plugins to function with less stored state on the node.

```protobuf
message NodeGetVolumeStatsRequest {
  // The ID of the volume. This field is REQUIRED.
  string volume_id = 1;

  // It can be any valid path where volume was previously
  // staged or published.
  // It MUST be an absolute path in the root filesystem of
  // the process serving this request.
  // This is a REQUIRED field.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string volume_path = 2;

  // The path where the volume is staged, if the plugin has the
  // STAGE_UNSTAGE_VOLUME capability, otherwise empty.
  // If not empty, it MUST be an absolute path in the root
  // filesystem of the process serving this request.
  // This field is OPTIONAL.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string staging_target_path = 3;
}

message NodeGetVolumeStatsResponse {
  // This field is OPTIONAL.
  repeated VolumeUsage usage = 1;
  // Information about the current condition of the volume.
  // This field is OPTIONAL.
  // This field MUST be specified if the VOLUME_CONDITION node
  // capability is supported.
  VolumeCondition volume_condition = 2 [(alpha_field) = true];
}

message VolumeUsage {
  enum Unit {
    UNKNOWN = 0;
    BYTES = 1;
    INODES = 2;
  }
  // The available capacity in specified Unit. This field is OPTIONAL.
  // The value of this field MUST NOT be negative.
  int64 available = 1;

  // The total capacity in specified Unit. This field is REQUIRED.
  // The value of this field MUST NOT be negative.
  int64 total = 2;

  // The used capacity in specified Unit. This field is OPTIONAL.
  // The value of this field MUST NOT be negative.
  int64 used = 3;

  // Units by which values are measured. This field is REQUIRED.
  Unit unit = 4;
}

// VolumeCondition represents the current condition of a volume.
message VolumeCondition {
  option (alpha_message) = true;

  // Normal volumes are available for use and operating optimally.
  // An abnormal volume does not meet these criteria.
  // This field is REQUIRED.
  bool abnormal = 1;

  // The message describing the condition of the volume.
  // This field is REQUIRED.
  string message = 2;
}
```

##### NodeGetVolumeStats Errors

If the plugin is unable to complete the `NodeGetVolumeStats` call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.


| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist on specified `volume_path`. | Caller MUST verify that the `volume_id` is correct and that the volume is accessible on specified `volume_path` and has not been deleted before retrying with exponential back off. |

#### `NodeGetCapabilities`

A Node Plugin MUST implement this RPC call.
This RPC allows the CO to check the supported capabilities of node service provided by the Plugin.

```protobuf
message NodeGetCapabilitiesRequest {
  // Intentionally empty.
}

message NodeGetCapabilitiesResponse {
  // All the capabilities that the node service supports. This field
  // is OPTIONAL.
  repeated NodeServiceCapability capabilities = 1;
}

// Specifies a capability of the node service.
message NodeServiceCapability {
  message RPC {
    enum Type {
      UNKNOWN = 0;
      STAGE_UNSTAGE_VOLUME = 1;
      // If Plugin implements GET_VOLUME_STATS capability
      // then it MUST implement NodeGetVolumeStats RPC
      // call for fetching volume statistics.
      GET_VOLUME_STATS = 2;
      // See VolumeExpansion for details.
      EXPAND_VOLUME = 3;
      // Indicates that the Node service can report volume conditions.
      // An SP MAY implement `VolumeCondition` in only the Node
      // Plugin, only the Controller Plugin, or both.
      // If `VolumeCondition` is implemented in both the Node and
      // Controller Plugins, it SHALL report from different
      // perspectives.
      // If for some reason Node and Controller Plugins report
      // misaligned volume conditions, CO SHALL assume the worst case
      // is the truth.
      // Note that, for alpha, `VolumeCondition` is intended to be
      // informative for humans only, not for automation.
      VOLUME_CONDITION = 4 [(alpha_enum_value) = true];

      // Indicates the SP supports the SINGLE_NODE_SINGLE_WRITER and/or
      // SINGLE_NODE_MULTI_WRITER access modes.
      // These access modes are intended to replace the
      // SINGLE_NODE_WRITER access mode to clarify the number of writers
      // for a volume on a single node. Plugins MUST accept and allow
      // use of the SINGLE_NODE_WRITER access mode (subject to the
      // processing rules for NodePublishVolume), when either
      // SINGLE_NODE_SINGLE_WRITER and/or SINGLE_NODE_MULTI_WRITER are
      // supported, in order to permit older COs to continue working.
      SINGLE_NODE_MULTI_WRITER = 5 [(alpha_enum_value) = true];

      // Indicates that Node service supports mounting volumes
      // with provided volume group identifier during node stage
      // or node publish RPC calls.
      VOLUME_MOUNT_GROUP = 6;
    }

    Type type = 1;
  }

  oneof type {
    // RPC that the controller supports.
    RPC rpc = 1;
  }
}
```

##### NodeGetCapabilities Errors

If the plugin is unable to complete the NodeGetCapabilities call successfully, it MUST return a non-ok gRPC code in the gRPC status.

#### `NodeGetInfo`

A Node Plugin MUST implement this RPC call if the plugin has `PUBLISH_UNPUBLISH_VOLUME` controller capability.
The Plugin SHALL assume that this RPC will be executed on the node where the volume will be used.
The CO SHOULD call this RPC for the node at which it wants to place the workload.
The CO MAY call this RPC more than once for a given node.
The SP SHALL NOT expect the CO to call this RPC more than once.
The result of this call will be used by CO in `ControllerPublishVolume`.

```protobuf
message NodeGetInfoRequest {
}

message NodeGetInfoResponse {
  // The identifier of the node as understood by the SP.
  // This field is REQUIRED.
  // This field MUST contain enough information to uniquely identify
  // this specific node vs all other nodes supported by this plugin.
  // This field SHALL be used by the CO in subsequent calls, including
  // `ControllerPublishVolume`, to refer to this node.
  // The SP is NOT responsible for global uniqueness of node_id across
  // multiple SPs.
  // This field overrides the general CSI size limit.
  // The size of this field SHALL NOT exceed 256 bytes. The general
  // CSI size limit, 128 byte, is RECOMMENDED for best backwards
  // compatibility.
  string node_id = 1;

  // Maximum number of volumes that controller can publish to the node.
  // If value is not set or zero CO SHALL decide how many volumes of
  // this type can be published by the controller to the node. The
  // plugin MUST NOT set negative values here.
  // This field is OPTIONAL.
  int64 max_volumes_per_node = 2;

  // Specifies where (regions, zones, racks, etc.) the node is
  // accessible from.
  // A plugin that returns this field MUST also set the
  // VOLUME_ACCESSIBILITY_CONSTRAINTS plugin capability.
  // COs MAY use this information along with the topology information
  // returned in CreateVolumeResponse to ensure that a given volume is
  // accessible from a given node when scheduling workloads.
  // This field is OPTIONAL. If it is not specified, the CO MAY assume
  // the node is not subject to any topological constraint, and MAY
  // schedule workloads that reference any volume V, such that there are
  // no topological constraints declared for V.
  //
  // Example 1:
  //   accessible_topology =
  //     {"region": "R1", "zone": "Z2"}
  // Indicates the node exists within the "region" "R1" and the "zone"
  // "Z2".
  Topology accessible_topology = 3;
}
```

##### NodeGetInfo Errors

If the plugin is unable to complete the NodeGetInfo call successfully, it MUST return a non-ok gRPC code in the gRPC status.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

#### `NodeExpandVolume`

A Node Plugin MUST implement this RPC call if it has `EXPAND_VOLUME` node capability.
This RPC call allows CO to expand volume on a node.

This operation MUST be idempotent.
If a volume corresponding to the specified volume ID is already larger than or equal to the target capacity of the expansion request, the plugin SHOULD reply 0 OK.

`NodeExpandVolume` ONLY supports expansion of already node-published or node-staged volumes on the given `volume_path`.

If plugin has `STAGE_UNSTAGE_VOLUME` node capability then:
* `NodeExpandVolume` MUST be called after successful `NodeStageVolume`.
* `NodeExpandVolume` MAY be called before or after `NodePublishVolume`.

Otherwise `NodeExpandVolume` MUST be called after successful `NodePublishVolume`.

If a plugin only supports expansion via the `VolumeExpansion.OFFLINE` capability, then the volume MUST first be taken offline and expanded via `ControllerExpandVolume` (see `ControllerExpandVolume` for more details), and then node-staged or node-published before it can be expanded on the node via `NodeExpandVolume`.

The `staging_target_path` field is not required, for backwards compatibility, but the CO SHOULD supply it.
Plugins can use this field to determine if `volume_path` is where the volume is published or staged,
and setting this field to non-empty allows plugins to function with less stored state on the node.

```protobuf
message NodeExpandVolumeRequest {
  // The ID of the volume. This field is REQUIRED.
  string volume_id = 1;

  // The path on which volume is available. This field is REQUIRED.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string volume_path = 2;

  // This allows CO to specify the capacity requirements of the volume
  // after expansion. If capacity_range is omitted then a plugin MAY
  // inspect the file system of the volume to determine the maximum
  // capacity to which the volume can be expanded. In such cases a
  // plugin MAY expand the volume to its maximum capacity.
  // This field is OPTIONAL.
  CapacityRange capacity_range = 3;

  // The path where the volume is staged, if the plugin has the
  // STAGE_UNSTAGE_VOLUME capability, otherwise empty.
  // If not empty, it MUST be an absolute path in the root
  // filesystem of the process serving this request.
  // This field is OPTIONAL.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string staging_target_path = 4;

  // Volume capability describing how the CO intends to use this volume.
  // This allows SP to determine if volume is being used as a block
  // device or mounted file system. For example - if volume is being
  // used as a block device the SP MAY choose to skip expanding the
  // filesystem in NodeExpandVolume implementation but still perform
  // rest of the housekeeping needed for expanding the volume. If
  // volume_capability is omitted the SP MAY determine
  // access_type from given volume_path for the volume and perform
  // node expansion. This is an OPTIONAL field.
  VolumeCapability volume_capability = 5;

  // Secrets required by plugin to complete node expand volume request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 6
    [(csi_secret) = true, (alpha_field) = true];
}

message NodeExpandVolumeResponse {
  // The capacity of the volume in bytes. This field is OPTIONAL.
  int64 capacity_bytes = 1;
}
```

##### NodeExpandVolume Errors

| Condition             | gRPC code | Description           | Recovery Behavior                 |
|-----------------------|-----------|-----------------------|-----------------------------------|
| Exceeds capabilities | 3 INVALID_ARGUMENT | Indicates that the CO has specified capabilities not supported by the volume. | Caller MAY verify volume capabilities by calling ValidateVolumeCapabilities and retry with matching capabilities. |
| Volume does not exist | 5 NOT FOUND | Indicates that a volume corresponding to the specified volume_id does not exist. | Caller MUST verify that the volume_id is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |
| Volume in use | 9 FAILED_PRECONDITION | Indicates that the volume corresponding to the specified `volume_id` could not be expanded because it is node-published or node-staged and the underlying filesystem does not support expansion of published or staged volumes. | Caller MUST NOT retry. |
| Unsupported capacity_range | 11 OUT_OF_RANGE | Indicates that the capacity range is not allowed by the Plugin. More human-readable information MAY be provided in the gRPC `status.message` field. | Caller MUST fix the capacity range before retrying. |

## Protocol

### Connectivity

* A CO SHALL communicate with a Plugin using gRPC to access the `Identity`, and (optionally) the `Controller` and `Node` services.
  * proto3 SHOULD be used with gRPC, as per the [official recommendations](http://www.grpc.io/docs/guides/#protocol-buffer-versions).
  * All Plugins SHALL implement the REQUIRED Identity service RPCs.
    Support for OPTIONAL RPCs is reported by the `ControllerGetCapabilities` and `NodeGetCapabilities` RPC calls.
* The CO SHALL provide the listen-address for the Plugin by way of the `CSI_ENDPOINT` environment variable.
  Plugin components SHALL create, bind, and listen for RPCs on the specified listen address.
  * Only UNIX Domain Sockets MAY be used as endpoints.
    This will likely change in a future version of this specification to support non-UNIX platforms.
* All supported RPC services MUST be available at the listen address of the Plugin.

### Security

* The CO operator and Plugin Supervisor SHOULD take steps to ensure that any and all communication between the CO and Plugin Service are secured according to best practices.
* Communication between a CO and a Plugin SHALL be transported over UNIX Domain Sockets.
  * gRPC is compatible with UNIX Domain Sockets; it is the responsibility of the CO operator and Plugin Supervisor to properly secure access to the Domain Socket using OS filesystem ACLs and/or other OS-specific security context tooling.
  * SP’s supplying stand-alone Plugin controller appliances, or other remote components that are incompatible with UNIX Domain Sockets MUST provide a software component that proxies communication between a UNIX Domain Socket and the remote component(s).
    Proxy components transporting communication over IP networks SHALL be responsible for securing communications over such networks.
* Both the CO and Plugin SHOULD avoid accidental leakage of sensitive information (such as redacting such information from log files).

### Debugging

* Debugging and tracing are supported by external, CSI-independent additions and extensions to gRPC APIs, such as [OpenTracing](https://github.com/grpc-ecosystem/grpc-opentracing).

## Configuration and Operation

### General Configuration

* The `CSI_ENDPOINT` environment variable SHALL be supplied to the Plugin by the Plugin Supervisor.
* An operator SHALL configure the CO to connect to the Plugin via the listen address identified by `CSI_ENDPOINT` variable.
* With exception to sensitive data, Plugin configuration SHOULD be specified by environment variables, whenever possible, instead of by command line flags or bind-mounted/injected files.


#### Plugin Bootstrap Example

* Supervisor -> Plugin: `CSI_ENDPOINT=unix:///path/to/unix/domain/socket.sock`.
* Operator -> CO: use plugin at endpoint `unix:///path/to/unix/domain/socket.sock`.
* CO: monitor `/path/to/unix/domain/socket.sock`.
* Plugin: read `CSI_ENDPOINT`, create UNIX socket at specified path, bind and listen.
* CO: observe that socket now exists, establish connection.
* CO: invoke `GetPluginCapabilities`.

#### Filesystem

* Plugins SHALL NOT specify requirements that include or otherwise reference directories and/or files on the root filesystem of the CO.
* Plugins SHALL NOT create additional files or directories adjacent to the UNIX socket specified by `CSI_ENDPOINT`; violations of this requirement constitute "abuse".
  * The Plugin Supervisor is the ultimate authority of the directory in which the UNIX socket endpoint is created and MAY enforce policies to prevent and/or mitigate abuse of the directory by Plugins.

### Supervised Lifecycle Management

* For Plugins packaged in software form:
  * Plugin Packages SHOULD use a well-documented container image format (e.g., Docker, OCI).
  * The chosen package image format MAY expose configurable Plugin properties as environment variables, unless otherwise indicated in the section below.
    Variables so exposed SHOULD be assigned default values in the image manifest.
  * A Plugin Supervisor MAY programmatically evaluate or otherwise scan a Plugin Package’s image manifest in order to discover configurable environment variables.
  * A Plugin SHALL NOT assume that an operator or Plugin Supervisor will scan an image manifest for environment variables.

#### Environment Variables

* Variables defined by this specification SHALL be identifiable by their `CSI_` name prefix.
* Configuration properties not defined by the CSI specification SHALL NOT use the same `CSI_` name prefix; this prefix is reserved for common configuration properties defined by the CSI specification.
* The Plugin Supervisor SHOULD supply all RECOMMENDED CSI environment variables to a Plugin.
* The Plugin Supervisor SHALL supply all REQUIRED CSI environment variables to a Plugin.

##### `CSI_ENDPOINT`

Network endpoint at which a Plugin SHALL host CSI RPC services. The general format is:

    {scheme}://{authority}{endpoint}

The following address types SHALL be supported by Plugins:

    unix:///path/to/unix/socket.sock

Note: All UNIX endpoints SHALL end with `.sock`. See [gRPC Name Resolution](https://github.com/grpc/grpc/blob/master/doc/naming.md).

This variable is REQUIRED.

#### Operational Recommendations

The Plugin Supervisor expects that a Plugin SHALL act as a long-running service vs. an on-demand, CLI-driven process.

Supervised plugins MAY be isolated and/or resource-bounded.

##### Logging

* Plugins SHOULD generate log messages to ONLY standard output and/or standard error.
  * In this case the Plugin Supervisor SHALL assume responsibility for all log lifecycle management.
* Plugin implementations that deviate from the above recommendation SHALL clearly and unambiguously document the following:
  * Logging configuration flags and/or variables, including working sample configurations.
  * Default log destination(s) (where do the logs go if no configuration is specified?)
  * Log lifecycle management ownership and related guidance (size limits, rate limits, rolling, archiving, expunging, etc.) applicable to the logging mechanism embedded within the Plugin.
* Plugins SHOULD NOT write potentially sensitive data to logs (e.g. secrets).

##### Available Services

* Plugin Packages MAY support all or a subset of CSI services; service combinations MAY be configurable at runtime by the Plugin Supervisor.
  * A plugin MUST know the "mode" in which it is operating (e.g. node, controller, or both).
  * This specification does not dictate the mechanism by which mode of operation MUST be discovered, and instead places that burden upon the SP.
* Misconfigured plugin software SHOULD fail-fast with an OS-appropriate error code.

##### Linux Capabilities

* Plugin Supervisor SHALL guarantee that plugins will have `CAP_SYS_ADMIN` capability on Linux when running on Nodes.
* Plugins SHOULD clearly document any additionally required capabilities and/or security context.

##### Namespaces

* A Plugin SHOULD NOT assume that it is in the same [Linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces) as the Plugin Supervisor.
  The CO MUST clearly document the [mount propagation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt) requirements for Node Plugins and the Plugin Supervisor SHALL satisfy the CO’s requirements.

##### Cgroups Isolation

* A Plugin MAY be constrained by cgroups.
* An operator or Plugin Supervisor MAY configure the devices cgroups subsystem to ensure that a Plugin MAY access requisite devices.
* A Plugin Supervisor MAY define resource limits for a Plugin.

##### Resource Requirements

* SPs SHOULD unambiguously document all of a Plugin’s resource requirements.
