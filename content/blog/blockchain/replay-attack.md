---
title: 리플레이 어택이란?
date: "2020-11-16T17:07:21.849Z"
description: "내 코인이 사라질 수 있다?"
category: blockchain
draft: false
---

## 요약

리플레이 어택이란, 하드포크되면서 발생할 수 있는 일종의 이중지불 공격입니다.


1. 하드포크 전 1000 ETH를 보유한 A가 있다
2. A는 이더리움이 ETC와 ETH로 포크 된 후
3. 새로 생긴 1000 ETC를 B에게 ETH의 10분의 1 가격으로 팔았다. 
4. 그런데, 얼마 후 ETH가 모두 사라졌다

[해시넷](http://wiki.hash.kr/index.php/리플레이_공격)에 적혀있는 시나리오입니다. 

그런데 실제로 이런일이 일어납니다.

## 어떻게 가능한가?


`A chain`에 특정시점에 하드포크가 일어난 `B chain`이 있다고 가정합니다.

하드포크 이후 `B chain`에서 발생한 트랜잭션의 정보가 다음과 같다고 가정합니다

```javascript
// Transaction (in B chain)
{
  from: 'add1',
  to: 'add2',
  amount: '1',
}
```

이때, 위 트랜잭션은 `A chain`에서 똑같이 실행할 수 있습니다.

두 체인은 모두 특정 블럭 이후에 갈라졌기에
잔고(or `UTXO`), 거래내역(or `nonce`), 개인키등이 모두 같습니다.

하나의 체인에서 일어난 `Transaction`을 그래도 정확히 리플레이(재사용) 할 수 있습니다.


간단히 요약하면, 위 예시 트랜잭션에 `RawTransaction`이 `abc...`라면 해당 값을 구한 후 그대로 복사해 `A network`에 전파하면 됩니다.

이렇게 됬을때, 다른 마이너는 복사한 트랜잭션을 '정상' 트랜잭션으로 인식하고 정상적으로 블록에 담기고 `confirm`됩니다

중요한것은 동일한 거래가 양체인에서 발생하는것입니다. `toAddress`는 달라질 수 없습니다

\* RawTransaction은 간략화하면, 트랜잭션의 여러 정보를 섞어, PrivateKey로 서명한 결과물입니다.

## 해커의 주소로 잔고를 송금 할 수 있나?

만약 공격자가 해당 트랜잭션의 `toAddress`를 다른 주소로 변경하는것은 불가능합니다.

왜냐하면, `A chain`에서 발생한 `Transaction`과 다르기때문에, 새로운 `RawTransaction`을 만들어야 하기때문입니다. (즉 다시 서명을 해야합니다)

`PrivateKey`는 공격자가 알지못합니다. (새로운 트랜잭션은 만들 수 없습니다)


## 해결법

하드포크시, 프로토콜이 변경되면 간단하게 해결 가능합니다.

`RawTransaction`을 만드는 과정에 새로운 `Marker`를 추가하면 됩니다.

예를들어 위 예제에서, 리플레이 공격이 성공할 수 있는 이유는 두 네트워크의 트랜잭션 직렬화(or `Parse`) 과정이 같기때문입니다. 

만약 `RawTransaction`을 만들때 필요한 데이터가 아래와 같이 변경된다면, 

더 이상 양쪽 체인에서 리플레이 공격은 성립하지 못합니다. 
- `A chain`에서 발생한 트랜잭션을 `B network`로 옮기면, `Marker`가 Tx에 없어서  `Invalid`
- 그 반대로, `A chain`에서는 `Marker`가 녹아있는 TX `Invalid`

이를 `two-way replay protection`라고 합니다.

```javascript
// Transaction (in B chain)
{
  Marker: 'B',
  from: 'add1',
  to: 'add2',
  amount: '1',
}
```

이렇게 하드포크시, 프로토콜의 변경이 있을 경우 간단합니다. (사용자가 처리할 것이 없습니다)

하지만, 만약 지원되지 않는다면 

```
그런 경우가 있습니다.
예를들어, 서로 양측 체인이 '원조'라고 주장하며
이러한 해결책을 지원하지 않는다면 사용자는 어려움을 겪습니다
```
이럴때, 다음과 같이 생각해 볼 수 있습니다.

1. 안정될때까지, `Transaction`을 만들지 않는다. 리플레이 할것이 없습니다.
2. 각체인에 임시 주소를 만들고, 자산을 전송한 후, 다시 원래 주소로 되돌린 다음 사용하는 방법
   - 정확히는, 두 체인에서 조금 다른 결과가 만들어져야합니다.
   - 두 체인에서 임시 주소 전송/반환의 결과가 다르다면(리플레이 공격이 불가능해진다면), 사용 가능합니다.



\* 추가로, 한쪽에서 `Transaction`을 `broadcast`하면 반대 체인에서도 블록에 담길 수 있기에 주의가 필요합니다

각 체인은 블록을 쌓을때 사용하는 합의의 차이가 있는것이지, 트랜잭션 검증이 달라진것은 아니기 때문입니다


## 참고

- [리플레이 공격 - 해시넷](http://wiki.hash.kr/index.php/리플레이_공격)

- [Life after Hard Forks: What You Need to Know About Replay Protection](https://www.sfox.com/blog/life-after-hard-forks-what-you-need-to-know-about-replay-protection/)
