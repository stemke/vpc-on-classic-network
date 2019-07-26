---

copyright:
  years: 2017, 2018, 2019
lastupdated: "2019-05-14"

keywords: VPC, subnet, address prefixes, design, addressing

subcollection: vpc-on-classic-network

---

{:shortdesc: .shortdesc}
{:codeblock: .codeblock}
{:screen: .screen}
{:new_window: target="_blank"}
{:pre: .pre}
{:tip: .tip}
{:note: .note}
{:table: .aria-labeledby="caption"}
{:download: .download}


# VPC에 대한 주소 지정 플랜 디자인 
{: #vpc-addressing-plan-design}

VPC를 디자인하는 첫 번째 단계는 주소 지정 플랜을 디자인하는 것이어야 합니다. 올바르게 실행되는 주소 지정 플랜에는 다음과 같은 두 가지 목표가 있습니다.

* VPC 인스턴스의 통신 요구사항을 충족합니다.
* 향후 확장을 위해 유연성을 유지합니다. 

이 문서에서는 각 티어가 다중 구역에서 지원되는 3-티어 웹 애플리케이션에 대한 주소 지정 플랜을 디자인하는 예를 제공합니다. 

각 {{site.data.keyword.cloud}} VPC는 특정 지역에 배치되지만 VPC는 해당 지역 내의 모든 구역에 걸쳐 있을 수 있습니다. {{site.data.keyword.cloud_notm}} VPC는 모든 구역에 대한 기본 주소 접두부를 정의합니다. 이러한 주소 접두부를 사용하면 [내재적 라우터](/docs/vpc-on-classic?topic=vpc-on-classic-vpc-glossary#implicit-router)에 필요한 구역 관련 라우팅 정보를 제공하여 여러 구역에 있는 {{site.data.keyword.cloud_notm}} VPC 인스턴스 간에 통신할 수 있습니다.

애플리케이션이 클라우드에 완전히 포함되었는지 여부 또는 애플리케이션의 일부가 다른 위치에서 실행 중인지 여부에 관계없이 동일한 디자인 단계가 포함됩니다.
{: tip}

## 디자인 고려사항 및 가정
{: #design-considerations-and-assumptions}

애플리케이션에 대한 주소 지정 플랜을 디자인할 때 기본 고려사항은 단일 구역 내에서 서브넷을 작성하는 데 사용되는 CIDR 블록을 가능한 연속적으로 유지하는 것입니다. 이를 통해 디자이너가 서브넷이 단일 주소 접두부로 요약될 수 있도록 하며 향후 확장을 위한 공간을 확보할 수 있습니다.

다른 고려사항은 수평적 확장을 위해 서브넷에 필요할 수 있는 사용 가능한 주소의 수입니다. 다음 표에는 지정된 CIDR 블록 크기를 기반으로 서브넷에서 사용 가능한 주소의 수가 나열되어 있습니다.

| CIDR 블록 크기 | 사용 가능한 주소 |
| --------------- | ------------------- |
|      /22        |        1019         |
|      /23        |         507         |
|      /24        |         251         |
|      /25        |         123         |
|      /26        |          59         |
|      /27        |          27         |
|      /28        |          11         |

이 예제에서는 이러한 두 가지 고려사항에 기반하여 다음과 같이 가정합니다.

* 이 예제에서는 RFC 1918 주소 172.16.0.0/12 블록의 CIDR 범위가 모든 서브넷에 사용됩니다.
* 애플리케이션의 프리젠테이션 계층이 REST API 위의 얇은 판이라고 가정합니다. 따라서 수평적 확장은 프리젠테이션 티어에 영향을 미치는 것보다 중간 티어에 더 많은 영향을 미칩니다.

## 각 티어의 서브넷 크기 판별
{: #determine-each-tier-s-subnet-size}

다음 단계는 사용 가능한 주소의 관점에서 각 티어의 서브넷 크기를 판별하는 것입니다. 애플리케이션의 각 티어는 각 구역에 존재하므로 각 구역에는 각 구역에는 세 개의 서브넷이 필요합니다.

다음은 각 티어의 서브넷 크기를 계획할 때 사용하는 고려사항입니다.

* 데이터베이스 티어(백엔드)는 동적 스케일링이 필요할 가능성이 가장 낮은 티어이므로 이러한 서브넷은 가장 작을 수 있습니다. 즉, 이러한 서브넷에는 가장 적은 수의 사용 가능한 주소가 포함될 수 있습니다. 
    * _이 예제에서는 이 티어에서 27개의 주소를 허용하는 `/27` CIDR 블록을 사용합니다._
* 중간 티어는 동적 스케일링이 필요할 가능성이 가장 높기 때문에 이러한 서브넷이 가장 큽니다. 즉, 가장 많은 수의 사용 가능한 주소가 포함되어야 합니다. 
    * _이 예제에서는 이 티어에서 123개의 주소를 허용하는 `/25` CIDR 블록을 사용합니다._
* 프론트 엔드 티어는 이 사이에 있습니다. 중간 티어만큼의 주소가 필요하지는 않지만 데이터베이스 티어보다 많은 주소가 필요합니다. 
    * _이 예제에서는 이 티어에서 59개의 주소를 허용하는 `/26` CIDR 블록을 사용합니다._

## 서브넷 결합 및 주소 접두부 선택
{: #combining-the-subnets-and-selecting-the-address-prefixes}

각 구역에 허용 가능한 주소 접두부를 선택하려면 각 티어에 있는 세 개의 서브넷을 모두 수용할 수 있을 만큼의 서브넷 크기가 필요하며, 수평적 확장 및 향후 확장을 위한 공간을 남겨두어야 합니다.  

`/24` 주소 접두부는 이러한 세 개의 서브넷을 결합(27 + 123 + 59)할 수 있는 가장 작은 접두부입니다. 가장 작은 크기가 아니라 다음으로 큰 서브넷 크기를 선택하는 것이 가장 좋습니다. 다음으로 큰 서브넷 크기(`/23`)를 지정하면 동일한 주소 접두부 내에서 각 계층에 새 서브넷을 추가할 수 있으므로 이전에 지정된 한계를 넘어서는 수평적 확장이 허용됩니다.

이제 올바른 서브넷 크기를 판별했으므로 실제 주소 접두부를 각 구역마다 하나씩 지정할 수 있습니다.

|구역 |주소 접두부 |
| ------ | --------------- |
|구역 1 |`172.16.0.0/23` |
|구역 2 |`172.16.2.0/23` |
|구역 3 |`172.16.4.0/23` |

또한 이를 기초로 하여 각 구역 내에 세 개의 서브넷을 지정할 수 있습니다.

|구역 |티어 |서브넷 CIDR |
| ------ | -------- | ----------------- |
|구역 1 |중간 |`172.16.0.0/25`  |
|구역 1 |프론트 |`172.16.1.0/26`  |
|구역 1 |데이터베이스 |`172.16.1.128/27` |
|구역 2 |중간 |`172.16.2.0/25`  |
|구역 2 |프론트 |`172.16.3.0/26`  |
|구역 2 |데이터베이스 |`172.16.3.128/27` |
|구역 3 |중간 |`172.16.4.0/25`  |
|구역 3 |프론트 |`172.16.5.0/26`  |
|구역 3 |데이터베이스 |`172.16.5.128/27` |

## 기존 인프라 확장 시 고려사항
{: #considerations-for-extending-an-existing-infrastructure}

기존 인프라를 확장하는 VPC를 계획 중인 경우 해당 인프라가 온프레미스 인프라인지, 다른 VPC 또는 다른 클라우드인지에 관계없이 이전 단계를 따를 수 있습니다. 기존 주소 범위를 재사용해서는 안된다는 점에 유의하십시오. 주소 재사용을 방지하면 {{site.data.keyword.cloud_notm}} VPC 기능을 최대한 활용할 수 있습니다.