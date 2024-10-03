# BOLT #1: 基本プロトコル

## 概要

このプロトコルは、個々のメッセージのフレーミングを処理する、認証済みで順序付けられたトランスポートメカニズムを前提としています。Lightning で使用される標準的なトランスポート層は [BOLT #8](08-transport.md) で指定されていますが、上記の保証を満たすトランスポートであれば置き換えることができます。

デフォルトの TCP ポートは使用するネットワークによって異なります。最も一般的なネットワークは以下の通りです：

- Bitcoin メインネットはポート番号 9735 または対応する 16 進数 `0x2607`。
- Bitcoin テストネットはポート番号 19735 (`0x4D17`)。
- Bitcoin シグネットはポート番号 39735 (`0x9B37`)。

LIGHTNING の Unicode コードポイント<sup>[1](#reference-1)</sup>とポートの慣習は、Bitcoin Core の慣習に従うようにしています。

特に指定がない限り、すべてのデータフィールドは符号なしビッグエンディアンです。

## 目次

  * [接続処理と多重化](#connection-handling-and-multiplexing)
  * [Lightning メッセージフォーマット](#lightning-message-format)
  * [タイプ・長さ・値フォーマット](#type-length-value-format)
  * [基本的な型](#fundamental-types)
  * [セットアップメッセージ](#setup-messages)
    * [`init` メッセージ](#the-init-message)
    * [`error` と `warning` メッセージ](#the-error-and-warning-messages)
  * [制御メッセージ](#control-messages)
    * [`ping` と `pong` メッセージ](#the-ping-and-pong-messages)
  * [付録 A: BigSize テストベクトル](#appendix-a-bigsize-test-vectors)
  * [付録 B: タイプ・長さ・値テストベクトル](#appendix-b-type-length-value-test-vectors)
  * [付録 C: メッセージ拡張](#appendix-c-message-extension)
  * [付録 D: 符号付き整数テストベクトル](#appendix-d-signed-integers-test-vectors)
  * [謝辞](#acknowledgments)
  * [参考文献](#references)
  * [著者](#authors)

## 接続処理と多重化

実装は、ピアごとに単一の接続を使用しなければなりません (MUST)。チャネルメッセージ (チャネル ID を含む) は、この単一の接続上で多重化されます。

## Lightning メッセージフォーマット

復号化後、すべての Lightning メッセージは以下の形式になります：

1. `type`: メッセージの種類を示す 2 バイトのビッグエンディアンフィールド
2. `payload`: メッセージの残りを構成する可変長のペイロードで、`type` に一致する形式に従います
3. `extension`: オプションの [TLV ストリーム](#type-length-value-format)

`type` フィールドは `payload` フィールドをどのように解釈するかを示します。各タイプのフォーマットは、このリポジトリ内の仕様によって定義されています。タイプは _it's ok to be odd_ ルールに従っており、ノードは受信者が理解しているか確認せずに _奇数_ 番号のタイプを送信してもかまいません。

メッセージは論理的に 5 つのグループに分類され、設定されている最上位ビットによって順序付けられています：

- セットアップ & コントロール (タイプ `0`-`31`)：接続のセットアップ、コントロール、サポートされている機能、およびエラー報告に関連するメッセージ（以下に説明）
- チャネル (タイプ `32`-`127`)：マイクロペイメントチャネルのセットアップと解体に使用されるメッセージ（[BOLT #2](02-peer-protocol.md) に記載）
- コミットメント (タイプ `128`-`255`)：現在のコミットメントトランザクションの更新に関連するメッセージで、HTLC の追加、取り消し、決済、手数料の更新、署名の交換を含む（[BOLT #2](02-peer-protocol.md) に記載）
- ルーティング (タイプ `256`-`511`)：ノードとチャネルのアナウンスメント、ならびにアクティブなルート探索を含むメッセージ（[BOLT #7](07-routing-gossip.md) に記載）
- カスタム (タイプ `32768`-`65535`)：実験的およびアプリケーション固有のメッセージ

メッセージのサイズは、トランスポート層によって 2 バイトの符号なし整数に収まる必要があり、したがって最大サイズは 65535 バイトです。

送信ノードは：
- 事前の交渉なしに、ここに記載されていない偶数タイプのメッセージを送信してはなりません。
- 事前の交渉なしに、`extension` 内で偶数タイプの TLV レコードを送信してはなりません。
- この仕様でオプションを交渉する場合：
  - そのオプションで注釈されたすべてのフィールドを含めなければなりません。
- カスタムメッセージを定義する際：
  - 他のカスタムタイプとの衝突を避けるためにランダムな `type` を選ぶべきです。
  - [この問題](https://github.com/lightningnetwork/lightning-rfc/issues/716) に記載されている他の実験と競合しない `type` を選ぶべきです。
  - 通常のノードが追加データを無視すべき場合は、奇数の `type` 識別子を選ぶべきです。
  - 通常のノードがメッセージを拒否し、接続を閉じるべき場合は、偶数の `type` 識別子を選ぶべきです。

受信ノード：

- _奇数_ で未知のタイプのメッセージを受信した場合：
  - 受信したメッセージを無視しなければなりません。
- _偶数_ で未知のタイプのメッセージを受信した場合：
  - 接続を閉じなければなりません。
  - チャネルを失敗させてもかまいません。
- 内容に対して長さが不十分な既知のメッセージを受信した場合：
  - 接続を閉じなければなりません。
  - チャネルを失敗させてもかまいません。
- `extension` を含むメッセージを受信した場合：
  - `extension` を無視してもかまいません。
  - それ以外で、`extension` が無効な場合：
    - 接続を閉じなければなりません。
    - チャネルを失敗させてもかまいません。

### 理論的根拠

デフォルトでは `SHA2` と Bitcoin 公開鍵はどちらもビッグエンディアンでエンコードされているため、他のフィールドで異なるエンディアンを使用するのは通常ではありません。

長さは暗号ラッピングによって 65535 バイトに制限されており、プロトコル内のメッセージはどれもその長さを超えることはありません。

_奇数であることは問題ない_ ルールは、クライアントでの交渉や特別なコーディングなしに将来のオプション拡張を可能にします。_extension_ フィールドも同様に、送信者が追加の TLV データを含めることを許可することで将来の拡張を可能にします。メッセージの `payload` がすでに 65535 バイトの最大長を満たしていない場合にのみ、_extension_ フィールドを追加できることに注意してください。

実装は、メッセージデータを 8 バイト境界（ここでの最大自然アライメント要件）に揃えることを好むかもしれません。しかし、タイプフィールドの後に 6 バイトのパディングを追加するのは無駄と考えられました。アライメントは、メッセージを 6 バイトのプレパディングを持つバッファにデコードすることで達成できます。

## タイプ・長さ・値フォーマット

プロトコル全体で、TLV (Type-Length-Value) フォーマットが使用されており、既存のメッセージタイプに新しいフィールドを後方互換的に追加することを可能にしています。

`tlv_record` は単一のフィールドを表し、次の形式でエンコードされます：

* [`bigsize`: `type`]
* [`bigsize`: `length`]
* [`length`: `value`]

`tlv_stream` は（ゼロ個の場合もある）`tlv_record` の一連のもので、エンコードされた `tlv_record` の連結として表されます。既存のメッセージを拡張するために使用される場合、`tlv_stream` は通常、現在定義されているすべてのフィールドの後に配置されます。

`type` は BigSize フォーマットを使用してエンコードされます。これは `tlv_record` におけるメッセージ固有の 64 ビット識別子として機能し、`value` の内容をどのようにデコードするかを決定します。`type` 識別子のうち 2^16 未満のものはこの仕様で使用するために予約されています。2^16 以上の `type` 識別子はカスタムレコードに利用可能です。この仕様で定義されていないレコードはすべてカスタムレコードと見なされます。これには実験的およびアプリケーション固有のメッセージが含まれます。

`length` は BigSize フォーマットを使用してエンコードされ、`value` のバイト単位のサイズを示します。

`value` は完全に `type` に依存し、`type` によって決定されるメッセージ固有のフォーマットに従ってエンコードまたはデコードされるべきです。

### 要件

送信ノード：
 - `tlv_record` を `tlv_stream` 内で厳密に増加する `type` によって順序付けしなければならず、したがって同じ `type` を持つ TLV レコードを複数生成してはなりません。
 - `type` と `length` を最小限にエンコードしなければなりません。
 - カスタムレコード `type` 識別子を定義する際：
   - 他のカスタムタイプとの衝突を避けるためにランダムな `type` 識別子を選ぶべきです。
   - 通常のノードが追加データを無視すべき場合は奇数の `type` 識別子を選ぶべきです。
   - 通常のノードがカスタムレコードを含む tlv ストリーム全体を拒否すべき場合は偶数の `type` 識別子を選ぶべきです。
 - `tlv_record` で冗長な可変長エンコーディングを使用してはなりません。

受信ノード：
 - `type` を解析する前にバイトがゼロ残っている場合：
   - `tlv_stream` の解析を停止しなければなりません。
 - `type` または `length` が最小限にエンコードされていない場合：
   - `tlv_stream` の解析に失敗しなければなりません。
 - デコードされた `type` が厳密に増加していない場合（同じ `type` が二度以上出現する場合を含む）：
   - `tlv_stream` の解析に失敗しなければなりません。
 - `length` がメッセージ内に残っているバイト数を超える場合：
   - `tlv_stream` の解析に失敗しなければなりません。
 - `type` が既知の場合：
   - 既知のエンコーディングを使用して次の `length` バイトをデコードしなければなりません。
   - `length` が `type` の既知のエンコーディングに必要な長さと正確に一致しない場合：
     - `tlv_stream` の解析に失敗しなければなりません。
   - `type` の既知のエンコーディング内の可変長フィールドが最小限でない場合：
     - `tlv_stream` の解析に失敗しなければなりません。
 - それ以外で、`type` が未知の場合：
   - `type` が偶数の場合：
     - `tlv_stream` の解析に失敗しなければなりません。
   - それ以外で、`type` が奇数の場合：
     - 次の `length` バイトを破棄しなければなりません。

### 理論的根拠

TLV を使用する主な利点は、各フィールドがエンコードされた要素の正確なサイズを持っているため、リーダーが理解できない新しいフィールドを無視できることです。TLV がない場合、ノードが特定のフィールドを使用したくない場合でも、そのフィールドの後に続くフィールドのオフセットを決定するために、そのフィールドの解析ロジックを追加することを余儀なくされます。

厳密な単調性制約により、すべての `type` がユニークであり、最大で一度しか現れないことが保証されます。ベクトル、マップ、構造体などの複雑なオブジェクトにマップされるフィールドは、オブジェクトが単一の `tlv_record` 内にシリアライズされるようにエンコードを定義することでそうすべきです。ユニーク性の制約は、他のこととともに、次の最適化を可能にします：
 - 標準的な順序は、エンコードされた `value` に依存せずに定義されます。
 - 標準的な順序は、エンコード時に動的に決定されるのではなく、コンパイル時に知られることができます。
 - 標準的な順序の検証には、より少ない状態が必要で、コストが低くなります。
 - 可変サイズのフィールドは、要素を順次追加して二重コピーのオーバーヘッドを発生させるのではなく、予想されるサイズを事前に予約できます。

`type` と `length` にビッグサイズを使用することで、小さな `type` や短い `value` に対してスペースを節約できます。これにより、通信線上やオニオンペイロード内でアプリケーションデータのためのスペースがより多く残る可能性があります。

すべての `type` は、基礎となる `tlv_record` の標準的なエンコードを作成するために、増加順に現れなければなりません。これは、`tlv_stream` 上で署名を計算する際に重要であり、検証者が署名者と同じメッセージダイジェストを再計算できることを保証します。フィールドのセットに対する標準的な順序は、検証者がフィールドの内容を理解していなくても強制されることができることに注意してください。

ライターは、`tlv_record` 内で冗長な可変長エンコーディングを使用することを避けるべきです。これは、長さを二重にエンコードし、外部の長さを計算するのを複雑にするためです。例えば、可変長のバイト配列を書く場合、`value` は生のバイトのみを含み、追加の内部長を省略すべきです。なぜなら、`tlv_record` はすでに後に続くバイト数を持っているからです。一方で、`tlv_record` が複数の可変長要素を含む場合、これは冗長とは見なされず、受信者が `value` から個々の要素を解析できるようにするために必要です。

## 基本的な型

メッセージ仕様では、さまざまな基本的な型が参照されます。

* `byte`: 8ビットのバイト
* `s8`: 8ビットの符号付き整数
* `u16`: 2バイトの符号なし整数
* `s16`: 2バイトの符号付き整数
* `u32`: 4バイトの符号なし整数
* `s32`: 4バイトの符号付き整数
* `u64`: 8バイトの符号なし整数
* `s64`: 8バイトの符号付き整数

符号付き整数は標準のビッグエンディアンの2の補数表現を使用します（テストベクトルについては[下記](#appendix-d-signed-integers-test-vectors)を参照してください）。

TLVレコードの最終値には、切り詰められた整数を使用することができます。切り詰められた整数の先頭のゼロは省略しなければなりません（MUST）。

* `tu16`: 0から2バイトの切り詰められた符号なし整数
* `tu32`: 0から4バイトの切り詰められた符号なし整数
* `tu64`: 0から8バイトの切り詰められた符号なし整数

金額をエンコードするために使用される場合、これらのフィールドは2100万BTCの上限を遵守しなければなりません（MUST）。

* サトシの金額は最大で `0x000775f05a074000` でなければなりません（MUST）
* ミリサトシの金額は最大で `0x1d24b2dfac520000` でなければなりません（MUST）

以下の便利な型も定義されています。

* `chain_hash`: 32バイトのチェーン識別子（[BOLT #0](00-introduction.md#glossary-and-terminology-guide)を参照）
* `channel_id`: 32バイトのchannel_id（[BOLT #2](02-peer-protocol.md#definition-of-channel-id)を参照）
* `sha256`: 32バイトのSHA2-256ハッシュ
* `signature`: 64バイトのビットコイン楕円曲線署名
* `bip340sig`: [BIP-340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)に基づく64バイトのビットコイン楕円曲線シュノア署名
* `point`: 33バイトの楕円曲線ポイント（[SEC 1 standard](http://www.secg.org/sec1-v2.pdf#subsubsection.2.3.3)に基づく圧縮エンコーディング）
* `short_channel_id`: チャネルを識別する8バイトの値（[BOLT #7](07-routing-gossip.md#definition-of-short-channel-id)を参照）
* `sciddir_or_pubkey`: ノードを参照または識別する9または33バイト
    * 最初のバイトが0または1の場合、8バイトの`short_channel_id`が続き、合計で9バイトになります
        * 最初のバイトが0の場合、これは`short_channel_id`の`channel_announcement`内の`node_id_1`を参照します
        * 最初のバイトが1の場合、これは`short_channel_id`の`channel_announcement`内の`node_id_2`を参照します
          （[BOLT #7](07-routing-gossip.md#the-channel_announcement-message)を参照）
    * 最初のバイトが2または3の場合、値は33バイトの`point`です
* `bigsize`: ビットコインのCompactSizeエンコーディングに似た可変長の符号なし整数ですが、ビッグエンディアンです。[BigSize](#appendix-a-bigsize-test-vectors)で説明されています。
* `utf8`: UTF-8文字列の一部としてのバイト。ライターはこれらの配列が有効なUTF-8文字列であることを保証しなければなりません（MUST）。リーダーはこれらの配列が有効なUTF-8文字列でないメッセージを拒否することができます（MAY）。

## セットアップメッセージ

### `init` メッセージ

認証が完了すると、最初のメッセージでこのノードがサポートまたは必要とする機能が明らかになります。これは再接続の場合でも同様です。

[BOLT #9](09-features.md) は機能のリストを指定しています。各機能は通常 2 ビットで表されます。最下位ビットは 0 で、これは _偶数_ です。次に重要なビットは 1 で、これは _奇数_ です。歴史的な理由から、機能はグローバルおよびローカルの機能ビットマスクに分けられています。

機能は、ピアが現在の接続のために `init` メッセージでそれを設定した場合（偶数または奇数のいずれかとして）*提供*されます。機能は、両方のピアがそれを提供した場合、またはローカルノードが偶数として提供した場合に*交渉*されます。ピアがそれをサポートしていると仮定できます。なぜなら、そうでなければ切断する必要があるからです。

`features` フィールドは 0 でバイトにパディングしなければなりません。

1. タイプ: 16 (`init`)
2. データ:
   * [`u16`:`gflen`]
   * [`gflen*byte`:`globalfeatures`]
   * [`u16`:`flen`]
   * [`flen*byte`:`features`]
   * [`init_tlvs`:`tlvs`]

1. `tlv_stream`: `init_tlvs`
2. タイプ:
    1. タイプ: 1 (`networks`)
    2. データ:
        * [`...*chain_hash`:`chains`]
    1. タイプ: 3 (`remote_addr`)
    2. データ:
        * [`...*byte`:`data`]

オプションの `networks` はノードが関心を持つチェーンを示します。
オプションの `remote_addr` は NAT の問題を回避するために使用できます。

#### 要件

送信ノード:
  - どの接続に対しても最初の Lightning メッセージとして `init` を送信しなければなりません。
  - [BOLT #9](09-features.md) で定義されたように機能ビットを設定しなければなりません。
  - 未定義の機能ビットは 0 に設定しなければなりません。
  - `globalfeatures` で 13 より大きい機能を設定すべきではありません。
  - `features` フィールドを表すのに必要な最小の長さを使用すべきです。
  - ゴシップまたはチャネルを開くすべてのチェーンに `networks` を設定すべきです。
  - ノードが受信者であり、接続が IP 経由で行われた場合、受信接続のリモート IP アドレス（およびポート）を反映するように `remote_addr` を設定すべきです。
  - `remote_addr` を設定する場合:
    - [BOLT 7](07-routing-gossip.md#the-node_announcement-message) で説明されているように、有効な `address descriptor`（1 バイトのタイプとデータ）に設定しなければなりません。
    - プライベートアドレスを `remote_addr` として設定すべきではありません。

受信ノード：

- 他のメッセージを送信する前に `init` を受信するまで待たなければなりません。
- 2 つの機能ビットマップをビット単位の OR 演算で結合し、1 つの論理的な `features` マップにしなければなりません。
- [BOLT #9](09-features.md) に指定されているように、既知の機能ビットに応答しなければなりません。
- 非ゼロの未知の _奇数_ 機能ビットを受信した場合：
  - ビットを無視しなければなりません。
- 非ゼロの未知の _偶数_ 機能ビットを受信した場合：
  - 接続を閉じなければなりません。
- 共通のチェーンを含まない `networks` を受信した場合：
  - 接続を閉じてもかまいません。
- 機能ベクトルがすべての既知の推移的依存関係を設定していない場合：
  - 接続を閉じなければなりません。
- `remote_addr` を使用して `node_announcement` を更新してもかまいません。

#### 理論的根拠

ここには以前、2 つの機能ビットフィールドがありましたが、後方互換性のために現在は 1 つに結合されています。

このセマンティクスにより、将来の非互換な変更と後方互換な変更の両方が可能になります。ビットは一般にペアで割り当てられるべきであり、オプションの機能が後に必須になる可能性があります。

ノードは、機能が互換性がない場合にエラー診断を簡素化するために、他のノードの機能の受信を待ちます。

すべてのネットワークが同じポートを共有していますが、ほとんどの実装は単一のネットワークのみをサポートしているため、`networks` フィールドはノードが誤って自分の好むネットワークについての更新を受け取ると信じたり、チャネルを開くことができると信じたりすることを避けます。

### `error` と `warning` メッセージ

診断を簡素化するために、ピアに何かが間違っていることを伝えるのが有用なことがよくあります。

1. タイプ: 17 (`error`)
2. データ:
   * [`channel_id`:`channel_id`]
   * [`u16`:`len`]
   * [`len*byte`:`data`]

1. タイプ: 1 (`warning`)
2. データ:
   * [`channel_id`:`channel_id`]
   * [`u16`:`len`]
   * [`len*byte`:`data`]

#### 要件

チャネルは `channel_id` によって参照されます。ただし、`channel_id` が 0（すなわち、すべてのバイトが 0）の場合は、すべてのチャネルを指します。

チャネル確立 v1 (`open_channel`) を使用する資金提供ノード：
- `funding_created` メッセージを含む以前に送信されたすべてのエラーメッセージについて：
  - `channel_id` の代わりに `temporary_channel_id` を使用しなければなりません。

チャネル確立 v1（`accept_channel`）を使用する資金提供者ノード：

- `funding_signed` メッセージより前に送信されるすべてのエラーメッセージについて：
  - `channel_id` の代わりに `temporary_channel_id` を使用しなければなりません。

チャネル確立 v2（`open_channel2`）を使用するオープナーノード：

- `accept_channel2` メッセージが受信される前に送信されるすべてのエラーメッセージについて：
  - `channel_id` の代わりに `temporary_channel_id` を使用しなければなりません。

チャネル確立 v2（`open_channel2`）を使用するアクセプターノード：

- `accept_channel2` メッセージより前に送信されるすべてのエラーメッセージについて：
  - `channel_id` の代わりに `temporary_channel_id` を使用しなければなりません。

送信ノード：

- プロトコル違反やチャネルが使用不能になる内部エラー、またはさらなる通信が不可能になる場合には、`error` を送信するべきです。
- 未知のチャネルに関連するタイプ `32`-`255` のメッセージに対しては、未知の `channel_id` を含む `error` を送信するべきです。
- `error` を送信する際：
  - エラーメッセージで参照されるチャネルを失敗させなければなりません。
  - すべてのチャネルを示すために `channel_id` を全ゼロに設定してもかまいません。
- `warning` を送信する際：
  - 特定のチャネルに関連しない警告の場合、`channel_id` を全ゼロに設定してもかまいません。
- 空の `data` フィールドを送信してもかまいません。
- 失敗が無効な署名チェックによって引き起こされた場合：
  - `funding_created`、`funding_signed`、`closing_signed`、または `commitment_signed` メッセージに対する返信として、生の16進エンコードされたトランザクションを含めるべきです。

受信ノード：

- `error` を受信した場合：
  - `channel_id` が全ゼロの場合：
    - 送信ノードとのすべてのチャネルを失敗させなければなりません。
  - それ以外の場合：
    - そのチャネルが送信ノードとのものである場合、`channel_id` で参照されるチャネルを失敗させなければなりません。
- `warning` を受信した場合：
  - 後で診断するためにメッセージを記録するべきです。
  - この時点で許可されている場合、`shutdown` を試みてもかまいません。
- `channel_id` によって参照される既存のチャネルがない場合：
  - メッセージを無視しなければなりません。
- `data` が印刷可能な ASCII 文字（参考：印刷可能な文字セットにはバイト値 32 から 126 が含まれます）のみで構成されていない場合：
  - `data` をそのまま印刷すべきではありません。

#### 理由

回復不能なエラーが発生した場合、会話を中止する必要があります。接続が単に切断されると、ピアが接続を再試行するかもしれません。また、プロトコル違反を診断のために説明することも有用です。これは、片方のピアにバグがあることを示します。

一方で、エラーメッセージの過剰使用は、実装がそれを無視することにつながりました（さもなければ高価なチャネルの破壊を避けるため）、そのため「警告」メッセージが追加され、誤ったエラーに対してある程度の再試行や回復を可能にしました。

情報漏洩を防ぐために、本番環境ではエラーを区別しないことが賢明かもしれません。そのため、オプションの `data` フィールドがあります。

## 制御メッセージ

### `ping` と `pong` メッセージ

長期間にわたる TCP 接続の存在を許可するために、アプリケーションレベルで TCP 接続を維持することが必要な場合があります。このようなメッセージは、トラフィックパターンの難読化も可能にします。

1. タイプ: 18 (`ping`)
2. データ:
    * [`u16`:`num_pong_bytes`]
    * [`u16`:`byteslen`]
    * [`byteslen*byte`:`ignored`]

`pong` メッセージは、`ping` メッセージを受信したときに送信されるべきです。これは返信として機能し、接続を維持しつつ、受信者がまだアクティブであることを明示的に通知します。受信した `ping` メッセージ内で、送信者は `pong` メッセージのデータペイロードに含めるバイト数を指定します。

1. タイプ: 19 (`pong`)
2. データ:
    * [`u16`:`byteslen`]
    * [`byteslen*byte`:`ignored`]

#### 要件

`ping` メッセージを送信するノード:
  - `ignored` を 0 に設定するべきです。
  - `ignored` を秘密情報や初期化されたメモリの一部などの機密データに設定してはなりません。
  - 対応する `pong` を受信しない場合:
    - ネットワーク接続を閉じてもよいです、
      - この場合、チャネルを失敗させてはなりません。

`pong` メッセージを送信するノード:
  - `ignored` を 0 に設定するべきです。
  - `ignored` を秘密情報や初期化されたメモリの一部などの機密データに設定してはなりません。

`ping` メッセージを受信するノード:
  - `num_pong_bytes` が 65532 未満の場合:
    - `byteslen` を `num_pong_bytes` に等しくして `pong` メッセージを送信しなければなりません。
  - それ以外の場合（`num_pong_bytes` が 65532 未満では**ない**場合）:
    - `ping` を無視しなければなりません。

ノードが `pong` メッセージを受信する場合：
  - `byteslen` が送信した `ping` の `num_pong_bytes` 値に対応していない場合：
    - 接続を閉じても構いません（MAY）。

### 理論的根拠

最大のメッセージサイズは 65535 バイトです。したがって、最大の合理的な `byteslen` は 65531 です。これは、タイプフィールド（`pong`）と `byteslen` 自体を考慮するためです。これにより、返信を送信しないことを示すための便利なカットオフとして `num_pong_bytes` を使用できます。

ネットワーク内のノード間の接続は長期間にわたる可能性があります。なぜなら、支払いチャネルは無期限の寿命を持つからです。しかし、接続の寿命のかなりの部分で新しいデータが交換されない可能性があります。また、いくつかのプラットフォームでは、Lightning クライアントが事前の警告なしにスリープ状態にされる可能性があります。したがって、接続の反対側の生存性を確認し、確立された接続をアクティブに保つために、別個の `ping` メッセージが使用されます。

さらに、送信者が受信者に特定のバイト数の応答を要求する機能により、ネットワーク上のノードは_合成_トラフィックを作成できます。このようなトラフィックは、ノードがそれぞれのチャネルに実際の更新を適用せずに、典型的な交換のトラフィックパターンを偽装できるため、パケットおよびタイミング分析に対する部分的な防御に使用できます。

[BOLT #4](04-onion-routing.md) で定義されたオニオンルーティングプロトコルと組み合わせることで、慎重に統計的に駆動された合成トラフィックは、ネットワーク内の参加者のプライバシーをさらに強化するのに役立ちます。

`ping` フラッディングに対する限定的な予防策が推奨されますが、ネットワーク遅延のためにある程度の余地が与えられています。着信トラフィックのフラッディングには他の方法もあることに注意してください（例：_奇数_の未知のメッセージタイプを送信する、またはすべてのメッセージを最大限にパディングする）。

最後に、定期的な `ping` メッセージの使用は、[BOLT #8](08-transport.md) で指定されているように頻繁なキーのローテーションを促進します。

## 付録 A: BigSize テストベクトル

以下のテストベクトルは、TLV フォーマットで使用される BigSize 実装の正確性を確認するために使用できます。このフォーマットは、ビットコインで使用される CompactSize エンコーディングと同一ですが、マルチバイト値のリトルエンディアンエンコーディングをビッグエンディアンに置き換えています。


BigSize でエンコードされた値は、整数のサイズに応じて 1、3、5、または 9 バイトのエンコードを生成します。エンコードは、`uint64` 値 `x` を取り、次のように生成する区分的な関数です：
```
        uint8(x)                if x < 0xfd
        0xfd + be16(uint16(x))  if x < 0x10000
        0xfe + be32(uint32(x))  if x < 0x100000000
        0xff + be64(x)          otherwise.
```

ここで `+` は連結を示し、`be16`、`be32`、および `be64` は、それぞれ 16、32、および 64 ビット整数の入力のビッグエンディアンエンコードを生成します。

値がより少ないバイト数でエンコードできない場合、その値は _最小限にエンコードされている_ と言います。たとえば、5 バイトを占める BigSize エンコードで、その値が 0x10000 未満である場合は、最小限にエンコードされていません。BigSize でデコードされたすべての値は、最小限にエンコードされていることを確認する必要があります。

### BigSize デコードテスト

以下は、BigSize デコードテストを実行する方法の例です。
```golang
func testReadBigSize(t *testing.T, test bigSizeTest) {
        var buf [8]byte 
        r := bytes.NewReader(test.Bytes)
        val, err := tlv.ReadBigSize(r, &buf)
        if err != nil && err.Error() != test.ExpErr {
                t.Fatalf("expected decoding error: %v, got: %v",
                        test.ExpErr, err)
        }

        // If we expected a decoding error, there's no point checking the value.
        if test.ExpErr != "" {
                return
        }

        if val != test.Value {
                t.Fatalf("expected value: %d, got %d", test.Value, val)
        }
}
```

正しい実装は、これらのテストベクトルに対して合格する必要があります：
```json
[
    {
        "name": "zero",
        "value": 0,
        "bytes": "00"
    },
    {
        "name": "one byte high",
        "value": 252,
        "bytes": "fc"
    },
    {
        "name": "two byte low",
        "value": 253,
        "bytes": "fd00fd"
    },
    {
        "name": "two byte high",
        "value": 65535,
        "bytes": "fdffff"
    },
    {
        "name": "four byte low",
        "value": 65536,
        "bytes": "fe00010000"
    },
    {
        "name": "four byte high",
        "value": 4294967295,
        "bytes": "feffffffff"
    },
    {
        "name": "eight byte low",
        "value": 4294967296,
        "bytes": "ff0000000100000000"
    },
    {
        "name": "eight byte high",
        "value": 18446744073709551615,
        "bytes": "ffffffffffffffffff"
    },
    {
        "name": "two byte not canonical",
        "value": 0,
        "bytes": "fd00fc",
        "exp_error": "decoded bigsize is not canonical"
    },
    {
        "name": "four byte not canonical",
        "value": 0,
        "bytes": "fe0000ffff",
        "exp_error": "decoded bigsize is not canonical"
    },
    {
        "name": "eight byte not canonical",
        "value": 0,
        "bytes": "ff00000000ffffffff",
        "exp_error": "decoded bigsize is not canonical"
    },
    {
        "name": "two byte short read",
        "value": 0,
        "bytes": "fd00",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "four byte short read",
        "value": 0,
        "bytes": "feffff",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "eight byte short read",
        "value": 0,
        "bytes": "ffffffffff",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "one byte no read",
        "value": 0,
        "bytes": "",
        "exp_error": "EOF"
    },
    {
        "name": "two byte no read",
        "value": 0,
        "bytes": "fd",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "four byte no read",
        "value": 0,
        "bytes": "fe",
        "exp_error": "unexpected EOF"
    },
    {
        "name": "eight byte no read",
        "value": 0,
        "bytes": "ff",
        "exp_error": "unexpected EOF"
    }
]
```

### BigSize エンコードテスト

以下は、BigSize エンコードテストを実行する方法の例です。
```golang
func testWriteBigSize(t *testing.T, test bigSizeTest) {
        var (
                w   bytes.Buffer
                buf [8]byte
        )
        err := tlv.WriteBigSize(&w, test.Value, &buf)
        if err != nil {
                t.Fatalf("unable to encode %d as bigsize: %v",
                        test.Value, err)
        }

        if bytes.Compare(w.Bytes(), test.Bytes) != 0 {
                t.Fatalf("expected bytes: %v, got %v",
                        test.Bytes, w.Bytes())
        }
}
```

正しい実装は、以下のテストベクトルに対して合格する必要があります：
```json
[
    {
        "name": "zero",
        "value": 0,
        "bytes": "00"
    },
    {
        "name": "one byte high",
        "value": 252,
        "bytes": "fc"
    },
    {
        "name": "two byte low",
        "value": 253,
        "bytes": "fd00fd"
    },
    {
        "name": "two byte high",
        "value": 65535,
        "bytes": "fdffff"
    },
    {
        "name": "four byte low",
        "value": 65536,
        "bytes": "fe00010000"
    },
    {
        "name": "four byte high",
        "value": 4294967295,
        "bytes": "feffffffff"
    },
    {
        "name": "eight byte low",
        "value": 4294967296,
        "bytes": "ff0000000100000000"
    },
    {
        "name": "eight byte high",
        "value": 18446744073709551615,
        "bytes": "ffffffffffffffffff"
    }
]
```

## Appendix B: Type-Length-Value テストベクトル

以下のテストは、2 つの別々の TLV 名前空間 n1 と n2 が存在することを前提としています。

n1 名前空間は、以下の TLV タイプをサポートします：

1. `tlv_stream`: `n1`
2. types:
   1. type: 1 (`tlv1`)
   2. data:
     * [`tu64`:`amount_msat`]
   1. type: 2 (`tlv2`)
   2. data:
     * [`short_channel_id`:`scid`]
   1. type: 3 (`tlv3`)
   2. data:
     * [`point`:`node_id`]
     * [`u64`:`amount_msat_1`]
     * [`u64`:`amount_msat_2`]
   1. type: 254 (`tlv4`)
   2. data:
     * [`u16`:`cltv_delta`]

n2 名前空間は、以下の TLV タイプをサポートします：

1. `tlv_stream`: `n2`
2. types:
   1. type: 0 (`tlv1`)
   2. data:
     * [`tu64`:`amount_msat`]
   1. type: 11 (`tlv2`)
   2. data:
     * [`tu32`:`cltv_expiry`]

### TLV デコード失敗

任意の名前空間で次の TLV ストリームはデコード失敗を引き起こすべきです：


1. 無効なストリーム: 0xfd
2. 理由: 型が切り詰められています

1. 無効なストリーム: 0xfd01
2. 理由: 型が切り詰められています

1. 無効なストリーム: 0xfd0001 00
2. 理由: 型が最小限にエンコードされていません

1. 無効なストリーム: 0xfd0101
2. 理由: 長さが不足しています

1. 無効なストリーム: 0x0f fd
2. 理由: (長さが切り詰められています)

1. 無効なストリーム: 0x0f fd26
2. 理由: (長さが切り詰められています)

1. 無効なストリーム: 0x0f fd2602
2. 理由: 値が不足しています

1. 無効なストリーム: 0x0f fd0001 00
2. 理由: 長さが最小限にエンコードされていません

1. 無効なストリーム: 0x0f fd0201 000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
2. 理由: 値が切り詰められています

以下のいずれかの名前空間の TLV ストリームは、デコードエラーを引き起こすべきです：

1. 無効なストリーム: 0x12 00
2. 理由: 未知の偶数型です。

1. 無効なストリーム: 0xfd0102 00
2. 理由: 未知の偶数型です。

1. 無効なストリーム: 0xfe01000002 00
2. 理由: 未知の偶数型です。

1. 無効なストリーム: 0xff0100000000000002 00
2. 理由: 未知の偶数型です。

名前空間 `n1` の以下の TLV ストリームは、デコードエラーを引き起こすべきです：

1. 無効なストリーム: 0x01 09 ffffffffffffffffff
2. 理由: `n1` の `tlv1` のエンコード長を超えています。

1. 無効なストリーム: 0x01 01 00
2. 理由: `n1` の `tlv1` の `amount_msat` のエンコードが最小限ではありません

1. 無効なストリーム: 0x01 02 0001
2. 理由: `n1` の `tlv1` の `amount_msat` のエンコードが最小限ではありません

1. 無効なストリーム: 0x01 03 000100
2. 理由: `n1` の `tlv1` の `amount_msat` のエンコードが最小限ではありません

1. 無効なストリーム: 0x01 04 00010000
2. 理由: `n1` の `tlv1` の `amount_msat` のエンコードが最小限ではありません

1. 無効なストリーム: 0x01 05 0001000000
2. 理由: `n1` の `tlv1` の `amount_msat` のエンコードが最小限ではありません


1. 無効なストリーム: 0x01 06 000100000000
2. 理由: `n1` の `tlv1` の `amount_msat` のエンコーディングが最小ではありません

1. 無効なストリーム: 0x01 07 00010000000000
2. 理由: `n1` の `tlv1` の `amount_msat` のエンコーディングが最小ではありません

1. 無効なストリーム: 0x01 08 0001000000000000
2. 理由: `n1` の `tlv1` の `amount_msat` のエンコーディングが最小ではありません

1. 無効なストリーム: 0x02 07 01010101010101
2. 理由: `n1` の `tlv2` のエンコーディング長より短いです。

1. 無効なストリーム: 0x02 09 010101010101010101
2. 理由: `n1` の `tlv2` のエンコーディング長より長いです。

1. 無効なストリーム: 0x03 21 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb
2. 理由: `n1` の `tlv3` のエンコーディング長より短いです。

1. 無効なストリーム: 0x03 29 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb0000000000000001
2. 理由: `n1` の `tlv3` のエンコーディング長より短いです。

1. 無効なストリーム: 0x03 30 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb000000000000000100000000000001
2. 理由: `n1` の `tlv3` のエンコーディング長より短いです。

1. 無効なストリーム: 0x03 31 043da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb00000000000000010000000000000002
2. 理由: `n1` の `node_id` が有効なポイントではありません。

1. 無効なストリーム: 0x03 32 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb0000000000000001000000000000000001
2. 理由: `n1` の `tlv3` のエンコーディング長より長いです。

1. 無効なストリーム: 0xfd00fe 00
2. 理由: `n1` の `tlv4` のエンコーディング長より短いです。

1. 無効なストリーム: 0xfd00fe 01 01
2. 理由: `n1` の `tlv4` のエンコーディング長より短いです。

1. 無効なストリーム: 0xfd00fe 03 010101
2. 理由: `n1` の `tlv4` のエンコーディング長より長いです。

1. 無効なストリーム: 0x00 00
2. 理由: `n1` の名前空間に未知の偶数フィールドがあります。

### TLV Decoding Successes

以下の名前空間内の TLV ストリームは正しくデコードされ、無視されるべきです：

1. 有効なストリーム: 0x
2. 説明: 空のメッセージ

1. 有効なストリーム: 0x21 00
2. 説明: 未知の奇数タイプ。

1. 有効なストリーム: 0xfd0201 00
2. 説明: 未知の奇数タイプ。

1. 有効なストリーム: 0xfd00fd 00
2. 説明: 未知の奇数タイプ。


1. 有効なストリーム: 0xfd00ff 00
2. 説明: 不明な奇数タイプ。

1. 有効なストリーム: 0xfe02000001 00
2. 説明: 不明な奇数タイプ。

1. 有効なストリーム: 0xff0200000000000001 00
2. 説明: 不明な奇数タイプ。

以下の `n1` 名前空間の TLV ストリームは、ここで示されている値で正しくデコードされるべきです：

1. 有効なストリーム: 0x01 00
2. 値: `tlv1` `amount_msat`=0

1. 有効なストリーム: 0x01 01 01
2. 値: `tlv1` `amount_msat`=1

1. 有効なストリーム: 0x01 02 0100
2. 値: `tlv1` `amount_msat`=256

1. 有効なストリーム: 0x01 03 010000
2. 値: `tlv1` `amount_msat`=65536

1. 有効なストリーム: 0x01 04 01000000
2. 値: `tlv1` `amount_msat`=16777216

1. 有効なストリーム: 0x01 05 0100000000
2. 値: `tlv1` `amount_msat`=4294967296

1. 有効なストリーム: 0x01 06 010000000000
2. 値: `tlv1` `amount_msat`=1099511627776

1. 有効なストリーム: 0x01 07 01000000000000
2. 値: `tlv1` `amount_msat`=281474976710656

1. 有効なストリーム: 0x01 08 0100000000000000
2. 値: `tlv1` `amount_msat`=72057594037927936

1. 有効なストリーム: 0x02 08 0000000000000226
2. 値: `tlv2` `scid`=0x0x550

1. 有効なストリーム: 0x03 31 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb00000000000000010000000000000002
2. 値: `tlv3` `node_id`=023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb `amount_msat_1`=1 `amount_msat_2`=2

1. 有効なストリーム: 0xfd00fe 02 0226
2. 値: `tlv4` `cltv_delta`=550

### TLV ストリームのデコード失敗

無効なストリームを有効なストリームに追加すると、デコード失敗を引き起こすべきです。

より高い番号の有効なストリームをより低い番号の有効なストリームに追加しても、デコード失敗を引き起こすべきではありません。

さらに、`n1` 名前空間の以下の TLV ストリームはデコード失敗を引き起こすべきです：

1. 無効なストリーム: 0x02 08 0000000000000226 01 01 2a
2. 理由: 有効な TLV レコードだが無効な順序

1. 無効なストリーム: 0x02 08 0000000000000231 02 08 0000000000000451
2. 理由: 重複した TLV タイプ

1. 無効なストリーム: 0x1f 00 0f 01 2a
2. 理由: 有効（無視される）な TLV レコードだが無効な順序

1. 無効なストリーム: 0x1f 00 1f 01 2a
2. 理由: 重複した TLV タイプ（無視される）


`n2` 名前空間における次の TLV ストリームはデコード失敗を引き起こすべきです：

1. 無効なストリーム: 0xffffffffffffffffff 00 00 00
2. 理由: 有効な TLV レコードですが、順序が無効です

## Appendix C: メッセージ拡張 {/*appendix-c-message-extension*/}

このセクションでは、`init` メッセージにおける有効および無効な拡張の例を示します。これらの例の基本の `init` メッセージ（拡張なし）は `0x001000000000`（すべての機能がオフ）です。

次の `init` メッセージは有効です：

- `0x001000000000`: 拡張が提供されていない
- `0x001000000000c9012acb0104`: 拡張には 2 つの未知の _奇数_ TLV レコード（タイプ `0xc9` と `0xcb`）が含まれています

次の `init` メッセージは無効です：

- `0x00100000000001`: 拡張が存在しますが切り捨てられています
- `0x001000000000ca012a`: 拡張には未知の _偶数_ TLV レコードが含まれています（TLV タイプ `0xca` が未知であると仮定）
- `0x001000000000c90101c90102`: 拡張 TLV ストリームが無効です（TLV レコードタイプ `0xc9` が重複）

メッセージが署名される場合、_拡張_ は署名されたバイトの一部であることに注意してください。ノードは、たとえそれらを理解できなくても、署名を正しく検証するために _拡張_ バイトを保存する必要があります。

## Appendix D: 符号付き整数のテストベクトル {/*appendix-d-signed-integers-test-vectors*/}

次のテストベクトルは、符号付き整数（`s8`、`s16`、`s32`、および `s64`）がビッグエンディアンの 2 の補数を使用してどのようにエンコードされるかを示しています。

```json
[
    {
        "value": 0,
        "bytes": "00"
    },
    {
        "value": 42,
        "bytes": "2a"
    },
    {
        "value": -42,
        "bytes": "d6"
    },
    {
        "value": 127,
        "bytes": "7f"
    },
    {
        "value": -128,
        "bytes": "80"
    },
    {
        "value": 128,
        "bytes": "0080"
    },
    {
        "value": -129,
        "bytes": "ff7f"
    },
    {
        "value": 15000,
        "bytes": "3a98"
    },
    {
        "value": -15000,
        "bytes": "c568"
    },
    {
        "value": 32767,
        "bytes": "7fff"
    },
    {
        "value": -32768,
        "bytes": "8000"
    },
    {
        "value": 32768,
        "bytes": "00008000"
    },
    {
        "value": -32769,
        "bytes": "ffff7fff"
    },
    {
        "value": 21000000,
        "bytes": "01406f40"
    },
    {
        "value": -21000000,
        "bytes": "febf90c0"
    },
    {
        "value": 2147483647,
        "bytes": "7fffffff"
    },
    {
        "value": -2147483648,
        "bytes": "80000000"
    },
    {
        "value": 2147483648,
        "bytes": "0000000080000000"
    },
    {
        "value": -2147483649,
        "bytes": "ffffffff7fffffff"
    },
    {
        "value": 500000000000,
        "bytes": "000000746a528800"
    },
    {
        "value": -500000000000,
        "bytes": "ffffff8b95ad7800"
    },
    {
        "value": 9223372036854775807,
        "bytes": "7fffffffffffffff"
    },
    {
        "value": -9223372036854775808,
        "bytes": "8000000000000000"
    }
]
```

## Acknowledgments {/*acknowledgments*/}

[ TODO: (roasbeef); fin ]

## References {/*references*/}

1. <a id="reference-1">http://www.unicode.org/charts/PDF/U2600.pdf</a>

## Authors {/*authors*/}

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) の下でライセンスされています。