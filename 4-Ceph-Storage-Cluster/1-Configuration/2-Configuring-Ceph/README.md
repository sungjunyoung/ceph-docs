# CONFIGURING CEPH
Ceph 서비스를 시작했다면, 초기 프로세스는 여러 데몬을 백그라운드에서 실행시킵니다. Ceph Storage Cluster 는 세가지 타입의 데몬을 실행합니다.

- Ceph Monitor (ceph-mon)
- Ceph Manager (ceph-mgr)
- Ceph OSD Daemon (ceph-osd)

Ceph filesystem 을 지원하는 Ceph Storage Cluster 는 적어도 하나의 Ceph Metadata Server (ceph-mds) 를 실행합니다. Ceph Object Storage 를 지원하는 클러스터는 Ceph Gateway daemons (radosgw) 를 실행합니다.

각각의 데몬은 기본값으로 되어 있는 여러 설정 옵션들이 있으며, 이런 설정 옵션들을 변경함으로서 적절한 행동을 취할 수 있습니다.

## OPTION NAMES
모든 Ceph 설정 옵션들은 underscore(_)로 이어져 소문자로 이루어진 unique한 이름을 가지고 있습니다.

옵션의 이름이 커맨드라인에서 쓰여질 때에는, underscore(_) 와 대시 (-)는 같은 것으로 간주됩니다. ( --mon-host 와 --mon_host 는 같습니다. )

설정 파일에서 옵션 이름이 나타날 때는,공백은  마찬가지로 underscore 나 dash 로 사용될 수 있습니다.

## CONFIG SOURCES
각각의 Ceph  데몬, 프로세스와 라이브러리는 아래의 여러 소스들로부터 설정을 가져옵니다. 리스트 아래에 있는 항목들이 위에 있는 항목들을 덮어쓸 수 있습니다.

- compiled-in 기본 값
- 모니터 클러스터의 설정 데이터베이스
- 로컬 호스트에 있는 설정 파일
- 환경변수
- 커맨드라인 arguments
- 관리자가 셋팅한 런타임 오버라이드

이러한 것들로부터 Ceph 프로세스는 설정 옵션들을 가져옵니다. 프로세스는 이후에 모니터 클러스터에게 전체 클러스터를 위한 설정을 알립니다. 완료되면, 데몬이나 프로세스가 진행되게 됩니다.

## CONFIGURATION SECTIONS
모든 주어진 프로세스 또는 데몬은 각 구성 옵션에 대해 단일 값을가집니다. 그러나, 옵션에 대한 값은 동일한 유형의 데몬에서도 서로 달라질 수 있습니다. MON 설정 데이터베이스나 로컬 설정파일의 Ceph 옵션들은 어떤 데몬이나 클라이언트에 적용되어질지에 따라 다른 섹션에 존재합니다.

다음과 같은 섹션들이 존재합니다.
- global
  - Description: `global` 섹션에 있는 세팅들은 Ceph Storage Cluster 에 있는 모든 데몬과 클라이언트들에게 영향을 미칩니다.
  - Example: `log_file = /var/log/ceph/$cluster-$type.$id.log`
- mon
  - Description: `mon` 섹션의 세팅들은 Ceph Storage Cluster 의 모든 `ceph-mon` 데몬들에게 영향을 미칩니다. 이 세팅들은 `global` 섹션의 세팅들을 덮어쓸 수 있습니다.
  - Example: `mon_cluster_log_to_syslog = true`
- mgr
  - Description: `mgr` 섹션의 세팅들은 Ceph Storage Cluster 의 모든 `ceph-mgr` 데몬들에게 영향을 미칩니다. 이 세팅들은 `global` 섹션의 세팅들을 덮어쓸 수 있습니다.
  - Example: `mgr_stats_period = 10`
- osd
  - Description: `osd` 섹션의 세팅들은 Ceph Storage Cluster 의 모든 `ceph-osd` 데몬들에게 영향을 미칩니다. 이 세팅들은 `global` 섹션의 세팅들을 덮어쓸 수 있습니다.
  - Example: `osd_op_queue = wpq`
