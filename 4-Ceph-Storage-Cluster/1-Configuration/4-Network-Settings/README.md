# NETWORK CONFIGURATION REFERENCE

네트워크 세팅은 높은 성능의 Ceph Storage Cluster 를 구성하는데 굉장히 중요합니다. Ceph Storage Cluster 는 Ceph Client 를 대신하여 요청 라우팅 또는 디스 패치를 수행하지 않습니다. 대신에, Ceph Client 들은 Ceph OSD 데몬들에게 직접적으로 요청합니다. Ceph OSD 데몬은 Ceph 클라이언트를 대신하여 레플리케이션을 수행하며, 이는 이러한 요소들이 Ceph Storage Cluster 에 네트워크 부하를 가하는 것을 의미합니다.

Quick Start 모니터 IP 주소와 데몬 호스트 이름 만 설정하는 간단한 Ceph 구성 파일을 제공합니다. 클러스터 네트워크를 지정하지 않으면, Ceph 은 단일 public 네트워크를 가정합니다. Ceph은 public 네트워크에서 정상적으로 동작하지만, 대규모 클러스터에서 두번째 클러스터 네트워크를 사용하면 성능이 크게 향상되는 것을 볼 수 있습니다.

Ceph Storage Cluster 를 두개의 네트워크로 실행하기를 권장합니다: public (front-side) 네트워크와 cluster (back-side) 네트워크. 두 네트워크를 지원하기 위해서는, 각각의 Ceph 노드는 하나 이상의 NIC 를 가지고 있어야 합니다. [Hardware Recommandations - Networks](http://docs.ceph.com/docs/master/start/hardware-recommendations#networks) 를 참고하세요.

![ceph-cluster-network](/Images/ceph-cluster-network.png)

두개의 분리된 네트워크로 수행해야 할 몇가지 이유가 있습니다.

1. **Performance**: Ceph OSD 데몬은 Ceph 클라이언트를 위해 데이터 레플리케이션을 담당합니다. Ceph OSD 데몬이 한번 이상 데이터를 레플리케이션 할 때, Ceph OSD 데몬간의 네트워크로 인해 Ceph 클라이언트와 Ceph Storage Cluster 간의 네트워크가 줄어듭니다. 이것은 레이턴스 저하를 일으킬 수 있습니다. 리커버리와 리밸런싱 또한 public 네트워크에서 상당한 레이턴시를 유발할 수 있습니다. Ceph 이 어떻게 데이터를 레플리케이션 하는지에 대한 디테일한 정보는 [Scalability and High Availability](http://docs.ceph.com/docs/master/architecture#scalability-and-high-availability) 를 참고하고, 하트비트 트레픽이 대해서는 [Monitor / OSD Interaction](http://docs.ceph.com/docs/master/rados/configuration/mon-osd-interaction) 을 참고하세요.
2. **Security**: 몇몇 사람들은 이른바 DoS 공격을 시도할 수 있습니다. Ceph OSD 데몬의 트래픽에서 디스럽트가 있으면, placement group 들은 active + clean 상태를 유지할 수 없습니다. 이는 유저들이 데이터를 읽거나 쓸 수 없음을 의미합니다. 이런 공격을 막는 좋은 방법은 인터넷에 연결되지 않은 완전히 별개의 클러스터 네트워크를 유지하는 것입니다. 또한, [Message Signatures](http://docs.ceph.com/docs/master/rados/configuration/auth-config-ref#signatures) 를 사용해 공격을 방지할 수도 있습니다.

## IP TABLES
기본값으로, 데몬은 6789:7300 범위의 포트에 바인드되어 있습니다. IP 테이블을 세팅하기 전에, default iptables 설정을 확인하세요.
```
sudo iptables -L
```
몇몇 리눅스 배포판에는, SSH를 제외한 네트워크 인터페이스로부터의 요청을 거절하는 룰을 포함하고 있을 수도 있습니다. 예를들어,
```
REJECT all -- anywhere anywhere reject-with icmp-host-prohibited
```

public 네트워크와 cluster 네트워크 모두에서 이 룰을 지워야 하며, Ceph 노드의 포트를 커스텀할 준비가 되면 적절한 규칙으로 대체해야 합니다.

### MONITOR IP TABLES
기본적으로, Ceph Monitor 는 6789 포트를 사용합니다. 더불어, Ceph Monitor 는 항상 public 네트워크를 사용합니다. 아래와 같이 룰을 추가할 수 있습니다.

```
sudo iptables -A INPUT -i {iface} -p tcp -s {ip-address}/{netmask} -dport 6789 -j ACCEPT
```

### MDS AND MANAGER IP TABLES
[Ceph Metadata Server](http://docs.ceph.com/docs/master/glossary/#term-ceph-metadata-server) 혹은 [Ceph Manager](http://docs.ceph.com/docs/master/glossary/#term-ceph-manager) 는 6800 포트부터 시작해 처음으로 사용가능한 public 네트워크의 포트를 사용합니다. 이것은 고정적이지 않습니다. 따라서 같은 호스트에 하나 이상의 OSD 혹은 MDS 를 운영하고 있거나, 데몬을 재시작한다면, 데몬은 높은 포트에 바인딩될 것입니다. 그러므로 6800-7300 의 범위 안의 포트를 열어 놔야 합니다. 아래와 같이 룰을 추가할 수 있습니다.
```
sudo iptables -A INPUT -i {iface} -m multiport -p tcp -s {ip-address}/{netmask} -dport 6800:7300 -j ACCEPT
```

### OSD IP TABLES
기본으로, Ceph OSD 데몬은 6800 부터 시작해 첫번째로 가능한 포트에 바인딩됩니다. 역시 고정적이지 않으며, 같은 호스트에 하나 이상의 OSD 나 MDS 를 운영하고 있거나, 데몬을 재시작한다면, 데몬은 더 높은 포트에 바인딩됩니다. 각각의 OSD 데몬은 4개의 포트를 사용합니다:

1. 한개는 클라이언트와 모니터를 받아들임
2. 한개는 다른 OSD 에게 데이터를 전송
3. 두개는 각 인터페이스에서 하트비팅을 하는데 사용

![osd-port](/Images/osd-port.png)

데몬이 사용불가이거나 재시작되면, 데몬은 새로운 포트에 바인딩됩니다. 이 가능성을 처리하려면 전체 6800-7300 포트 범위를 열어야합니다.

만약 분리된 public 네트워크와 cluster 네트워크를 세팅했다면, 두 네트워크를 위한 룰을 추가해야 합니다. 왜냐하면 클라이언트가 public 네트워크를 통해 접속하고, Ceph OSD 데몬들이 cluster 네트워크를 통해 연결되기 때문입니다. 아래의 에제로 룰을 추가할 수 있습니다.

```
sudo iptables -A INPUT -i {iface} -m multiport -p tcp -s {ip-address}/{netmask} -dport 6800:7300 -j ACCEPT
```

> **Tip**: Ceph 메타데이터 서버를 Ceph OSD 데몬과 동일한 Ceph Node 에서 실행하는 경우, public 네트워크 구성 단계를 통합할 수 있습니다.

## CEPH NETWORKS
Ceph 네트워크를 설정하기 위해서, 설정 파일의 `[global]` 섹션에 네트워크 설정을 추가해야 합니다. 5분 Quick Start 는 클라이언트와 서버가 함께 있는 하나의 public 네트워크를 가정합니다 ([Ceph configuration file](http://docs.ceph.com/docs/master/start/quick-ceph-deploy/#create-a-cluster)). Ceph의 기능들은 public 네트워크만 있는 환경에서 잘 동작하지만, 여러 IP 네트워크 및 public 네트워크의 서브넷 마스크를 포함하여 훨씬 더 구체적인 기준을 설정할 수 있습니다. OSD 하트비트, 오브젝트 레플리케이션과 리커버리 트래픽을 위해 cluster 네트워크를 분리할 수도 있습니다. 설정한 IP 주소와 네트워크 클라이언트가 서비스에 엑세스 하는데 사용할 수 있는 공용 IP 주소를 혼동하지 마십시오. 일반적으로 내부 IP 어드레스는 대게 192.168.0.0 또는 10.0.0.0 입니다.

> **Tip**: public 또는 cluster 네트워크에 둘 이상의 IP 주소와 서브넷 마스크를 지정하면 네트워크 내의 서브넷이 서로 라우팅 할 수 있어야합니다. 또한 각 IP 주소 / 서브넷을 IP 테이블에 포함하고 필요한 경우 해당 포트를 열어야합니다.
> **Note**: Ceph 은 서브넷에서 CIDR 노테이션을 사용합니다. (ex, 10.0.0.0/24)

네트워크 섲렁을 완료했다면, 클러스터를 재시작하거나, 각각의 데몬을 재시작 합니다. Ceph 데몬들은 동적으로 바인딩되며, 네트워크 설정을 한번 변경했다면 전체 클러스터를 재시작할 필요가 ㅇ벗습니다.

### PUBLIC NETWORK
public 네트워크를 설정하기 위해, `[global]` 섹션에 다음의 옵션을 추가합니다.
```
[global]
  # ... elided configuration
  public network = {public-network}/{netmask}
```

### CLUSTER NETWORK
cluster 네트워크를 지정했다면, OSD 들은 cluster 네트워크를 통하여 하트비트, 레플리케이션, 리커버리를 라우팅합니다. 이것은 단일 네트워크에 비해 성능 향상을 가져다 줄 것입니다. cluster 네트워크를 구성하려면, `[global]` 섹션에 다음의 옵션을 추가합니다.
```
[global]
  # ... elided configuration
  cluster network = {cluster-network}/{netmask}
```

## CEPH DAEMONS
Ceph 은 모든 데몬에 적용되는 하나의 네트워크 설정 요구사항을 가지고 있습니다: 각각의 데몬에 대해서 `host`를 반드시 지정해 주어야 합니다. 또한 Ceph 설정 파일에 모니터의 IP 주소와 포트를 지정해야 합니다.

> **Important** 몇몇 배포 툴들 (ex, ceph-deploy, Chef) 는 설정 파일을 생성해줍니다. 배포 툴을 사용한다면, 이런 값들을 건드리지 마십시오.
> **Tip**: host 세팅은 IP 주소가 아닌 host 의 짧은 이름입니다. 호스트의 이름을 얻으려면 `hostname -s` 커맨드를 사용합니다.

```
[mon.a]

        host = {hostname}
        mon addr = {ip-address}:6789

[osd.0]
        host = {hostname}
```
데몬을 위해서 host 의 IP 주소를 세팅할 필요는 없습니다. public 네트워크와 cluster 네트워크에 고정 IP 설정이 있다면, Ceph 설정파일은 각각의 데몬에 대해서 host IP 를 명세해 주어야 합니다. 데몬의 고정 IP 주소를 세팅하려면, 다음의 설정이 있어야 합니다.

```
[osd.0]
        public addr = {host-public-ip-address}
        cluster addr = {host-cluster-ip-address}
```

> **2 개의 네트워크 클러스터에서 하나의 NIC OSD**  
> 일반적으로 두 개의 네트워크가있는 클러스터에 단일 NIC가있 는 OSD 호스트를 배포하지 않는 것이 좋습니다. 그러나 Ceph 구성 파일의 [osd.n] 섹션에 public addr 항목을 추가하여 OSD 호스트가 public 네트워크에서 작동하도록 강제 할 수 있습니다. 여기서 n은 하나의 NIC가 있는 OSD 번호입니다. 또한 public 네트워크와 cluster 네트워크는 트래픽을 서로 라우팅 할 수 있어야하며 보안상의 이유로 권장하지 않습니다.





















.
