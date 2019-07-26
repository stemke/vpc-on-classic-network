---

copyright:
  years: 2017, 2018, 2019
lastupdated: "2019-05-29"

keywords: VRF, router, hypervisor, address prefixes, classic access, implicit router, packet flows, NAT, data flows

subcollection: vpc-on-classic-network


---

{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:pre: .pre}
{:screen: .screen}
{:tip: .tip}
{:note: .note}
{:download: .download}

# VPC の裏側
{: #vpc-behind-the-curtain}

このページでは、VPC ネットワーキングの「裏側」で行われる処理の概念像について詳しく説明します。 ネットワーキングについての基本知識を多少持っている読者を対象にしています。

## ネットワーク分離
{: #network-isolation}

VPC のネットワーク分離は、次の 3 つのレベルで行われます。

* **ハイパーバイザー**: VSI (仮想サーバー・インスタンス) は、ハイパーバイザー自体によって分離されます。 VSI は、同じ VPC 内に存在する VSI を除き、同じハイパーバイザーでホストされている他の VSI に直接アクセスできません。

* **ネットワーク**: 分離は、**仮想ネットワーク ID** (VNI) を使用してネットワーク・レベルで行われます。 これらの ID の適用範囲は、ローカル・ゾーンです。VPC のゾーンに入ってきたデータ・パケット、つまり、VSI から送信され、ハイパーバイザーから入ってくるデータ・パケットと、暗黙ルーティング機能によって送信され、クラウドからゾーンに入ってくるデータ・パケットはすべてこの VNI を追加されます。

ゾーンから出るパケットの VNI は除去されます。パケットが宛先ゾーンに到達し、暗黙ルーティング機能によってそのゾーンに入るときには、必ず、暗黙ルーターがそのゾーンの適切な VNI を追加します。
{: note}

* **ルーター**: _暗黙ルーター機能_ によって、VPC 間の分離が行われます。そのために、クラウドのバックボーンに**仮想ルーティング機能** (VRF) および MPLS (マルチプロトコル・ラベル・スイッチング) 対応 VPN が装備されます。 各 VPC の VRF は固有の ID を持つので、このレベルの分離により各 VPC は、IPv4 アドレス・スペースの独自のコピーにアクセスすることができます。 MPLS VPN により、クラシック・インフラストラクチャー、Direct Link、VPC というすべてのクラウド・エッジを統合できます。

## アドレス接頭部
{: #address-prefixes}

アドレス接頭部とは、宛先 VSI が存在するアベイラビリティー・ゾーンに関係なく、VPC の暗黙ルーティング機能が_宛先 VSI_ を見つけるために使用するサマリー情報です。 アドレス接頭部の主要な機能は、異常なルーティングの発生を防止しながら、MPLS VPN によるルーティングを最適化することです。 VPC 内のすべての VSI が同一 VPC 内の他のすべての VSI からアクセスできるように、VPC 内に作成されたすべてのサブネットが、1 つのアドレス接頭部に含まれている必要があります。

## データ・パケット・フローと暗黙ルーター
{: #data-packet-flows-and-the-implicit-router}

VPC には、6 種類の VSI データ・パケット・フローがあります。 それらのフローを、複雑さが増す順に以下に示します。

* サブネット内部、ホスト内部 (同じハイパーバイザー)
* サブネット内部、ホスト間
* サブネット間、ゾーン内部
* サブネット間、ゾーン間
* VPC 外部のサービス (IaaS アクセスまたは CSE アクセス用)
* VPC 外部のインターネット (インターネット・アクセス用)

**サブネット内部、ホスト内部**のデータ・フロー: 最も単純なデータ・フローです。 同じハイパーバイザー上の VSI 間のパケット・フローであり、パケットはハイパーバイザーを出ていきません。

**サブネット内部、ホスト間**のデータ・フロー: この種類のフローには、ハイパーバイザーを出ていくパケットが含まれます。データ分離を確保するために、各パケットは適切な VNI (仮想化ネットワーク ID) のタグを付けられた後に、宛先 VSI をホストしている宛先ハイパーバイザーに送信されます。宛先ハイパーバイザーは、その VNI を取り除いてから、データ・パケットを宛先 VSI に転送します。

**サブネット間、ゾーン内部**のデータ・フロー: この種類のフローには、VPC の暗黙ルーター機能を利用するパケットが含まれます。この機能は、VPC 内に作成されたすべてのサブネットを接続します。この機能により、データ・パケットが正しい宛先ハイパーバイザーにルーティングされます。 宛先ハイパーバイザーとソース・ハイパーバイザーが異なる場合、データ・パケットは適切な VNI のタグを付けられた後に宛先ハイパーバイザーに送信されます。 そこで VNI が取り除かれた後、データ・パケットは宛先 VSI に転送されます (最後の数ステップは、前の種類のデータ・フローと同じです)。

**サブネット間、ゾーン間**のデータ・フロー: この種類のフローの場合は、暗黙ルーター機能が VNI を除去し、クラウド・バックボーン経由で転送するために VPC の MPLS VPN でパケットを転送します。宛先ゾーンに到達すると、暗黙ルーター機能がデータ・パケットに適切な VNI のタグを付けます。その後、パケットは宛先ハイパーバイザーに転送され、(前述のように) そこで VNI が再び取り除かれ、データ・パケットを宛先 VSI に転送できるようになります。

**VPC 外部のサービス**のデータ・フロー: IaaS サービスまたは IBM Cloud サービス・エンドポイント (CSE) サービス宛てのパケットには、VPC の暗黙ルーター機能が利用され、さらにネットワーク・アドレス変換 (NAT) 機能も利用されます。この変換機能によって VSI アドレスが IPv4 アドレスに置き換えられ、要求されている IaaS サービスまたは CSE サービスに対する VPC が識別されます。

**VPC 外部のインターネット**のデータ・フロー: インターネット宛てのパケットが最も複雑です。 この種類のフローでは、VPC の暗黙ルーター機能が使用されるほか、暗黙ルーターの次の 2 つのネットワーク・アドレス変換 (NAT) 機能のどちらかも使用されます。

  * パブリック・ゲートウェイ機能による明示的な 1 対多の NAT。パブリック・ゲートウェイに接続されているすべてのサブネットに対して機能します。
  * 個々の VSI に割り当てられる 1 対 1 の NAT。

暗黙ルーターは、NAT 変換後に、この種類のインターネット宛てのパケットを、クラウド・バックボーンを使用してインターネットに転送します。

## クラシック・アクセス
{: #classic-access}

VPC の[**クラシック・アクセス**](/docs/vpc-on-classic?topic=vpc-on-classic-setting-up-access-to-your-classic-infrastructure-from-vpc)機能は、VPC の VRF ID として {{site.data.keyword.cloud}} のクラシック・インフラストラクチャーのアカウントの VRF ID を再利用することで実現されます。 この実装により、VPC の暗黙ルーター機能は、クラシック・インフラストラクチャーのアカウントと同じ MPLS VPN に参加できます。 それにより、VPC はクラシック・リソースや、既存の Direct Link 接続を使用してアクセスできるその他のリソースを利用できます。