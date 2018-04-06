# ERASURE CODE PROFILES

Erasure code 는 **profile** 에 의해서 정의되고, 이것은 관련된 CRUSH 룰과 ec pool 을 만드는데 사용됩니다.

**기본** erasure code profile (Ceph 클러스터가 초기화 될 때 생성되는) 는 2 copy 와 같은 레벨의 견고함을 제공하면서, 디스크 공간을 25% 덜 차지합니다. 이것은 k=2, m=1 로 설정된 profile 이며, 세 개의 OSD (k+m == 3) 에 나누어져 있고 이들 중 하나가 없어져도 된다는 뜻입니다.

raw storage 를 늘리지 않고 durability 를 높이기 위해서, 새로운 프로필을 만들 수 있습니다. 예를들어, k=10, m=4 로 이루어진 프로필은 14개의 OSD (k+m=14) 에 분산하여 4개의 OSD 손실을 (m=4) 견딜 수 있습니다. 처음에 오브젝트는 10개의 청크로 나누어지고, 리커버리를 위해 4개의 코딩 청크가 계산되어집니다. (코딩 청크는 데이터 청크와 같은 사이즈입니다.) 공간은 40% 정도밖에 더 쓰지 않습니다.

- [Jerasure erasure code plugin](../2-Jerasure-erasure-code-plugin/README.md)
- [ISA erasure code plugin](../3-ISA-erasure-code-plugin/README.md)
- [Locally repairable erasure code plugin](../4-Locally-repairable-erasure-code/README.md)
- [SHEC erasure code plugin](../5-SHEC-erasure-code-plugin/README.md)

## OSD ERASURE-CODE-PROFILE SET
새로운 erasure code 프로필을 만드려면,

```
ceph osd erasure-code-profile set {name} \
  [{directory=directory}] \
  [{plugin=plugin}] \
  [{stripe_unit=stripe_unit}] \
  [{key=value} ...] \
  [--force]
```

- {directory=directory}
  - Description: erasure code plugin 이 로드된 디렉토리를 설정합니다.
  - Type: String
  - Required: No.
  - Default: /usr/lib/ceph/erasure-code
- {plugin=plugin}
  - Description: 코딩 청크 계산과 누실된 청크 복구에 사용할 erasure code plugin 을 설정합니다. [list of available plugins](http://docs.ceph.com/docs/master/rados/operations/erasure-code-profile/#list-of-available-plugins) 를 참고하세요.
  - Type: String
  - Required: No.
  - Default: jerasure
- {stripe_unit=stripe_unit}
  - Description: 단일 stripe 당 데이터 청크 안의 데이터의 양. 예를들어, 2개의 데이터 청크와 stripe_unit=4K 로 된 프로필은 0-4K 를 청크 0에, 4K-8K 를 청크 1에, 그리고 8K-12K 를 다시 청크 0에 넣습니다. 최상의 성능을 위해서는이 값이 4K의 배수여야 합니다. 기본값은 모니터 구성 옵션에서 가져옵니다.
  - Type: String
  - Required: No.
- {key=value}
  - Description: 나머지 키 / 값 쌍의 의미는 ec 플러그인에 의해 정의됩니다.
  - Type: String
  - Required: No.
- --force
  - Description: 같은 이름일때 프로필을 덮어 쓰고, 4K 의 배수가 아닌 stripe_unit 을 허가합니다.
  - Type: String
  - Required: No

## OSD ERASURE-CODE-PROFILE RM

ec 프로필을 삭제하려면, 다음을 입력합니다.
```
ceph osd erasure-code-profile rm {name}
```
어떤 pool에서 프로필을 사용중일 경우, 삭제는 실패합니다.

## OSD ERASURE-CODE-PROFILE GET

ec 프로필을 조회하려면, 다음을 입력합니다.
```
ceph osd erasure-code-profile get {name}
```

## OSD ERASURE-CODE-PROFILES

모든 ec 프로필을 조회합니다.
```
ceph osd erasure-code-profile ls
```