- mds
  - Description: `mds` 섹션의 세팅들은 Ceph Storage Cluster 의 모든 `ceph-mds` 데몬들에게 영향을 미칩니다. 이 세팅들은 `global` 섹션의 세팅들을 덮어쓸 수 있습니다.
  - Example: `mds_cache_size = 10G`
- client
  - Description: `client` 섹션의 세팅들은 모든 Ceph Client 들에게 영향을 미칩니다. (예, 마운트된 Ceph Filesystem들, 마운트된 Ceph Block Devices...)
  - Example: `objecter_infligt_ops = 512`

`mon.foo`, `osd.123`, `client.smith` 처럼 특정 데몬이나 클라이언트 이름을 섹션에 지정할 수도 있습니다.

더 세부적인 위치의 세팅이 상위의 세팅들을 덮어쓰게 되어 최종적으로 적용됩니다. 예를 들어, `global`, `mon`, `mon.foo` 에 같은 이름의 옵션이 다른 값으로 적용되어 있다면, `mon.foo`가 적용될 것입니다.

로컬 설정 파일로부터의 값은 항상 모니터 설정 데이터베이스보다 우선 적용됩니다.

## METAVARIABLES
Metavariables 는 Ceph Storage Cluster 설정을 극적으로 단순화시킵니다. metavariable 이 설정 값에 셋팅되어 있으면, Ceph 은 이 값이 사용되는 시점에선 metavariable을 concrete value 로 확장시킵니다. Ceph metavariable 들은 bash 에서의 변수 사용과 유사합니다.

Ceph 은 다음과 같은 metavariable을 지원합니다.

- $cluster
  - Description: Ceph Storage Cluster 이름으로 확장합니다. 여러 Ceph Storage Cluster 를 같은 하드웨어에서 운영하고 있을 때 유용합니다.
  - Example: `etc/ceph/$cluster.keyring`
  - Default: ceph
- $type
  - Description: 데몬이나 프로세스 타입으로 확장합니다.(`mds`, `osd`, `mon`)
  - Example: `/var/lib/ceph/$type/$cluster-$id`
- $host
  - Description: 프로세스가 실행중인 호스트 이름으로 확장합니다.
- $name
  - Description: $type.$id 로 확장합니다.
  - Example: `/var/run/ceph/$cluster-$name.asok`
- $pid
  - Description: 데몬의 pid로 확장합니다.
  - Example: `/var/run/ceph/$cluster-$name-$pid.asok`

## THE CONFIGURATION FILE
시작 시점에서, Ceph 은 다음의 위치에서 설정 파일들을 찾습니다.

1. `$CEPH_CONF` (환경변수)
2. `-c path/path` (커맨드라인 인자)
3. `/etc/ceph/$cluster.conf`
4. `~/.ceph/$cluster.conf`
5. `./$cluster.conf` (현재 디렉토리)
6. FreeBSD 시스템에서만, `/usr/local/etc/ceph/$cluster.conf`

`$cluster`는 클러스터 이름입니다. (기본값 `ceph`)

Ceph 설정파일은 `ini` 스타일을 따릅니다. 샵(`#`) 이나 세미콜론(`;`)으로 주석을 추가할 수 있습니다. 예시:

```
# <--A number (#) sign precedes a comment.
; A comment may be anything.
# Comments always follow a semi-colon (;) or a pound (#) on each line.
# The end of the line terminates a comment.
# We recommend that you provide comments in your configuration file(s).

```

### CONFIG FILE SECTION NAMES
설정 파일은 `[global]`이나 `[mon.foo]` 같은 섹션으로 나누어집니다.

글로벌 세팅은 Ceph Storage Cluster 안의 모든 데몬의 인스턴스에게 영향을 미칩니다. `[global]` 세팅을 다음과 같이 덮어쓸 수 있습니다.

1. 특정한 프로세스 타입 내의 세팅을 변경 (`[osd]`,`[mon]`,`[mds]`)
2. 특정한 프로세스를 변경 (`[osd.1]`)

