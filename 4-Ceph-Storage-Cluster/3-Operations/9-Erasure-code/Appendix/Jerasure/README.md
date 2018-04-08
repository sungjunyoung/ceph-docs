> 참고자료들
> 1. Jerasure documenation: http://jerasure.org/jerasure-2.0/
> 2. Optimizing Cauchy Reed-Solomon Codes for Fault-Tolerant Storage Application: http://web.eecs.utk.edu/~plank/plank/papers/CS-05-569.pdf

# 1. Jerasure

> 참고: http://jerasure.org/jerasure-2.0/

## INTRODUCTION

- modular, fast and flexible
- support a horizontal mode of erasure code
- ec 가 Maximum Distance Separable (MDS) 라면 m 개의 손실에도 전체 시스템이 동작 가능하다.
- w 파라미터 (word size) : 각 장치가 안의 w 만큼의 데이터를 가짐

![jerasure-intro](/Images/jerasure-intro.png)

## MODULES
- 모듈 사용 시 jerasure + galois 는 필수, 나머지 하나를 조합하여 사용함
- jerasure 모듈은 galois 디펜던시 외에 없음

### jerasure.h/jerasure.c
- matrix-based encoding/decoding 지원
- bit-matrix-based encoding/decoding 지원
- bit-matrix -> schedule 변환
- matrix <-> bit-matrix 변형

### galois.h/galois.c
- Galois Field Arithmetic 구현체

### reedsol.h/reesol.c
- Reed-Solomon coding 에 따라 분산 matrix 를 만듬

### crs.h/crs.c
- Cauchy Reed-Solomon 코딩 구현체
- 전통적인 Reed-Solomon matrix 와 다른 형태의 matrix 를 만든다.
- Jerasure 에 RAID-6 에 최적의 Cauchy distribution matrix 를 만들고, 더 우수한 distribution matrix 를 생성

## MATRIX-BASED CODING IN GENERAL

![matrix-vector-product](/Images/matrix-vector-product.png)

상단의 K * K matrix
하단의 M * K matrix : coding matrix

- 두개의 matrix 가 데이터와 매트릭스곱 연산을 거쳐 최종 K/M 을 만들어냄
- 최종 K/M 결과에서 최대 M개의 에러가 났을 때, distribution matrix 의 역행렬을 곱하면 원본 데이터를 복구 할 수 있음

## BIT-MATRIX CODING IN GENERAL
- Cauchy Reed-Solomon 코딩 논문에서 처음 소개되었음
- bit-matrix 를 이용해 인코딩/디코딩을 하기 위해서는 기존 distribution matrix 를 확장해야함 (binary distribution matrix (BDM))

![bit-matrix-coding](/Images/bit-matrix-coding.png)

---

# Optimizing Cauchy Reed-Solomon Codes for Fault-Tolerant Storage Applications
> 참고:  http://web.eecs.utk.edu/~plank/plank/papers/CS-05-569.pdf

## THE CURRENT STATE OF THE ART
- MDS (Maximum Distance Separable) code
- k = 1 이라고 할때, MDS 코드는 단일 코딩 블록 당 k - 1 의 연산을 수행한다.
- 이 연산들은 XOR 연산 (fast)으로 이루어지고, 두번째 연산은 XOR 연산보다 비싼 Galois Field multiplication 으로 이루어진다.
- m = 1 인 RAID 5 에서, MDS 코드가 최적이고, 디스크 어레이 시스템에서 메인으로 쓰여진다. m = 2, m = 3 일때 다른 코딩 방식들이 나타남. (EVENODD, STAR) 하지만 최적은 아니엿음
- 1999 년에, m = 2 에서 k + 2 연산을 수행하는 X-Code 가 나왔다.
- 이 코드들과는 별개로, MDS 코드들은 Reed-Solomon 코드이고, k / m 이 몇이든 별개든 좋은 성능을 보여줌
- 그러나, Reed-Solomon 을 사용하는 코드들은 코딩 블록 당 n 개의 Galois Field 곱셈을 요구하는 단점을 가지고 있으며, 코딩 블록은 일반적으로 기계의 워드 크기보다 작기 때문에, 기계 워드 당 2k 에서 8k 의 곱셈을 요구함. (연산이 비싸다)
- 그래서 나온 것이 Cauchy Reed-Solomon (CRS) 코딩
  - 첫째로, 모든 인코딩 연산을 XOR 로 바꾸었고, 인코딩이 코딩 블록당 O(nlog2(k+n)) 로 최적화 되었음
  - 둘째로 Cauchy distribution matrix 를 사용함으로서 디코딩을 위한 matrix inversion (역행렬) 연산이 최적화됨
  - 이제는 CRS 코딩이 일반적인 MDS erasure coding 으로 자리잡음
- MSD 코드가 아닌것들에는 HoVer, WEAVER 코드가 있음








.
