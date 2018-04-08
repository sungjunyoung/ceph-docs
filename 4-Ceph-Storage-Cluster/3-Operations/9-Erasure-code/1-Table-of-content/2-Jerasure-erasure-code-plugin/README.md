# JERASURE ERASURE CODE PLUGIN

jerasure plugin 은 가장 일반적이고 유연한 플러그인이며, Ceph ec pool 의 기본 플러그인이기도 합니다.

jerasure plugin 은 [Jerasure](../../Appendix/Jerasure/README.md) 라이브러리를 캡슐화한 것입니다. 매개변수에 대해 더 자세히 알고 싶다면, jerasure documenation 을 읽는 것을 추천합니다.

## CREATE A JERASURE PROFILE

새로운 jerasure ec 프로필을 만드려면, 다음과 같이 입력합니다.
```
ceph osd erasure-code-profile set {name} \
  plugin=jerasure \
  k={data-chunks} \
  m={coding-chunks} \
  technique={reed_sol_van|reed_sol_r6_op|cauchy_orig|cauchy_good|liberation|blaum_roth|liber8tion} \
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
  - Example: 4
- m={coding-chunks}
  - Description: 각 오브젝트에서 코딩 청크를 계산하고, 각기 다른 OSD 에 저장합니다. 코딩 청크의 갯수는 데이터 손실 없이 다운시킬 수 있는 OSD 의 갯수와 동일합니다.
  - Type: Integer
  - Required: Yes.
  - Example: 2
- technique={reed_sol_van|reed_sol_r6_op|cauchy_orig|cauchy_good|liberation|blaum_roth|liber8tion}
  - Description: 가장 유연한 테크닉은 `reed_sol_van` 입니다: k, m 을 세팅하는 것으로 충분합니다. `cauchy_good` 테크닉은 더 빠를 수 있지만, packetsize 를 조심스럽게 선택해야 합니다. reed_sol_r6_op, liberation, blaum_roth, liber8tion 은 RAID6 와 동일하며, m=2 일때만 설정할 수 있습니다.
  - Type: String
  - Required: No.
  - Example: reed_sol_van
- packetsize={bytes}
  - Description: 인코딩은 한 번에 바이트 크기의 패킷에 대해 수행됩니다. 올바른 패킷 크기를 선택하는 것은 어렵습니다. jerasure 문서에 주제에 대한 광범위한 정보가 들어 있습니다.
  - Type: Integer
  - Required: No.
  - Example: 2048
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
