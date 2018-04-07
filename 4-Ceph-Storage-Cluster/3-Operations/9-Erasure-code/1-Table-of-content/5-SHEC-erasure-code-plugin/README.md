# SHEC ERASURE CODE PLUGIN

shec plugin 은 [isa](../../Appendix/multiple-SHEC/README.md) 라이브러리를 캡슐화한 것입니다. Reed Solomon code 보다 더 효율적으로 Ceph의 데이터를 복구할 수 있습니다.

## CREATE AN SHEC PROFILE

새로운 shec ec 프로필을 만드려면, 다음과 같이 입력합니다.
```
ceph osd erasure-code-profile set {name} \
  plugin=shec \
  [k={data-chunks}] \
  [m={coding-chunks}] \
  [c={durability-estimator}] \
  [crush-root={root}] \
  [crush-failure-domain={bucket-type}] \
  [crush-device-class={device-class}] \
  [directory={directory}] \
  [--force]
```

- k={data-chunks}
  - Description: 각기 다른 OSD 에 저장되는 오브젝트가 나누어진 data-chunks 들의 갯수
  - Type: Integer
  - Required: Yes.
  - Example: 7
- m={coding-chunks}
  - Description: 각 오브젝트에서 코딩 청크를 계산하고, 각기 다른 OSD 에 저장합니다. 코딩 청크의 갯수는 데이터 손실 없이 다운시킬 수 있는 OSD 의 갯수와 동일합니다.
  - Type: Integer
  - Required: Yes.
  - Example: 3
- c={durability-estimator}
  - Description: 각 데이터 청크가 계산 범위에 포함되는 패리티 청크의 수입니다. 이 숫자는 durability estimator 로 사용됩니다. 예를들어, c=2 이면, 2 개의 OSD 가 데이터 손실 없이 내려갈 수 있습니다.
- crush-root={root}
  - Description: CRUSH 룰의 첫번째 단계로 사용되는 crush bucket 의 이름.
  - Type: String
  - Required: No.
  - Example: default
- crush-failure-domain={bucket-type}
  - Description: 동일한 failure domain 을 가진 bucket 에 두 개의 청크가 없는지 확인함. 예를들어, 만약 failure domain 이 **host** 라고 되어 있으면, 같은 host 에 두개의 청크가 저장되지 않습니다. **step chooseleaf host** 같은 CRUSH rule step 을 만드는 데에 사용됩니다.
  - Type: String
  - Required: No.
  - Example: host
- crush-device-class={device-class}
  - Description: CRUSH map 에서 crush device 클래스 이름을 사용하여 특정 클래스 (예 : ssd 또는 hdd)의 장치로 배치를 제한합니다.
  - Type: String
  - Required: No.
- directory={directory}
  - Description: ec plugin 이 로드되는 디렉토리 이름으로 설정합니다.
  - Type: String
  - Required: No.
  - Default: /usr/lib/ceph/erasure-code
- --force
  - Description: 같은 이름의 존재하는 프로필을 덮어씁니다.
  - Type: String
  - Required: No.

## BRIEF DESCRIPTION OF SHEC'S LAYOUTS
### SPACE EFFICIENCY
공간 효율은 오브젝트에서 모든 데이터 청크와 데이터 청크의 비율이며 k / (k + m)로 표시됩니다. 공간 효율성을 높이려면 k를 높이거나 m을 줄여야합니다.

```
space efficiency of SHEC(4,3,2) = 4/(4+3) = 0.57
SHEC(5,3,2) or SHEC(4,2,2) improves SHEC(4,3,2)'s space efficiency
```
### DURABILITY
SHEC 의 세번째 매개변수인 c 는 durability estimator 입니다. 이것은 데이터 손실 없이 내릴 수 있는 OSD 갯수의 근사치입니다.

```
durability estimator of SHEC(4,3,2) = 2
```

### RECOVERY EFFICIENCY
복구 효율 계산에 대한 설명은 이 문서의 범위를 벗어나지만 적어도 c 를 증가시키지 않으면서 m 을 늘리면 복구 효율성이 향상됩니다. (그러나, 이 경우 공간 효율성을 희생할 수 있습니다.)

```
SHEC(4,2,2) -> SHEC(4,3,2) : 복구 효율성 증가
```

## ERASURE CODE PROFILE EXAMPLE
```
$ ceph osd erasure-code-profile set SHECprofile \
     plugin=shec \
     k=8 m=4 c=3 \
     crush-failure-domain=host
$ ceph osd pool create shecpool 256 256 erasure SHECprofile
```