## MONITOR CONFIGURATION DATABASE
모니터 클러스터는 전체 클러스터가 사용하는 설정 옵션들의 데이터베이스를 관리하여 전체 시스템의 중앙 설정 관리를 간소화합니다. 대부분의 설정 옵션은 관리 용이성과 투명성을 위해 이곳에 저장할 수 있고 저장해야 합니다.

일부 설정은 모니터 연결, 인증 및 설정 정보 fetch 기능에 영향을 미치므로 여전히 로컬 구성 파일에 저장해야 할 수 있습니다. `DNS SRV records`를 사용함으로서 피할 수는 있지만, 이 경우, 대부분은 `mon_host` 옵션에만 적용됩니다.

### SECTION AND MASKS
모니터에 저장되는 설정 옵션들은 설정 파일이 할 수 있는 것처럼 global, daemon type, 특정 daemon 에 한헤 적용할 수 있습니다.

또한 옵션에 적용되는 데몬 또는 클라이언트를 추가로 제한하기 위해 옵션에 관련된 마스크가 있을 수 있습니다. 마스크는 두가지 형태를 가집니다.

1. `type:location` `type`은 `rack`이나 `host`같은 CRUSH 속성이며, `location`은 속성에 대한 값입니다. 예를 들어, `host:foo` 는 옵션들을 오직 특정 호스트에 실행 중인 데몬이나 클라이언트로 제한할 것입니다.
2. `class:device-class` `device-class`는 CRUSH 디바이스 클래스의 이름입니다. (hdd, ssd) 예를들어, `class:ssd` 는 옵션들을 오직 `SSD`를 백엔드로 가지는 `OSD`들로 제한할 것입니다. (이 마스크는 `OSD` 가 아닌 데몬이나 클라이언트들에게는 영향을 미치지 않습니다.)

설정 옵션들을 세팅할때, `who`는 섹션 이름, 마스크 혹은 슬래시(/) 로 구분된 문자의 조합이 될 수 있습니다. 예를 들어, `osd/rack:foo`는 foo rack 의 모든 OSD 데몬을 의미합니다.

설정 옵션들을 볼 때, 섹션 이름과 마스크는 일반적으로 가독성을 높이기 위해 별도의 필드 또는 열로 구분됩니다.

### COMMANDS
클러스터를 설정하기 위해 다음의 CLI 커맨드들이 사용됩니다.
- `ceph config dump` 는 클러스터의 모든 설정 데이터베이스를 dump 하여 보여줍니다.
- `ceph config get <who>` 는 특정한 데몬이나 클라이언트의 설정을 보여줍니다. (ex, `mds`)
- `ceph config set <who> <option> <value>` 는 모니터 설정 데이터베이스에 설정을 세팅합니다.
- `ceph config show <who>` 는 실행 중인 데몬의 설정을 보여줍니다. 이러한 설정은 사용 중인 로컬 구성 파일이 있거나 커맨드라인이나 런타임에서 옵션이 재정의된 경우 모니터에 저장된 설정과 다를 수 있습니다. 설정 값의 일부가 출력됩니다.
- `ceph config assimilate-conf -i <input file> -o <output file>` 는 입력 파일에서 구성 파일을 손상시키고(?) 유효한 옵션을 모니터의 구성 데이터베이스로 이동합니다. 모니터가 컨트롤하지 못하거나, 잘못된 설정들은 output 파일로 저장됩니다. 이 커맨드는 중앙화된 모니터 베이스의 설정으로 기존 설정 파일들을 옮기는 데 유용합니다.

