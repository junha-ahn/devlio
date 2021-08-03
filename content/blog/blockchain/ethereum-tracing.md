---
title: 이더리움 Trace에 대해 알아보자
date: "2020-11-12T15:32:58.145Z"
description: "스마트컨트랙트를 통한 ETH Transfer를 확인하기 위해서는 어떻게 해야할까?"
category: blockchain
draft: false
---

Geth Tracing Document [링크](https://geth.ethereum.org/docs/dapp/tracing)


먼저, 공부를 하는 입장에서 오류가 있을 수 있음을 양해 부탁드립니다.

### 요약

스마트컨트랙트를 통한 ETH Transfer를 명확히 확인하기 위해서는 `Tracing`이 필요합니다

이는 이더리움 노드가 해당 Transaction을 재실행 후, 결과를 다시 확인하는 과정입니다

<br>

### 왜 필요할까?

먼저 보통의 ETH 이동 Transaction, 즉 `value`값에 `amount`를 넣어 전송하는 트랜잭션에 대해서는 이 글에서 설명하지 않겠습니다

이더리움은 SmartContract를 이용하여 ETH를 전송할 수 있습니다.

```solidity
address.transfer(amount)
```

그러나 이러한 ETH 이동의 결과는 `Transaction Receipt` 에 포함되지 않습니다.

<br>

즉, 어떤일이 일어났는지 알기 매우 힘듭니다.

참고로 ERC20 또한 SmartConract를 통하여 token을 이동시킵니다. (물런 ETH를 이동시키는게 아닙니다)

ERC20 표준을 살펴보면 아래와 같은 이벤트를 볼 수 있습니다.

```solidity
event Transfer(address indexed _from, address indexed _to, uint256 _value);
```

우리는 이 event를 통해, `Transaction Receipt`에 남은 logs를 확인하고 자산이동을 확인 가능합니다.
 
<br>

그렇다면 아래와 같은 경우에는 어떻게, ETH를 받았는지 확인 할 수 있을까요?

1. Smart Contract를 이용한 Transfer
2. 어떠한 Log도 남지 않은 경우

물런, 계정의 총 자산을 확인 하면 가능하지만... transaction 내역이 많아 힘들다고 가정하겠습니다.

결론은 해당 트랜잭션을 이더리움 노드가 다시 실행하고 EVM의 결과를 확인하면 됩니다.

그것이 `Tracing` 입니다.

<br>

### 필요사항

Tracing을 위해서는, 이더리움 노드가 재실행을 위한 데이터를 보유하고 있어야 하는데, 원문 중 하나를 참조하면,

`An archive node retaining all historical data can trace arbitrary transactions at any point in time. Tracing a single transaction also entails reexecuting all preceding transactions in the same block.`

geth 이더리움 node가 archive 모드로 동기화되었다면, 어떠한 트랜잭션이든 `Tracing`이 가능다라는 의미입니다.