# COMMON SETTINGS
[Hardware Recommandations](http://docs.ceph.com/docs/master/start/hardware-recommendations/) 섹션은 Ceph Storage Cluster 를 설정함에 있어서 몇몇 가이드라인을 제공합니다. 이것은 하나의 Ceph Node 에서 여러 데몬을 실행시키는 것을 가능하게 합니다. 예를들어, 여러 드라이브를 가진 하나의 노드는, 각각 드라이브에서 하나의 ceph-osd 를 실행할 수 있습니다. 이상적으로, 특정 타입의 프로세스를 위한 노드를 가질 수 있습니다. 예를 들어, 몇몇 노드는 ceph-osd 데몬을 실행하고, 다른 노드는 ceph-mds 데몬을 실행하고, 그리고 다른 노드는 ceph-mon 데몬을 실행할 수 있습니다.


각각의 노드는 `host` 세팅에 따라 이름 식별자를 가지고 있습니다. 모니터들은 IP 혹은 도메인이름과 포트로 구분할 수 있습니다. 기본 설정 파일은 일반적으로 모니터 데몬들의 최소한의 세팅만을 정의합니다.

예를 들어,
```
[global]
mon_initial_members = ceph1
mon_host = 10.0.0.1
```

> **중요**: host 세팅은 노드의 IP 어드레스가 아닌 노드의 짧은 이름입니다. `hostame -s` 커맨드를 입력하여 노드의 이름을 알아낼 수 있습니다. Ceph를 수동으로 배포하지 않는 한 초기 모니터 이외의 다른 설정에는 호스트 설정을 사용하지 마십시오. `chef` 혹은 `ceph-deploy` 같은 배포 툴에서 이 툴들이 클러스터 맵에 적절한 값으로 들어갈 수 있도록 툴을 사용할 때 호스트 특정하면 안됩니다. (?)

### NETWORKS
Ceph 을 사용하기 위한 디테일한 정보를 보려면, [Network Configuration Reference](http://docs.ceph.com/docs/master/rados/configuration/network-config-ref) 를 참고합니다.

### MONITORS
Ceph 의 프로덕션 클러스터는 높은 가용성을 위해 적어도 3개의 Ceph Monitor 데몬으로 배포합니다. 적어도 세개의 모니터가 있어야 `Paxos` 알고리즘이 쿼럼 내의 모니터들 중 가장 최근의 Ceph Cluster Map 을 결정할 수 있습니다.
> Note: 한개의 모니터로 Ceph 을 배포할 수는 있지만, 해당 인스턴스가 망가지면, 데이터 가용성에 인터럽트가 발생할 것입니다.

Ceph 모니터들은 일반적으로 6789 포트를 사용합니다.
```
[mon.a]
host = hostName
mon addr = 150.140.130.120:6789
```
기본값으로, 다음 경로에 모니터의 데이터들을 저장합니다.
```
/var/lib/ceph/mon/$cluster-$id
```
여러분 혹은 배포 툴 (ex, ceph-deploy) 가 적절한 디렉토리를 만들어 주어야 합니다. 적절한 디렉토리는 다음과 같습니다.
```
/var/lib/ceph/mon/ceph-a
```
추가적인 설명으로는, [Monitor Config Reference](http://docs.ceph.com/docs/master/rados/configuration/mon-config-ref) 를 참고합니다.

### AUTHENTICATION
Bobtail (v 0.56) 이하 버전에는, Ceph Configuration file 의 `[global]` 섹션에 직접적으로 써주어야 합니다.
```
auth cluster required = cephx
auth service required = cephx
auth client required = cephx
```

더불어, message signing 을 enable 해주어야 합니다. 자세한 내용은 [Cephx Config Reference](http://docs.ceph.com/docs/master/rados/configuration/auth-config-ref)를 참고합니다.

> **중요**: 업그레이드를 진행할 때, 수행 전 authentication 을 끄는 것을 추천합니다. 업그레이드 완료 후에 다시 enable 해주세요

### OSDS
Ceph 의 프로덕션 클러스터는 일반적으로 하나의 노드가 하나의 스토리지 드라이브에 filestore를 실행하는 하나의 OSD 데몬을 갖는 Ceph OSD 데몬을 배포합니다. 일반적인 배포는 journal size 를 상세합니다. 예를 들어,
```
[osd]
osd journal size = 10000

[osd.0]
host = {hostname} # manual deployment only
```
기본값으로, Ceph 은 다음 경로에 OSD 의 데이터를 저장한다고 가정합니다.
```
/var/lib/ceph/osd/$cluster-$id
```
마찬가지로, 적절한 경로에 디렉토리를 만들어 줍니다.
```
/var/lib/ceph/osd/ceph-0
```
osd 데이터 경로는 이상적으로는 운영 체제 및 데몬을 저장하고 실행하는 하드 디스크와 별개의 개인 하드 디스크가 있는 마운트 지점으로 연결됩니다. 만약 OSD가 OS 디스크가 아닌 다른 디스크를 위한 것이라면 Ceph와 함께 사용할 수 있도록 준비하고 방금 만든 디렉토리에 마운트하십시오 :
```
ssh {new-osd-host}
sudo mkfs -t {fstype} /dev/{disk}
sudo mount -o user_xattr /dev/{hdd} /var/lib/ceph/osd/ceph-{osd-number}
```

`mkfs` 명령을 사용할 때, `xfs` 파일 시스템을 사용하는 것을 권장합니다. (`btrfs` 혹은 `ext4` 는 더이상 테스트되지 않으며, 권장하지 않습니다.)

디테일한 설정들은 [OSD Config Reference](http://docs.ceph.com/docs/master/rados/configuration/osd-config-ref) 를 참고하세요.

### HEARTBEATS
런타임 시에, Ceph OSD 데몬들은 다른 OSD 데몬들을 체크하고, 정보들을 Ceph Monitor 에게 전달합니다. 다른 세팅을 제공할 필요는 없습니다. 그러나, network latency 문제를 겪고 있다면, 세팅을 수정하고 싶어할 수 있습니다. 이에 관한 정보들은, [Configuring Monitor/OSD Interaction](http://docs.ceph.com/docs/master/rados/configuration/mon-osd-interaction) 를 참고하세요.

### LOGS / DEBUGGING
가끔 로그를 수정하거나, Ceph 의 디버깅 모드를 사용해야 할 때가 있을 수 있습니다. [Debugging and Logging](http://docs.ceph.com/docs/master/rados/troubleshooting/log-and-debug) 을 참고하세요.

### EXAMPLE CEPH.CONF
```
[global]
fsid = {cluster-id}
mon initial members = {hostname}[, {hostname}]
mon host = {ip-address}[, {ip-address}]

#All clusters have a front-side public network.
#If you have two NICs, you can configure a back side cluster
#network for OSD object replication, heart beats, backfilling,
#recovery, etc.
public network = {network}[, {network}]
#cluster network = {network}[, {network}]

#Clusters require authentication by default.
auth cluster required = cephx
auth service required = cephx
auth client required = cephx

#Choose reasonable numbers for your journals, number of replicas
#and placement groups.
osd journal size = {n}
osd pool default size = {n}  # Write an object n times.
osd pool default min size = {n} # Allow writing n copy in a degraded state.
osd pool default pg num = {n}
osd pool default pgp num = {n}

#Choose a reasonable crush leaf type.
#0 for a 1-node cluster.
#1 for a multi node cluster in a single rack
#2 for a multi node, multi chassis cluster with multiple hosts in a chassis
#3 for a multi node cluster with hosts across racks, etc.
osd crush chooseleaf type = {n}
```

### RUNNING MULTIPLE CLUSTERS
동일한 하드웨어에서 다중의 Ceph Cluster 를 운영할 수 있습니다. 이는 같은 클러스터에서 다른 pool 을 이용하는 것 보다 높은 수준의 고립을 제공합니다. 분리된 클러스터는 분리된 monitor, OSD 와 metadata 서버 프로세스를 가집니다. 기본값으로 Ceph 을 실행할 때, ceph cluster 의 기본값 이름은 `ceph` 입니다. 이는 `/etc/ceph` 기본 디렉토리 안에 `ceph.conf` 라는 Ceph 설정 파일을 저장했다는 뜻입니다.

추가적인 정보는 [ceph-deploy new](http://docs.ceph.com/docs/master/rados/deployment/ceph-deploy-new/#naming-a-cluster) 를 참고하세요.

다중 클러스터를 실행시킬 때, 클러스터 이름을 지정하고, 그것으로 설정 파일을 저장해야 합니다. 예를 들어, 클러스터 이름이 `openstack` 이라면, 기본 디렉토리인 `/etc/ceph` 안에 `openstack.conf` 이름으로 저장해야 합니다.

> **중요**: 클러스터 이름은 영어 소문자 혹은 숫자로만 구성되어야 합니다.

분리된 클러스터는 클러스터간 공유하지 않는 분리된 데이터 디스크와 jounal 을 가집니다. [Metavariables](http://docs.ceph.com/docs/master/rados/configuration/ceph-conf#Metavariables) 에서 언급했듯이, `$cluster` metavariable 은 클러스터 이름을 지칭합니다. (ex, 위에 말했던 `openstack` 같은) 많은 세팅이 `$cluster` 를 사용합니다.

- keyring
- admin socket
- log file
- pid file
- mon data
- mon cluster log file
- osd data
- osd journal
- mds data
- rgw data

[General Settings](http://docs.ceph.com/docs/master/rados/configuration/general-config-ref), [OSD Settings](http://docs.ceph.com/docs/master/rados/configuration/osd-config-ref), [Monitor Settings](http://docs.ceph.com/docs/master/rados/configuration/mon-config-ref), [MDS Settings](http://docs.ceph.com/docs/master/cephfs/mds-config-ref), [RGW Settings](http://docs.ceph.com/docs/master/radosgw/config-ref/), 그리고 [Log Settings](http://docs.ceph.com/docs/master/rados/troubleshooting/log-and-debug) 를 참고하세요.

기본값인 디렉토리나 파일을 만들 때, 경로의 적절한 장소에 클러스터 이름을 써야 합니다. 예를들어,
```
sudo mkdir /var/lib/ceph/osd/openstack-0
sudo mkdir /var/lib/ceph/mon/opentsack-a
```

> **중요**: 같은 호스트에서 모니터를 실행할 때, 다른 포트를 사용해야 합니다. 기본적으로 모니터가 사용하는 포트는 6789이며, 다른 클러스터에서는 다른 포트를 사용하세요.

기본 ceph 클러스터 외에 다른 클러스터를 invoke 시키려면, `-c {filname}.conf` 옵션을 사용하세요. 예를 들어,
```
ceph -c {cluster-name}.conf health
ceph -c openstack.conf health
```












.
