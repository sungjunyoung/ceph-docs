# MONITOR CONFIG REFERENCE
Ceph 모니터를 어떻게 세팅하는지 이해하는 것은 신뢰할 수 있는 Ceph Storage 클러스터를 구축하는데 굉장히 중요한 부분입니다. **모든 Ceph Storage 클러스터는 적어도 하나의 모니터를 가지고 있습니다.** 모니터 설정은 대게 일정하게 유지되지만, 클러스터에서 모니터를 대체하거나, 추가하거나, 삭제할 수 있습니다. [Adding/Removing a Monitor](http://docs.ceph.com/docs/master/rados/operations/add-or-rm-mons) 와 [Add/Remove a Monitor (ceph-deploy)](http://docs.ceph.com/docs/master/rados/deployment/ceph-deploy-mon) 를 참고하세요.

## BACKGROUND
Ceph 모니터는 [cluster map](http://docs.ceph.com/docs/master/glossary/#term-cluster-map) 의 "master copy" 를 유지합니다. 이것은 [Ceph Client](http://docs.ceph.com/docs/master/glossary/#term-ceph-client)가 하나의 Ceph 모니터에 접속하고 cluster map 을 얻어옴으로서 모든 Ceph MON, OSD, MDS 의 위치를 알 수 있다는 의미입니다. Ceph 클라이언트가 Ceph OSD 혹은 MDS 로 부터 데이터를 읽거나 쓰기 전에, 클라이언트들은 Ceph 모니터에 먼저 접속해야 합니다. 최신의 cluster map 과 CRUSH 알고리즘으로, Ceph 클라이언트는 어떠한 오브젝트의 위치도 계산할 수 있습니다. 오브젝트의 위치를 계산할 수 있다는 것은, 클라이언트가 직접적으로 OSD 데몬과 통신할 수 있다는 것을 의미하며, 이것은 Ceph 의 높은 확장성과 성능의 중요한 점입니다. 추가적인 정보는 [Scalability and High Availability](http://docs.ceph.com/docs/master/architecture#scalability-and-high-availability)를 참고하세요.

Ceph 모니터의 가장 중요한 역할은 cluster map의 "master copy" 를 유지하는 것입니다. 또한 Ceph 모니터는 인증과 로깅 서비스를 제공합니다. Ceph 모니터는 단일 Paxos 인스턴스에 모든 모니터 서비스의 변경점을 쓰고, Paxos 는 일관성을 위해 이것들을 key/value 저장소에 씁니다. Ceph 모니터는 sync 오퍼레이션 동안에 가장 최신의 cluster map 을 쿼리할 수 있습니다. Ceph 모니터는 key/value 저장소의 스냅 샷과 iterator(leveldb 를 사용)들을 사용하여 저장소 전반의 동기화를 수행합니다.

![paxos-key-value](/Images/paxos-key-value.png)

0.58 이후 버전에서 사용되지 않음

Ceph 0.58 및 이전 버전에서는 Ceph 모니터들이 각 서비스에 대해 Paxos 인스턴스를 사용하고 맵을 파일로 저장합니다.

### CLUSTER MAPS
클러스터 맵 (cluster map)은 모니터 맵, OSD 맵, pg 맵과 메타데이터 서버 맵을 포함하는 집합체입니다. 클러스터 맵은 여러 중요한 것들을 추적합니다: 클러스터에 있는 프로세스가 실행 중인지, pg 그룹이 활성화 상태인지 비활성화 상태인지, 클러스터의 현재 상태, 총량 및 사용량 등등

클러스터의 상태에 중요한 변화가 있을 경우, (ex, OSD 데몬이 다운되거나,  pg 그룹이 사용불가 상태에 빠지거나) 클러스터 맵은 최신 클러스터 상태를 반영하기 위해 업데이트를 수행합니다. 게다가, Ceph 모니터는 이전의 클러스터 상태들의 역사를 유지합니다. 이러한 각각의 버전을 "epoch" 라고 부릅니다.

Ceph Storage 클러스터를 운영할 때, 이런한 상태를 유지하는 것은 시스템 관리자에게 굉장히 중요한 역할입니다. 추가적인 정보는 [Monitoring a Cluster](http://docs.ceph.com/docs/master/rados/operations/monitoring)와 [Monitoring OSDs and PGs](http://docs.ceph.com/docs/master/rados/operations/monitoring-osd-pg) 를 참고하세요.

### MONITORING QUORUM
Configuring ceph 섹션은 테스트 클러스터에서 하나의 모니터를 위한 [Ceph configuration file](http://docs.ceph.com/docs/master/rados/configuration/ceph-conf/#monitors) 을 제공합니다. 단일 모니터는 단일 모니터에서 문제 없이 실행 될 것입니다. 그러나, **단일 모니터는 SPOF 를 가지고 있습니다.** 프로덕션 레벨에서 클러스터의 높은 가용성을 확신하기 위해서는, SPOF 가 전체 클러스터의 다운을 야기시키지 않도록 여러개의 모니터를 사용해야 합니다.

높은 가용성을 위해 다수의 Ceph 모니터를 실행했을 경우, Ceph 모니터는  [Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science) 를 사용하여 마스터 클러스터 맵에 대한 합의를 도출합니다. 합의가 이루어지면 대다수의 모니터가 실행되어 클러스터 맵에 대한 합의를 위한 정족수를 설정해야합니다 (예 : 3 개 중 2 개, 5 개 중 3 개, 6 개 중 4 개 등).

- mon force quorum join
  - Description: 모니터가 이전에 맵에서 삭제되었더라도 쿼럼에 참여하도록 강제합니다.
  - Type: Boolean
  - Default: False

### CONSISTENCY
Ceph 설정 파일에 모니터 세팅을 추가하는 경우에, Ceph 모니터의 아키텍쳐를 조금 알아야 합니다. Ceph 는 클러스터에서 다른 Ceph 모니터를 발견할 때, Ceph 모니터에 엄격한 일관성을 부과합니다. 모니터를 발견하기 위해서 Ceph 클라이언트와 데몬들은 Ceph 설정 파일을 사용하는 데 반해, 모니터는 설정 파일이 아닌 다른 모니터들을 monmap 을 이용해 찾습니다.

Ceph 모니터는 Ceph Storage 클러스터 안에서 다른 Ceph 모니터를 찾기 위해서 항상 monmap 의 복사본을 유지합니다. Ceph 설정 파일 대신 monmap 을 사용하면, 클러스터를 깨뜨릴 수 있는 에러를 회피합니다 (ex, 모니터의 주소, 포트를 설정 파일에 적을 때의 오타 등). 모니터가 monmap 을 사용하고, 클라이언트와 데몬들과 monmap 을 공유하기 때문에, **monmap은 강력한 일관성 보증을 모니터에게 제공합니다.**

강력한 일관성은 monmap 을 업데이트 할 때에도 적용됩니다. Ceph 모니터에 어떠한 업데이트가 있을 때, monmap은 분산 합의 알고리즘인 [Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science))를 사용해 변경됩니다. Ceph 모니터들은 모니터를 생성/삭제하거나, 쿼럼에 있는 모니터가 같은 버전의 monmap 을 가지고 있는지 확인하는 등의 monmap 의 변화에 각각 동의해야 합니다. monmap에 대한 업데이트는 Ceph 모니터들이 최신 동의된 버전과 이전 버전들을 갖도록 증분됩니다.  이렇게 역사를 유지하는 것은 이전 버전의 monmap을 가진 Ceph 모니터가 Ceph Storage 클러스터의 현재 상태를 따라 잡을 수 있게 합니다.

만약 모니터들이 monmap 대신에 Ceph 설정 파일을 통해 다른 모니터를 찾는다면, Ceph 설정 파일은 자동으로 업데이트 되거나, 분산되지 않기 때문에 추가적인 위험요소가 있을 수 있습니다. 이렇게 Ceph 모니터는 실수로 이전 Ceph 구성 파일을 사용하거나, Ceph 모니터를 인식하지 못하거나, 쿼럼에서 벗어나거나, Paxos가 시스템의 현재 상태를 정확하게 판단 할 수 없는 상황을 만들 수 있습니다.

### BOOTSTRAPPING MONITORS
대부분의 설정과 배포 상황에서, Ceph 배포 툴들은 monmap 을 생성하여 Ceph 모니터를 부트스트래핑 하도록 도와줍니다. (ex, ceph-deploy 등) 이런 Ceph 모니터들은 몇가지의 세팅이 필요합니다.

- **Filesystem ID**: `fsid`는 오브젝트 스토어의 식별자입니다. 같은 하드웨어에서 여러 클러스터를 실행할 수 있기 때문에, 모니터를 부트스트래핑 할 때 오브젝트 스토어의 식별자를 명시해야 합니다. 배포 툴들은 일반적으로 이런 작업을 모두 해 줄 것입니다. (ex, ceph-deploy 는 uuidgen 같은 툴을 호출합니다.), 그러나 수동으로 `fsid` 를 지정해 줘야 할 수도 있습니다.
- **Monitor ID**: 모니터 ID 는 클러스터 내에 각각의 모니터에게 할당된 식별자입니다. 영어로 이루어져 있고, 컨벤션으로 알파벳 증분 규칙 (ex, a, b, c..) 을 따릅니다. 이것은 배포 툴 혹은 ceph 커맨드라인을 통해 Ceph 설정 파일에 세팅될 수 있습니다. (ex, [mon.a], [mon.b])
- **Keys**: 모니터는 비밀 키를 가지고 있습니다. `ceph-deploy` 와 같은 배포 툴들은 이런 것을 알아서 해주지만, 수동으로도 수행해 주어야 합니다. [Monitor Keyrings](http://docs.ceph.com/docs/master/dev/mon-bootstrap#secret-keys) 를 참고하세요.

부트스트래핑에서 추가적인 정보는, [Bootstrapping a Monitor](http://docs.ceph.com/docs/master/dev/mon-bootstrap) 를 참고하세요

## CONFIGURING MONITORS
전체 클러스터에 설정을 적용하기 위해서는, `[global]` 섹션 아래의 설정으로 들어갑니다. 클러스터의 모든 모니터에 설정을 적용하려면, `[mon]` 섹션 아래에 설정합니다. 특정 모니터를 설정하려면, 모니터 인스턴스 (ex, `[mon.a]`) 섹션 아래에 설정합니다. 컨벤션으로, 모니터 인스턴스 이름은 alpha notation 을 사용합니다.
```
[global]

[mon]

[mon.a]

[mon.b]

[mon.c]
```

### MINIMUM CONFIGURATION
Ceph 설정 파일을 통한 Ceph 모니터의 최소한의 설정은 hostname 과 각 모니터의 주소를 포함합니다. 이러한 것들을 [mon] 이나 특정 모니터 아래에 설정할 수 있습니다.
```
[mon]
        mon host = hostname1,hostname2,hostname3
        mon addr = 10.0.0.10:6789,10.0.0.11:6789,10.0.0.12:6789
```
```
[mon.a]
        host = hostname1
        mon addr = 10.0.0.10:6789
```
추가 정보는 [Network Configuration Reference](http://docs.ceph.com/docs/master/rados/configuration/network-config-ref)를 참고하세요

> **Note**: 이런 최소한의 모니터 세팅은 배포 툴들이 `fsid` 와 `mon` 을 생성 했다고 가정한 설정입니다.

일단 Ceph 클러스터를 배포했다면, 모니터의 IP 를 **절대** 변경하지 마십시오. 그러나, 혹여 IP 주소를 변경하기로 결정했다면, 특정 절차를 따라야 합니다. [Changing a Monitor's IP Address](http://docs.ceph.com/docs/master/rados/operations/add-or-rm-mons#changing-a-monitor-s-ip-address) 를 참고하세요.

모니터들은 DNS SRV record 를 사용하여 클라이언트들에 의해 찾아질 수도 있습니다. [Monitor lookup through DNS] 를 참고하세요

### CLUSTER ID
각각의 Ceph Storage 클러스터는 식별자를 가지고 있습니다.(fsid) 이것이 결정되어지면, 설정 파일 아래의 [global] 섹션 아래에 나타날 것입니다. 보통 배포 툴들은 `fsid`를 생성하고, monmap에 저장합니다. 그래서 설정 파일에 나타나지 않을 수도 있습니다. `fsid` 은 같은 하드웨어에 여러 클러스터를 위한 데몬을 실행시킬 수 있도록 만들어줍니다.

- fsid
  - Description: 클러스터 ID. 클러스터당 하나
  - Type: UUID
  - Required: Yes.
  - Default: N/A, 지정하지 않으면 배포 툴에 의해 생성됩니다.

> **Note**: 배포 툴을 사용한다면, 이 값을 세팅하지 마십시오

### INITIAL MEMBERS
높은 가용성을 위해 Ceph Storage 클러스터에는 적어도 세개 이상의 Ceph 모니터를 실행하는 것을 추천합니다. 여러 모니터를 실행한다면, 쿼럼을 형성하기 위해 클러스터의 멤버들이 되어야 하는 초기 모니터들을 지정해야 합니다. 이것은 클러스터가 on-line 상태로 가는 시간을 줄여줄 것입니다.
```
[mon]
      mon initial members = a,b,c
```
- mon initial members
  - Description: 클러스터의 시작 시점에서의 초기 모니터들의 ID. 지정되어 있으면, 초기 쿼럼을 위해서 홀수 개가 필요합니다.
  - Type: String
  - Default: None

> **Note**: 클러스터에 있는 대부분의 모니터는 쿼럼을 형성하기 위해 서로 연결할 수 있어야합니다. 이 세팅으로 쿼럼을 형성할 모니터의 초기 숫자를 줄일 수 있습니다.

### DATA
Ceph 은 모니터의 데이터들이 어디로 저장될 지 기본값을 가지고 있습니다. 프로덕션 Ceph Storage 클러스터에서 선택적인 퍼포먼스를 위해서, OSD 데몬과는 다른 호스트와 드라이브에 모니터를 실행하기를 권장합니다. leveldb 가 데이터를 쓰는데 mmap() 을 사용하기 때문에, 모니터들은 메모리에서 디스크로 데이터들을 매우 자주 플러시하고, OSD 데몬이 같은 위치에 있다면 OSD 의 작업에 영향을 줄 수 있습니다.

0.58 혹은 이전 버전에서는, Ceph 모니터들을 파일에 그들의 데이터를 저장했습니다. 이런 접근법은 `ls`나 `cat`으로 유저들에게 모니터 데이터에 접근할 수 있도록 했습니다. 그러나, 이것은 높은 일관성을 제공하진 않았습니다.

0.59 이상의 버전에서는, 모니터들은 key/value 쌍으로 데이터를 저장합니다. Ceph 모니터들은 [ACID](https://en.wikipedia.org/wiki/ACID) 트랜젝션이 필요합니다. 데이터 저장소를 사용하면 Ceph 모니터가 Paxos를 통해 손상된 버전을 실행하는 것을 방지 할 수 있으며 다른 장점은 단일 배치에서 여러 수정 작업을 수행 할 수 있다는 것입니다.

일반적으로, 기본 데이터 저장 위치를 바꾸지 않는 것을 추천합니다. 만약 기본 위치를 수정한다면, 모든 모니터들에 통일성 있게 [mon] 섹션에 세팅하십시오.

- mon data
  - Description: 모니터의 데이터 저장 위치
  - Type:	String
  - Default:	/var/lib/ceph/mon/$cluster-$id
- mon data size warn
  - Description:	모니터의 데이터 저장소가 15GB를 넘으면 클러스터 로그에서 HEALTH_WARN 을 발행합니다.
  - Type:	Integer
  - Default:	15*1024*1024*1024*
- mon data avail warn
  - Description:	모니터의 데이터 저장소에서 이용 가능한 디스크 용량이 이 퍼센트보다 낮거나 같으면 HEALTH_WARN 을 발행합니다.
  - Type:	Integer
  - Default:	30
- mon data avail crit
  - Description:	모니터의 데이터 저장소에서 이용 가능한 디스크 용량이 이 퍼센트보다 낮거나 같으면 HEALTH_ERR 을 발행합니다.
  - Type:	Integer
  - Default:	5
- mon warn on cache pools without hit sets
  - Description: Issue a HEALTH_WARN in cluster log if a cache pool does not have the hitset type set set. See hit set type for more details. 캐시 풀에 hitset 유형 세트가 설정되어 있지 않으면 클러스터 로그에서 HEALTH_WARN을 발행합니다. 자세한 내용은 [hit set type](http://docs.ceph.com/docs/master/rados/configuration/operations/pools#hit-set-type) 을 참조하십시오.
  - Type:	Boolean
  - Default:	True
- mon warn on crush straw calc version zero
  - Description: CRUSH 의 `straw_calc_version` 이 0 이면 HEALTH_WARN 을 발행합니다. [CRUSH map tunables](http://docs.ceph.com/docs/master/rados/configuration/operations/crush-map#tunables) 를 참고하세요
  - Type:	Boolean
  - Default:	True
- mon warn on legacy crush tunables
  - Description: CRUSH 의 튜너블이 너무 오래됬을 때(`mon_min_crush_required_version` 보다 오래됬을 때) HEALTH_WARN 을 발행합니다.
  - Type:	Boolean
  - Default:	True
- mon crush min required version
  - Description: The minimum tunable profile version required by the cluster. See CRUSH map tunables for details. 클러스터에 필요한 최소한의 튜너블 프로필 버전. [CRUSH map tunable](http://docs.ceph.com/docs/master/rados/configuration/operations/crush-map#tunables) 을 참고하세요
  - Type:	String
  - Default:	firefly
- mon warn on osd down out interval zero
  - Description: mon osd down out interval 이 0 일 때 HEALTH_WARN 을 발행합니다. 해당 옵션을 0 으로 지정하는 것은 `noout` 플래그와 비스샇게 동작합니다. noout 플래그가 설정되지 않은 채로 클러스터에서 어떤 문제가 발생하는지 알아내는 것은 어렵지만, 이 경우 동일한 경고문이 표시됩니다.
  - Type:	Boolean
  - Default:	True
- mon cache target full warn ratio
  - Description: 풀의 `cache_target_full`과 `target_max_object` 사이에 위치하여 경고를 시작합니다.
  - Type:	Float
  - Default:	0.66
- mon health data update interval
  - Description: (초 단위로) 얼마나 자주 쿼럼이 있는 모니턷르이 health 상태를 체크할 것인지 설정합니다. (음수는 disabled 됩니다.)
  - Type:	Float
  - Default:	60
- mon health to clog
  - Description: 클러스터 로그에 주기적으로 health 요약을 보내는 것을 활성화시킵니다.
  - Type:	Boolean
  - Default:	True
- mon health to clog tick interval
  - Description: 얼마나 자주 (초 단위로) 모니터가 클러스터 로그에 health 요약을 보낼 것인지 설정합니다. (음수는 disabled 됩니다.) 만약 현재 health 요약이 비어있거나 이전 시간과 같으면, 모니터는 클러스터 로그에 보내지 않습니다.
  - Type:	Integer
  - Default:	3600
- mon health to clog interval
  - Description:	얼마나 자주 (초 단위로) 모니터가 클러스터 로그에 health 요약을 보낼 것인지 설정합니다. (음수는 disabled 됩니다.) 바뀌든, 없든 항상 모니터는 요약을 클러스터 로그에 보내게 됩니다.
  - Type:	Integer
  - Default:	60

### STORAGE CAPACITY
Ceph Storage 클러스터가 용량이 가득 차게 되면, (ex, `mon osd full ratio`) Ceph 은 데이터 손실을 막기 위한 안전 조치로 OSD 로 부터 데이터를 읽거나 쓰지 못하게 막습니다. 그러므로, 프로덕션의 Ceph Storage 클러스터에 full ratio 와 근접하게 놔두는 것은 높은 가용성을 희생하기 때문에 좋지 않습니다. full ratio 의 기본값은 .95 (96%) 이며, 이것은 적은 OSD 의 테스트 클러스터를 위한 매우 극적인 세팅입니다.

> **Tip**: 클러스터를 모니털이할 때, nearfull 비율과 관련된 warning 에 주의하십시오. 이는 몇개의 OSD 실패가 일시적인 서비스 붕괴로 이어질 수 있다는 것을 의미합니다. 가용성을 높이기 위해 더 많은 OSD 를 추가하는 것을 고려하십시오.

테스트 클러스터를 위한 공통적인 시나리오는, 클러스터 리밸런싱을 위해 시스템 관리자가 클러스터에서 OSD 데몬을 없애는 것입니다. 이렇게 OSD 데몬을 없애는 행위는, 결국 클러스터가 full ratio 로 이르게 하고, 사용 불가능한 상태로 만들게 됩니다. 테스트 클러스터에서도 용량을 잘 계획하는 것을 추천합니다. 이는 높은 가용성을 위해서 필요한 여유 용량을 측정 가능하게 합니다. 이상적으로, OSD 데몬을 즉시 교체하지 않고도 클러스터가 active + clean 상태로 복구할 수 있는 OSD 데몬을 계획하는 것이 이상적입니다. active + degraded 상태로 클러스터를 실행할 수는 있지만 정상적인 작동 조건에는 이상적이지 않습니다.

다음 다이어그램은 호스트 당 하나의 Ceph OSD 데몬이 있는 33 개의 Ceph 노드를 포함하는 단순한 Ceph Storage 클러스터를 보여주는데, 각 Ceph OSD 데몬은 3TB 드라이브에서 I/O 작업을 하고 있습니다. 그래서 이 전형적인 클러스터는 99TB 의 최대 가용량을 가집니다. `mon osd full ratio` 가 0.95 로 세팅되어있을 때, 만약 클러스터의 나머지 용량이 5TB 로 떨어지면 클라이언트는 데이터를 읽고 쓸 수 없게 됩니다. 그래서 이 클러스터의 가용량은 99TB 가 아닌 95TB 가 됩니다.

![cluster-capacity](/Images/cluster-capacity.png)

하나 혹은 두개의 OSD 가 실패하는 것은 클러스터에서 일반적인 시나리오 입니다. 덜 빈번하지만 합리적인 시나리오는 랙의 라우터 또는 전원 공급 장치에 장애가 발생하여 동시에 여러 개의 OSD가 다운되는 경우입니다. (ex, OSD 7-12) 이러한 시나리오에서, 이러한 시나리오에서는 OSD를 추가 한 호스트 몇 개를 짧은 순서로 추가하는 경우에도 작동 상태를 유지하고 active + clean 상태를 유지할 수있는 클러스터를 위해 노력해야합니다. 용량 사용률이 너무 높으면 데이터가 손실 될 수는 없지만 클러스터의 용량 활용도가 전체 비율을 초과하면 장애 도메인 내에서 장애를 해결하면서도 데이터 가용성을 희생해야만 합니다.(?) 이러한 이유로, 대략적인 데이터 용량 계획을 추천합니다.

클러스터를 위해 두가지를 생각하십시오.ㄴ
1. OSD 의 수
2. 클러스터의 전체 가용량

만약 클러스터 내의 OSd 숫자로 클러스터 전체 가용량을 나눈다면, 클러스터 내의 평균 OSD 용량을 확인할 수 있습니다. 여기서 운영 중에 다운될 수도 있는 OSD 의 숫자를 곱해보십시오. (비교적 적은 양) 마지막으로 클러스터의 용량을 전체 비율로 곱하면 최대 가용량에 도달합니다. 합리적인 전체 비율로 도달하지 않을 것으로 예상되는 OSD 에서 데이터 양을 뺍니다. 보다 많은 수의 OSD 실패 (ex, OSD 랙)로 앞의 프로세스를 반복하여 전체 비율로 적당한 수에 도달하십시오. (?)...

```
[global]

        mon osd full ratio = .80
        mon osd backfillfull ratio = .75
        mon osd nearfull ratio = .70
```

- mon osd full ratio
  - Description: OSD가 가득 찬 것으로 간주되기 전에 사용 된 디스크 공간의 비율
  - Type: Float
  - Default: .95
- mon osd backfillfull ratio
  - Description: OSD가 가득 차서 백업 할 수 없게 되기까지 사용 된 디스크 공간의 비율
  - Type: Float
  - Default: .90
- mon osd nearfull ratio
  - Description: OSD 가 nearfull 상태로 들어갔다고 간주되기 전 사용된 디스크 공간의 비율
  - Type: Float
  - Default: .85

> **Tip**: 몇개의 OSD 가 nearfull 상태이지만 다른 OSD 들은 용량이 남는다면, nearfull OSD 에 대한 CRUSH weight 에서 문제가 발생할 수 도 있습니다.

### HEARTBEAT
Ceph 모니터는 OSD 로 부터 리포트를 요구함으로서 클러스터의 정보에 대해 알고 있습니다. Ceph 은 합리적인 모니터와 OSD 간의 상호작용을 위한 기본 세팅을 가지고 있습니다. 하지만, 필요한 경우 이 세팅을 수정해야 합니다. [Monitor/OSD Interaction](http://docs.ceph.com/docs/master/rados/configuration/mon-osd-interaction) 을 참고하세요.

### MONITOR STORE SYNCHRONIZATION
다수의 모니터로 프로덕슨 클러스터를 실행했을 때 (추천), 각각의 모니터는 이웃의 모니터가 클러스터 맵의 최신 버전을 가지고 있는지 체크합니다. 주기적으로 클러스터의 모니터 하나가 다른 모니터 뒤에 위치하여 쿼럼을 벗어나 클러스터에 대한 최신 정보를 검색하고 쿼럼에 다시 가입해야합니다. 이런 동기화의 목적으로 인해, 모니터는 다음의 세가지 롤을 가지고 있음을 가정합니다.

1. Leader: 리더는 클러스터 맵의 최신 Paxos 버전을 구현한 최초의 모니터입니다.
2. Providers: 프로바이더는 클러스터 맵의 최신 버전을 가지고 있지만, 최초로 구현하지는 않은 모니터입니다.
3. Requester: 리퀘스터는 리더의 뒤에서 떨어졌으면 쿼럼에 다시 참여하기 전에 클러스터에 대한 최신 정보를 검색하기 위해 동기화 해야하는 모니터입니다.

이러한 역할을 통해 리더는 동기화 의무를 공급자에게 위임 할 수 있으므로 동기화 요청이 리더의 성능을 과부하하는 것을 방지합니다. 아래의 다이어그램에서, 리퀘스터는 다른 모니터들에 비해 뒤떨어진 걸 깨닫습니다. 리퀘스터는 리더에게 동기화를 요청하고, 리더는 리퀘스터에게 프로바이더와 함께 동기화하도록 요청합니다.

![monitor-roles](/Images/monitor-roles.png)

동기화는 새로운 모니터가 클러스터에 참여했을 때 항상 일어납니다. 런타임 작업 중에 모니터는 각기 다른 시간에 클러스터 맵에 대한 업데이트를 수신 할 수 있습니다. 이는 리더와 프로바이더 롤들이 하나의 모니터로부터 다른 모니터로 마이그래이션 함을 의미합니다. 동기화 과정 중에 일어난다면, 프로바이더는 리퀘스터와 동기화를 중지할 수 있습니다.

동기화가 완료되면 Ceph는 클러스터 전체를 트리밍해야합니다. 트리밍을 수행하려면 placement group 들이 active + clean 해야합니다.
