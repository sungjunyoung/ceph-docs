# Sotrage device
디스크에 데이터를 저장하는 두가지의 Ceph 데몬이 있습니다.
- Ceph OSDs (Object Storage Daemons) 들은 Ceph 에서 대부분의 데이터가 저장되는 곳입니다. 일반적으로 말하면, 각각의 OSD 는 HDD나 SSD 같은 정통적인 single storage device 에서 돌아가거나, 파티션으로 나누어진 device의 조합에서 돌아갑니다. 클러스터에서 OSD의 숫자는 일반적으로 얼마나 많은 데이터가 저장되는지, 얼마나 큰 데이터가 저정되는지, 레플리케이션이나 이레이져 코딩의 레벨에 따라 달라집니다.
- Ceph Monitor 데몬은 클러스터 멤버십이나 인증 정보와 같은 중요한 클러스터 상태를 다룹니다. 작은 클러스터에서는 몇기가바이트가 필요하고, 클러스터가 커질수록 10기가바이트에서 100기가바이트까지 커질 수 있습니다.

## OSD Backends
OSD가 그들의 데이터를 저장하는 데에는 두가지 방법이 있습니다. Luminous 12.2.z 버전에서는, 새로운 기본값 (권장) 백엔드는 BlueStore 입니다. Luminous 이전 버전에서는, 기본값은 FileStore 입니다.

### Bluestore
BlueStore (이하 블루스토어) 는 Ceph OSD 워크로드를 위한 디스크 위의 데이터를 매니징하기 위해 특별히 디자인된 특수 목적의 storage backend 입니다. 10년 이상 FileStore 를 사용한 OSD 를 서포트하고, 매니징하면서 쌓인 경험에서 나왔습니다. BlueStore 의 특징은 다음과 같습니다.
- 스토리지 디바이스의 직접 관리. BlueStore 는 블록 디바이스 혹은 파티션을 소모합니다. 이것은 퍼포먼스를 제한하고, 복잡성을 증가시키는 추상 중재 레이어(XFS 같은 로컬 파일 시스템)를 회피합니다.
- RocksDB 를 이용한 메타데이터 관리. 우리는 내부의 메타데이터 (오브젝트 이름에 따른 디스크 블록의 위치 등) 를 관리하기 위해 RocksDB 의 키/벨류 데이터베이스를 사용합니다.
- full-data / metadata 체크섬. 기본적으로 블루스토어에 쓰여진 모든 데이터와 메타데이터는 하나 이상의 체크섬으로 보호됩니다. 확인 없이는 어떠한 데이터나 메타데이터도 디스크로부터 읽어들일 수 없습니다.
- Inline compression. 데이터에 쓰여지는 데이터들은 쓰여지기 전에 선택적으로 압축될 수 있습니다.
- Multi-device 메타데이터 티어링. 블루스토어는 내부 journal 을 따로 나누어 빠른 속도의 device 로 (SSD, NVMe, NVDIMM) 지정할 수 있게 합니다.
- 효율적인 copy-on-write. RBD 와 CephFS 스냅샷은 블루스토어에 효과적이도록 구현된 copy-on-write clone mechanism 에 의존하고 있습니다. 이는 regular 스냅샷과 erasure coded pool 에 있어 효율적인 IO를 제공합니다.

더 많은 정보는, [BlueStore Config Reference](http://docs.ceph.com/docs/master/rados/configuration/bluestore-config-ref/) 와 [BlueStore Migration](http://docs.ceph.com/docs/master/rados/operations/bluestore-migration/) 를 참고하세요

### Filestore
FileStore(이하 파일스토어)는 기존 ceph 에서 오브젝트를 저장하던 방식입니다. 이는 키/밸류 데이터베이스 (현재 RocksDB, 과거 LevelDB) 와 함께 표준 파일 시스템(일반적으로 XFS) 에 의존합니다.

파일스토어는 잘 테스트되어지고 실제 서비스로 널리 쓰여졌습니다. 그러나, 오브젝트 데이터를 저장의 전통 파일시스템과의 의존성과, 오버 디자인으로 인해 많은 퍼포먼스 저하로 고통받았습니다.

비록 파일스토어가 btrfs 나 ext4 같은 많은 POSIX-compatible 한 파일 시스템에서 일반적으로 쓰여지지만, XFS 를 사용하기를 권장합니다. btrfs 와  ext4 는 알려진 버그들과 성능저하가 존재하고, 이는 데이터 손실을 야기할 수 있습니다. ceph provisioning 툴의 기본값은 XFS 입니다.

더 많은 정보는, [Filestore Config Reference](http://docs.ceph.com/docs/master/rados/configuration/filestore-config-ref/) 를 참고하세요
