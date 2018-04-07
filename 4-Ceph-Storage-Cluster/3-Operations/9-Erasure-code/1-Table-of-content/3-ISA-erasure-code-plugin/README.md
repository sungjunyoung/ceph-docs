# ISA ERASURE CODE PLUGIN

isa plugin 은 [isa](../../Appendix/ISA/README.md) 라이브러리를 캡슐화한 것입니다. 오직 인텔 프로세서에서만 동작합니다.

## CREATE AN ISA PROFILE

새로운 isa ec 프로필을 만드려면, 다음과 같이 입력합니다.
```
ceph osd erasure-code-profile set {name} \
  plugin=isa \
  technique={reed_sol_van|cauchy} \
  [k={data-chunks}] \
  [m={coding-chunks}] \
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
- technique={reed_sol_van|cauchy}
  - Description: ISA 플러그인은 두개의 [Reed Solomon](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction) 형식을 사용합니다. 만약 reed_sol_val 으로 세팅되면, [Vandermonde](https://en.wikipedia.org/wiki/Vandermonde_matrix) 이 되고, cauchy 로 되면, [Cauchy](https://en.wikipedia.org/wiki/Cauchy_matrix)가 됩니다.
  - Type: String
  - Required: No.
  - Example: reed_sol_van
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
