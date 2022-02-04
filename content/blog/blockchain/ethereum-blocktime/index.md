---
title: 블록타임이 트랜잭션 생성 시간보다 빠르다면?
date: "2022-02-04T09:44:41.901Z"
description: "블록에 담기는 트랜잭션인데, 어떻게 블록이 먼저 생성됬나요?"
category: blockchain
draft: false
---
## 들어가며


직접 생성한 트랜잭션과, 그 트랜잭션이 담긴 블록의 시간이 이상합니다.
- transaction sign & broadcast timestamp: `15:01:04`
- [etherscan.io](etherscan.io)에서 트랜잭션이 담긴 블록의 blocktime: `14:59:24`

<br/>

어떻게 트랜잭션보다 **블록이 먼저 생길 수 있을까요**?

## 문제 상황

로직을 설명하면 다음과 같습니다.

1. DB에 transfer에 대한 정보를 담습니다 (`15:00:40`)
2. (1)의 정보를 바탕으로 트랜잭션을 생성 & 전파합니다 (`15:01:04`)
3. (2)에서 생상한 트랜잭션이 블록에 포함됩니다 (`14:59:24`)

<br/>

검색을 통해 원인 파악이 불가능해서, Github Issue에 등록했습니다. 
- [block timestamp is before than my transaction create time](https://github.com/ethereum/go-ethereum/issues/24152)


## 원인은 무엇일까요?

고더리움 개발자님들이 답변해주신 내용을 요약해보자면

<br/>

먼저 블록 헤더가 만들어진 뒤(마이닝은 이때 시작됩니다), 트랜잭션이 추가됩니다.
- 만약 새로운 Transaction이 발견되면 해당 Transaction을 포함시킵니다. 
- 이때 `txRootHash` 변경됩니다. 그러나 `blocktime`은 변경되지 않습니다.
- 왜냐하면 이때, 이미 블록에 포함된 모든 트랜잭션을 한번 더 다시 실행하기 때문입니다 (EVM에서 `timestamp`를 참조한다)
- 따라서 채굴에 1분이 걸리는 블록은 채굴이 완료되기 1초 전에 만들어진 트랜잭션을 포함하면서도 `blocktime`은 채굴 시작 시점에 고정되어 있습니다.

<br/>

## 그렇다면 마이너들은 시간 낭비를 하고 있는게 아닌가요?

마이닝 과정을 생각해 봅시다.

1. 새로운 블록(nonce + blockhash)을 생성하기 위해 찾아야 하는 target nonce가 있다.
2. 그러나 블록 내용의 변경으로, 블록 해시가 변경되면 찾아야 하는 target nonce가 변경된다.
3. 즉, TX List가 변경되면 txRoot가 변경되고 blockhash도 변경되므로 마이너는 "새로운 nonce"를 찾아야 한다.

<br/>

이렇게 생각해본다면 결국 tx memPool에서 Block GasLimit까지 적당한 트랜잭션을 모두 고르고, 그 후 새로운 트랜잭션을 받지 않는게 현명하지 않을까요? 

<br/>

즉 **한번 마이닝을 시작하면 내용 변경을 하지 않는게 효율적이지 않나요?**

<br/>

위 절차는 모두 맞습니다. 하지만 'mental model'이 잘못됬습니다.
- 마이닝이란 집에서 양말찾기가 아니라, 넓은 호주 해변에서 특정한 **모래알 하나 찾기**와 같습니다.
- (양말을 찾는다면) 먼저 거실을 뒤지고 복도 등을 뒤지고 나면 결국 양말을 찾을 수 있을 것입니다. 잠시 후, 화장실만 남았습니다. **만약 당신이 바꿔서 갑자기 '야구공'을 찾기 시작한다면, 지금 사용한 시간은 '낭비'이며**, 다시 거실에서 시작해야 합니다.
- (마이닝은) 거의 무한에 가까운 공간이 있기 때문에 무작위로 해안을 뛰어다녀도 아무 상관없다. 심지어 어차피 같은 '모래알'을 찾는게 아닙니다.

## 마치며

더 자세한 이해는 해당 링크를 통해 가능합니다. 
- [block timestamp is before than my transaction create time](https://github.com/ethereum/go-ethereum/issues/24152)