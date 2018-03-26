# CEPHX CONFIG REFERENCE

`cephx` 프로토콜은 기본적으로 활성화 되어 있습니다. 암호화 인증은 적은 비용으로 수행할 수 있어야 합니다. 클라이언트와 서버 호스트를 연결하는 이 네트워크 환경은 매우 안전하며 인증을 할 여유가 없는 경우 해제 할 수 있습니다. **그러나 이것은 추천하지 않습니다.**

> **Note**: 만약 인증을 비활성화 시키면, 클라이언트와 서버 간의 메시지에 보안 위협을 받을 수 있으며, 이것은 재앙으로 이어질 수 있습니다.

유저 생성은, [User Management](http://docs.ceph.com/docs/master/rados/operations/user-management) 를 참고하세요. Cephx 의 구조에 대한 디테일한 정보로는, [Architecture - High Availability Authentication](http://docs.ceph.com/docs/master/architecture#high-availability-authentication) 를 참고하십시오.

## DEPOLYMENT SCENARIOS
Cephx 를 어떻게 구성하느냐에 따라 두가지의 Ceph cluster 배포 시나리오가 존재합니다. 첫번째로, Ceph 유저는 클러스터를 만들기 위해  `ceph-deploy` 를 사용합니다(가장 쉽습니다.). 다른 배포 툴을 사용하는 클러스터는 (ex, Chef, Juju, Puppet 등등), 수동 절차를 거치거나, 배포 툴을 설정해 주어야 합니다.

### CEPH-DEPLOY
`ceph-deploy` 를 통해서 클러스터를 배포했다면, 수동으로 모니터를 부트스트래핑 하거나, `client.admin` 유저나 키링을 만들 필요가 없습니다. [Storage Cluster Quick Start](http://docs.ceph.com/docs/master/start/quick-ceph-deploy/) 는 `ceph-deploy`를 통해서 이러한 과정들을 자동화해 줍니다.

만약 `ceph-deploy new {initial-monitors(s)}` 를 실행한다면, Ceph 은 모니터의 키링을 만들어 주고, 아래와 같은 인증 설정이 포함된 Ceph 설정 파일을 만들어 줄 것입니다.

```
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
```

`ceph-deploy mon create-initial` 명령을 실행하면, Ceph 은 초기 모니터들을 부트스트래핑하고, `client.admin` 을 위한 키가 포함된 `ceph.client.admin.keyring` 파일을 검색합니다. 또한 `ceph-deploy` 및 `ceph-volume` 유틸리티가 OSD 및 MDS 를 준비하고 활성화 할 수있는 keyring 을 검색합니다.

`ceph-deploy admin {node-name}` (Ceph 이 설치되어 있어야 합니다.) 명령어를 실행하면, Ceph 설정 파일과 `ceph.client.admin.keyring` 을 노드의 `/etc/ceph` 디렉토리에 푸시합니다. 해당 노드의 커맨드라인에서 Ceph 관리 기능을 루트 권한으로 실행할 수 있습니다.

### MANUAL DEPLOYMENT
클러스터를 수동으로 배포한다면, 수동으로 모니터를 부트스트래핑하고, `client.admin` 유저와 키링을 만들어 주어야 합니다. 모니터를 부트스트래핑 하기 위해서는, [Monitor Bootstrapping](http://docs.ceph.com/docs/master/install/manual-deployment#monitor-bootstrapping) 절차를 따릅니다. 모니터 부트스트랩을위한 단계는 Chef, Puppet, Juju 등과 같은 타사 배포 도구를 사용할 때 수행해야하는 논리적 단계입니다.

## ENABLING/DISABLING CEPHX
Cephx 를 활성화하려면 MON 과 OSD, 그리고 MDS 에 키가 배포되어 있어야 합니다. 단순히 Cephx 를 켜고 끄는 경우 부트 스트랩 절차를 반복하지 않아도됩니다.

### ENABLING CEPHX
`cephx` 가 활성화 될 때, Ceph 은 기본 탐색 경로인 `/etc/ceph/$cluster.$name.keyring` 에서 키링을 찾습니다. `[global]` 섹션에서 위치를 변경할 수 있지만, 추천하지 않습니다.

아래의 절차는 인증이 비활성화된 클러스터에서 cephx 를 활성화 시키는 단계를 보여줍니다. 만약 키를 이미 생성했다면, 키 생성과 관련된 절차를 건너뛰어도 무방합니다.

1. `client.admin` 의 키를 생성합니다. 그리고 이것을 클라이언트에 복사해 놓습니다.
  ```
  ceph auth get-or-create client.admin mon 'allow *' mds 'allow *' mgr 'allow *' osd 'allow *' -o /etc/ceph/ceph.client.admin.keyring
  ```
  > **Warning**: 이 명령은 기존의 `/etc/ceph/client.admin.keyring` 파일을 덮어씁니다. 이미 이 절차를 수행했다면, 넘어가 주십시오.

2. 모니터 클러스터를 위한 키링을 만들고, 모니터 secret 키를 생성합니다.
  ```
  ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
  ```
3. 모니터 키링을 모든 모니터의 mon 데이터 디렉토리에있는 ceph.mon.keyring 파일에 복사하십시오. 예를 들어, 클러스터 ceph에서 mon.a로 복사하려면:
  ```
  cp /tmp/ceph.mon.keyring /var/lib/ceph/mon/ceph-a/keyring
  ```
4. 모든 MGR 를 위한 secret 키를 생성합니다.
  ```
  ceph auth get-or-create mgr.{$id} mon 'allow profile mgr' mds 'allow *' osd 'allow *' -o /var/lib/ceph/mgr/ceph-{$id}/keyring
  ```
5. 모든 OSD 를 위한 secret 키를 생성합니다.
  ```
  ceph auth get-or-create osd.{$id} mon 'allow rwx' osd 'allow *' -o /var/lib/ceph/osd/ceph-{$id}/keyring
  ```
6. 모든 MDS 를 위한 secret 키를 생성합니다.
  ```
  ceph auth get-or-create mds.{$id} mon 'allow rwx' osd 'allow *' mds 'allow *' mgr 'allow profile mds' -o /var/lib/ceph/mds/ceph-{$id}/keyring
  ```
7. `[global]` 섹션에 다음의 옵션을 세팅함으로서 `cephx` 인증을 활성화합니다.
  ```
  auth cluster required = cephx
  auth service required = cephx
  auth client required = cephx
  ```
8. Ceph 클러스터를 재시작합니다. 추가 정보를 보려면 [Manual Deployment](http://docs.ceph.com/docs/master/install/manual-deployment) 를 참고하세요

### DISABLING CEPHX
다음 절차는 어떻게 Cephx 를 비활성화 하는지 보여줍니다. 클러스터가 안전하다고 판단되면, 인증을 위해 사용되는 컴퓨팅 비용을 줄일 수 있습니다. 하지만 추천하지는 않습니다. 설치 또는 문제 해결 중에 일시적으로 인증을 사용하지 않도록 설정하는 것은 더 쉬우며, 권장합니다.

1. `[global]` 섹션의 인증 관련 옵션들을 수정합니다.
  ```
  auth cluster required = none
  auth service required = none
  auth client required = none
  ```
2. Ceph 클러스터를 재시작합니다.

## CONFIGURATION SETTINGS

### ENABLEMENT

- auth cluster required
  - Description:	활성화 되어 있으면, Ceph 클러스터의 데몬들은(ex, ceph-mon, ceph-osd, ceph-mds, ceph-mgr) 서로 인증 절차를 수행해야 합니다. 올바른 세팅은 `cephx` 혹은 `none` 입니다.
  - Type:	String
  - Required:	No
  - Default:	cephx.
- auth service required
  - Description:	활성화 되어 있으면, Ceph 클러스터의 데몬들은 Ceph 서비스에 엑세스 하기 위해 Ceph 클라이언트에게 인증을 요구합니다. 올바른 세팅은 `cephx` 혹은 `none` 입니다.
  - Type:	String
  - Required:	No
  - Default:	cephx.
- auth client required
  - Description:	활성화 되어 있으면, Ceph 클라이언트는 Ceph 스토리지 클러스터에게 인증을 요구합니다. 올바른 세팅은 `cephx` 혹은 `none` 입니다.
  - Type:	String
  - Required:	No
  - Default:	cephx.

### KEYS
인증이 활성화 된 상태에서 Ceph 을 실행시키면, ceph 관리 커맨드와 Ceph 클라이언트는 Ceph 스토리지 클러스터에 접근하기 위해 인증 키가 필요합니다.

ceph 관리 커맨드와 클라이언트에게 이런 키를 제공하는 가장 일반적인 방법은 Ceph 키링을 `/etc/ceph` 디렉토리에 포함시키는 것입니다. `ceph-deploy` 를 사용한다면, 파일명은 `ceph.client.admin.keyring` (혹은 `$cluster.client.admin.keyring`) 이 됩니다. 만약 `/etc/ceph` 디렉토리에 키링을 포함시켰다면, 설정 파일에 keyring entry 를 명시할 필요가 없습니다.

`client.admin` 키를 포함하고 있기 때문에, 관리 커맨드를 실행시키는 노드에 키링 파일을 복사하는 것을 추천합니다.

이러한 작업을 수행하기 위해서, `ceph-deploy admin` 커맨드를 사용할 것입니다. 구체적인 정보를 보려면 [Create an Admin Host](http://docs.ceph.com/docs/master/rados/deployment/ceph-deploy-admin) 를 참고하세요. 수동으로 진행하려면, 아래 커맨드를 수행합니다.
```
sudo scp {user}@{ceph-cluster-host}:/etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring
```

> **Tip**: `ceph.keyring` 파일이 적절한 권한 (chmod 644) 을 가지고 있는지 확인하십시오

key 세팅을 이용하여 직접적으로 키를 지정하거나 (추천하지 않습니다.), keyfile 세팅을 사용하여 경로를 지정할 수 있습니다.

- keyring
  - Description: 키링 파일의 경로
  - Type:	String
  - Required:	No
  - Default: /etc/ceph/$cluster.$name.keyring,/etc/ceph/$cluster.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin
- keyfile
  - Description: 키 파일의 경로 (key 만 포함하고 있는 파일)
  - Type:	String
  - Required:	No
  - Default: None
- key
  - Description: key (그 자체 텍스트, 추천하지 않습니다.)
  - Type:	String
  - Required:	No
  - Default: None

### DAEMON KEYRINGS
  관리 유저나 배포 툴(ex, ceph-deploy) 는 유저 키링을 생성하는 방법으로 데몬 키링을 생성합니다. 일반적으로, Ceph 은 데몬들의 키링들을 그들의 데이터 경로에 저장합니다. 기본 데몬의 키링의 경로와 데몬이 수행할 수 있는 capability 조건 들은 아래와 같습니다.

- ceph-mon
  - Location: $mon_data/keyring
  - Capabilities: mon 'allow \*'
- ceph-osd
  - Location: $osd_data/keyring
  - Capabilities:	mgr 'allow profile osd' mon 'allow profile osd' osd 'allow \*'
- ceph-mds
  - Location:	$mds_data/keyring
  - Capabilities:	mds 'allow' mgr 'allow profile mds' mon 'allow profile mds' osd 'allow rwx'
- ceph-mgr
  - Location:	$mgr_data/keyring
  - Capabilities:	mon 'allow profile mgr' mds 'allow \*' osd 'allow \*'
- radosgw
  - Location:	$rgw_data/keyring
  - Capabilities:	mon 'allow rwx' osd 'allow rwx'

> **Note**: 모니터 키링 (ex, mon.) 은 키를 포함하고 있지만, capabilites 가 없습니다. 그리고, 클러스터의 `auth` 데이터베이스의 일부분이 아닙니다.

데몬 데이터의 기본 경로는 아래와 같습니다.
```
/var/lib/ceph/$type/$cluster-$id
```

예를 들어, osd.12 는 아래와 같은 경로가 될 것입니다.
```
/var/lib/ceph/osd/ceph-12
```

이 경로를 오버라이드 할 수 있지만, 추천하지는 않습니다.

### SIGNATURES
Ceph Bobtail 및 이후 버전에서는 Ceph이 초기 인증을 위해 설정된 세션 키를 사용하여 엔티티간에 진행중인 모든 메시지를 인증하는 것을 선호합니다. 그러나, 이전 버너의 Ceph 데몬들은 오고 가는 메시지 인증을 어떻게 수행하는지 알지 못합니다. 다른 버전들이 같은 클러스터에 있을 때, 호환성을 맞추기 위해 message signing 이 기본적으로 비활성화 되어 있습니다. Bobtail 이후 버전의 데몬들로만 Ceph 클러스터를 구성하는 경우, 서명이 필요하도록 Ceph 을 설정하십시오.

다른 Ceph 인증방법들과 같이, Ceph는 세분화 된 제어 기능을 제공하므로 클라이언트와 Ceph 사이의 서비스 메시지에 대한 서명을 활성화 / 비활성화 할 수 있습니다. 또한, Ceph 데몬들 간의 서명도 활성/비활성화 시킬 수 있습니다.

- cephx require signatures
  - Description: `true`로 설정되어 있을 경우, Ceph 은 Ceph 클라이언트와 Ceph 클러스터 간의, 그리고 데몬끼리의 모든 트래픽에서 인증을 요구합니다.
  - Type: Boolean
  - Required: No
  - Default: false
- ceph cluster require signatures
  - Description: `true`로 설정되어 있을 경우, Ceph 클러스터 내의 모든 데몬끼리의 메시지에서 서명이 요구됩니다.
  - Type: Boolean
  - Required: No
  - Default: false
- cephx service require signatures
  - Description: `true`로 설정되어 있을 경우, Ceph 은 Ceph 클라이언트와 Ceph 클러스터 간의 모든 트래픽에서 인증을 요구합니다.
  - Type: Boolean
  - Default: false
- cephx sign messages
  - Description: Ceph 버전이 message signing 을 지원하는 경우, Ceph는 모든 메시지에 서명하여 스푸핑 할 수 없게 만듭니다.

### TIME TO LIVE

- auth service ticket ttl
  - Description: Ceph Storage 클러스터가 Ceph 클라이언트에 인증 티켓을 보내면 Ceph Storage 클러스터는 이 값 만큼 유지하는 티켓을 할당합니다.
  - Type: Double
  - Default: 60*60

## BACKWARD COMPATIBILITY
Cuttlefish 이전의 버전에서는, [Cephx](http://docs.ceph.com/docs/cuttlefish/rados/configuration/auth-config-ref/)를 참고하세요.

Ceph Argonaut v0.48 및 이전 버전에서 Cephx 인증을 사용하면 Ceph은 클라이언트와 데몬 간의 초기 통신만 인증합니다. Ceph은 서로에게 보내는 메시지를 인증하지 않으므로 보안에 영향을 미칩니다. Ceph Bobtail 및 후속 버전에서, Ceph은 초기 인증을 위해 설정된 세션 키를 사용하여 엔티티 간의 모든 메시지를 인증합니다.

우리는 Argonaut v0.48 (및 이전 버전)과 Bobtail (및 이후 버전) 사이의 호환성 문제를 확인했습니다. 테스트 할 때, Bobtail (및 이후 버전)과 Argonaut (및 이전 버전) 을 동시에 사용한다면, Argonaut 버전의 데몬은 Bobtail 버전의 메시지 인증을 어떻게 수행할 지 알지 못합니다. 이는 각 버전을 상호 운용하는 것을 불가능하게 합니다.

Argonaut (및 이전 시스템)가 Bobtail (및 후속) 시스템과 상호 작용할 수 있는 수단을 제공함으로써 이 잠재적인 문제를 해결했습니다. 다음과 같이 동작합니다. 기본적으로, 최신 시스템은 어떻게 수행하는지 모르는 오래된 시스템의 서명을 고집하지 않고, 단순히 인증 없이 수락하게 됩니다. 이런 새로운 방식은 서로다른 두 버전 간의 상호작용을 가능하게 합니다. 그러나 **장기적인 솔루션으로는 권장하지 않습니다.**

> 이후는 별로 필요없는 내용...
