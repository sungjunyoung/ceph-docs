> 참고자료들
> Shingled Erasure Code (SHEC) slideshare: https://www.slideshare.net/miyamae1994/shec-hotdep14-slides021slideshare

# 1. Shingled Erasure Code (SHEC) slideshare
- Reed Solomon code 는 recovery-efficient 하지 않다.
- 데이터 복구는 가능한 빨라야 한다. - multiple disk failure / data loss 를 피하기 위해
- Reed Solomon code 는 local parity methods 방법으로 향상되었다.
  - 그래서 ? 복구하는 동안 read 작업이 줄어들어야한다.

![reed-solomon-local-parity-methods](/Images/reed-solomon-local-parity-methods.png)

- local parity method for multiple disk failure
  - single disk failure 에 최적화 되어있다.
  - 하지만, multiple disk failure 에서는, recovery overhead 가 매우 크다

- SHEC (= Shingled Erasure Code)
  - local parity group 들로만 이루어진 erasure codes (multiple disk failure 에 더 효율적)
  - local parity 들이 마치 지붕 위의 기왓장처럼 부분적으로 겹쳐지고, shift 되어진다.

![local-parity](/Images/local-parity.png)

- 공간 효율, 복구 효율, Durability 세가지 요소에 대해 trade-off
- 복구 효율이 SHEC 의 가장 큰 특징

![shec-recovery-efficiency](/Images/shec-recovery-efficiency.png)

- multiple disk failure 에 대해서 다른 방법에 비해 더 효율적이다.
- Durability Estimator (=ml/k)
  - 얼마나 많은 디스크가 fail 될 수 있는지 지정
  - m * l(?) / k 로 계산되며, 이 그 이상의 디스크가 fail 일 시에는 데이터 손실을 일으킨다.
- Single disk failure 에서의 성능은 비슷
- 애초에 Ceph 의 Erasure code plugin 으로서 만들어졌음

![shec-ceph](/Images/shec-ceph.png)
