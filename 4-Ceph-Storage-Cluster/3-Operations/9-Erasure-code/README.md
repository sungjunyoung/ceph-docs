# ERASURE CODE

Ceph Pool은 OSD (즉, 디스크 당 하나의 OSD가 있기 때문에 대부분의 경우 디스크)의 손실을 막습니다. Pool을 만들 때 기본 선택 항목이 Replicated 되므로 모든 개체가 여러 디스크에 복사됩니다. 공간을 절약하기 위해 Erasure Code Pool 유형을 대신 사용할 수 있습니다.

## CREATING A SAMPLE ERASURE CODED POOL

가장 간단한 Erasure Code 가 적용된 Pool 은 RAID5 와 같으며 세개 이상의 호스트가 요구됩니다.
```sh
$ ceph osd pool create ecpool 12 12 eraure
pool 'ecpool' created
$ echo ABCDEFGHI | rados --pool ecpool put NYAN -
ABCDEFGHI
```
> **Note** Pool 을 만들 때 12 라는 숫자는 pg group 의 숫자를 의미합니다.

## ERASURE CODE PROFILES

기본적인 Erasure Code profile 은 하나의 OSD 유실을 막을 수 있습니다. 그러나 replicated: 2 인 경우에서, 1TB 의 데이터를 저장하는데 2TB 가 드는 반면에 1.5TB 만을 요구합니다. 기본 프로필은 다음과 같습니다.

```sh
$ ceph osd erasure-code-profile get default
k=2
m=1
plugin=jerasure
cursh-failure-domain=host
technique=read_sol_van
```

pool 이 만들어 진 후에는 수정할 수 없으므로 올바른 프로필을 선택하는 것이 중요합니다.: 다른 프로필의 pool 을 만드려면 pool 을 만든 후, 이전 pool 에서 새로운 pool 로 모든 오브젝트를 옮겨야 합니다.

프로필에서 가장 중요한 파라미터는 K, M 그리고 crush-failure-domain 입니다. 왜나하면, 이것은 스토리지 오버헤드와 데이터 견고성을 결정하기 때문입니다. 예를들어, 원하는 아키텍처가 40% 오버 헤드의 스토리지 오버 헤드로 2 개의 랙 손실을 견뎌야하는 경우 다음 프로파일을 정의 할 수 있습니다.

```sh
$ ceph osd erasure-code-profile set myprofile \
  k=3 \
  m=2 \
  crush-failure-domain=rack
$ ceph osd pool create ecpool 12 12 erasure myprofile
$ echo ABCDEFGHI | rados --pool ecpool put NYAN -
$ rados --pool ecpool get NYAN -
ABCDEFGHI
```

NYAN 오브젝트는 세개로 나누어지고 (K=3), 추가적인 청크가 생성됩니다(M=2). M 의 값은 데이터 손실 없이 얼마나 많은 OSD 가 동시에 잃을 수 있는지 정의합니다. `cursh-failure-domain=rack` 은 같은 rack 에 두개의 청크가 함께 있지 않도록 CRUSH 룰을 만듭니다.

![ec-architecture-example](/Images/ec-architecture-example.png)

더 많은 정보는 [erasure code profile](http://docs.ceph.com/docs/master/rados/operations/erasure-code-profile/) 을 참고하세요.

## ERASURE CODING WITH OVERWRITES

기본적으로 Erasure Coded Pool 은 전체 오브젝트를 쓸 때 추가하는 형태의 RGW와 같은 용도로만 작동합니다.

Luminous 버전부터, Erasure Coded pool 에 대해 per-pool setting 으로 부분적인 쓰기가 활성화됩니다. 이는 RBD 와 Cephfs 가 그들의 데이터를 Erasure Coded pool 에 저장되도록 합니다.

```
ceph osd pool set ec_pool allow_ec_overwrite true
```

이는 bluesotre 의 체크섬 작업이 deep-scrub 동안에 bitrot 이나 corruption 을 감지하기 때문에, bluestore 기반의 OSD 에서만 가능합니다. 안전하지 않은 것 외에도 filestore 를 ec overwrites 와 함께 사용하면 bluestore 에 비해 성능이 떨어집니다.

Erasure Coded pool 들은 omap 을 지원하지 않습니다. 따라서 RBD 및 CephFS와 함께 사용하려면 데이터를 ec pool 에 저장하고 메타 데이터를 replicated pool 에 저장하도록 지시해야합니다. RBD 에서는, erasure coded pool 을 이미지를 만들 때 --data-pool 로 만드는 것을 의미합니다.

```
rbd create --size 1G --data-pool ec_pool replicated_pool/image_name
```
CephFS 에서는, [file layouts](http://docs.ceph.com/docs/master/cephfs/file-layouts) 를 통해서 filesystem 생성 시 erasure coded pool 을 기본 pool 로 지정할 수 있습니다.

## ERASURE CODED POOL AND CACHE TIERING

ec pool 은 replicated pool 보다 많은 리소스가 필요하며 omap과 같은 일부 기능이 부족합니다. 이런 한계를 극복하기 위해, ec pool 앞에 [cache tier](http://docs.ceph.com/docs/master/rados/operations/cache-tiering) 를 세팅할 수 있습니다.

예를 들어, 만약 hot-storage pool 이 빠른 스토리지로 이루어진 경우:

```
$ ceph osd tier add ecpool hot-storage
$ ceph osd tier cache-mode hot-storage writeback
$ ceph osd tier set-overlay ecpool hot-storage
```

는 hot-storage pool 를 ecpool 의 writeback 모드인 티어로 놓게 되며, 이것은 ecpool 의 모든 write 와 read 에 있어 hot-storage 를 사용하여 유연성과 속도에 이점을 보게 됩니다.

더 많은 정보는 [cache-tiering](http://docs.ceph.com/docs/master/rados/operations/cache-tiering/) 을 참고하세요.

## GLOSSARY
- chunk
  - encoding 함수가 호출되면, 이것은 같은 사이즈의 chunk들 (원본 오브젝트를 재구성하기 위해 연결될 수 있는 데이터 청크 및 손실 된 청크를 재 구축하는 데 사용할 수있는 코딩 청크.) 을 리턴합니다.
- K
  - 데이터 청크의 갯수 (원본 오브젝트가 나누어진 청크의 갯수). 예를들어, 만약 K=2 이고, 10KB 의 오브젝트가 있다면 각각 5KB 로 나누어집니다.
- M
  - 코딩 청크의 갯수 (encoding 함수로 인해 계산된 추가적인 코딩 청크의 갯수). 만약 2개의 코딩 청크가 존재한다면, 이는 데이터 손실 없이 2개의 OSD 가 내려갈 수 있다는 것입니다.

## TABLE OF CONTENT
- [Erasure code profiles](./1-Table-of-content/1-Erasure-code-profile/README.md)
- [Jerasure erasure code plugin](./1-Table-of-content/2-Jerasure-erasure-code-plugin/README.md)
- [ISA erasure code plugin](./1-Table-of-content/3-ISA-erasure-code-plugin/README.md)
- [Locally repairable erasure code plugin](./1-Table-of-content/4-Locally-repairable-erasure-code/README.md)
- [SHEC erasure code plugin](./1-Table-of-content/5-SHEC-erasure-code-plugin/README.md)