### HELP
특정한 옵션의 사용법을 다음의 명령어로 얻을 수 있습니다.
```
ceph config help <option>
```
실행 중인 모니터애 컴파일된 설정 스키마를 사용합니다. 만약 혼재된 버전의 클러스터를 사용한다면, (ex, 업그레이드 중) 특정한 실행 중인 데몬으로부터 설정 스키마를 쿼리할 수 있습니다.
```
ceph daemon <name> config help [option]
```
예를 들어,
```
$ ceph config help log_file
log_file - path to log file
  (std::string, basic)
  Default (non-daemon):
  Default (daemon): /var/log/ceph/$cluster-$name.log
  Can update at runtime: false
  See also: [log_to_stderr,err_to_stderr,log_to_syslog,err_to_syslog]
```
혹은,
```
$ ceph config help log_file -f json-pretty
{
    "name": "log_file",
    "type": "std::string",
    "level": "basic",
    "desc": "path to log file",
    "long_desc": "",
    "default": "",
    "daemon_default": "/var/log/ceph/$cluster-$name.log",
    "tags": [],
    "services": [],
    "see_also": [
        "log_to_stderr",
        "err_to_stderr",
        "log_to_syslog",
        "err_to_syslog"
    ],
    "enum_values": [],
    "min": "",
    "max": "",
    "can_update_at_runtime": false
}
```
`level` 옵션은 `basic`, `advanced`, `dev` 가 될 수 있습니다. `dev` 옵션은 개발자를 위해 사용되고, 일반적으로 테스트 목적으로 사용합니다. 그리고 operator 에 의해 사용하는 것을 권장하지 않습니다.

### RUNTIME CHANGE
대부분의 경우에서, Ceph 은 런타임에서 설정을 바꿀 수 있습니다. 로그 아웃풋을 늘리거나, 줄일 때, 디버그 세팅을 켜거나 끌때, 심지어 런타임 최적화까지 유용하게 사용할 수 있습니다. 일반적으로, 설정 옵션들은 `ceph config set ` 명령을 통해서 업데이트 할 수 있습니다. 예를 들어, 특정한 OSD 데몬의 로그 레벨을 debug 로 변경하려면:
```
ceph config set osd.123 debug_ms 20
```
만약 로컬 설정 파일에서 같은 옵션을 커스터마이즈 했다면, 모니터 셋팅은 무시될 수 있습니다. (이 명령어는 로컬 설정 파일보다 낮은 우선순위를 가집니다.)

#### OVERRIDE VALUES
일시적으로 Ceph CLI 를 통해서 설정을 변경할 수도 있습니다. 이는 실행 중인 프로세스에만 영향을 미치며, 재시작 시 초기화 될 수 있습니다.

값을 덮어쓰는 것은 다음의 두가지 방법으로 할 수 있습니다.

1. 아무 호스트에서 네트워크를 통해서 데몬에게 메시지를 보낼 수 있습니다.
  ```
  ceph tell <name> config set <option> <value>
  ```
  예시:
  ```
  ceph tell osd.123 config set debug_osd 20
  ```
  `tell` 커맨드는 데몬 식별자 와일드카도로도 사용이 가능합니다. 예를 들어, 모든 OSD 데몬에게 디버그 레벨을 적용하려면,
  ```
  ceph tell osd.* config set debug_osd 20
  ```
2. 프로세스가 실행 중인 프로세스에서, `/var/run/ceph` 의 소켓을 통해 프로세스에 직접 연결할 수도 있습니다.
  ```
  ceph daemon <name> config set <option> <value>
  ```
  예시:
  ```
  ceph daemon osd.4 config set debug_osd 20
  ```

`ceph config show` 커맨드에서는 덮어쓰여진 일시적인 옵션들이 함께 나온다는 것을 기억하세요.

### VIEWING RUNTIME SETTINGS
`ceph config show` 명령어로 실행 중인 데몬의 현재 설정을 볼 수 있습니다. 예를 들어:
```
ceph config show osd.0
```
는 해당 데몬의 기본값이 아닌 옵션들을 보여줄 것입니다. 특정한 옵션을 보려면,
```
ceph config show osd.0 debug_osd
```
혹은 기본값을 포함한 옵션들을 보려면:
```
ceph config show-with-defaults osd.0
```
어드민 소켓을 통해 로컬 호스트에서 실행중인 데몬에 접속하여 세팅을 보려면,
```
ceph daemon osd.0 config show
```
위 명령어는 모든 현재 세팅들을 덤프합니다.
```
ceph daemon osd.0 config diff
```
위 명령어는 기본값이 아닌 옵션을 보여줍니다. 그리고:
```
ceph daemon osd.0 config get debug_osd
```
는 특정 옵션의 값만을 보여줍니다.











.
