---
title: Geth는 어떻게 동기화(Sync)할까?
date: "2021-07-21T08:35:28.062Z"
description: "Geth Sync Mode를 비교해보자"
category: blockchain
draft: false
---


이 문서는 Geth의 동기화 과정의 차이점을 이해하는 것을 목적으로 합니다.

먼저 문서에 오류가 있을 수 있음을 밝힙니다. 수정 요청  부탁드립니다! 


## 노드란?

[ethereum.org](https://ethereum.org/en/developers/docs/nodes-and-clients/)에서는 노드를 아래와 같이 설명합니다.

```
노드란 "실행 중"인 '클라이언트 소프트웨어'

클라이언트는 각 블록의 모든 트랜잭션을 확인하여, 
네트워크와 데이터를 정확하게 유지하는 Ethereum의 구현체
```

![node](./node.png) 



## Node의 타입

먼저 노드(실행중인 클라이언트)는 크게 3가지 타입으로 구분됩니다.

- Full node: 전체 블록체인 데이터를 저장하고, 블록 검증에 참여하고 모든 블록과 상태를 확인합니다.
- Archive node: Full Node가 저장 중인 것을 전부 저장하고, 과거 State의 아카이브 저장
- Light node: 블록 헤더를 저장하고 다른 모든것을 요청합니다 (설명하지 않겠습니다)

<br/>
Archive 이외의 노드는 동기화하면서 State 데이터가 정리(Pruning)됩니다.

다시말해, Archive node와 Full Node의 차이점은 과거 State 저장 여부입니다. 아래 사진을 통해 차이점을 명확히 확인 가능합니다.

![archive-vs-full](./archive-vs-full.png) 


<br>

이더리움 State에 대해서 궁금하다면
- [이더리움 상태(State)란 무엇일까?](/blockchain/ethereum-state/) 
<br/>

아래는 실제 이더리움 클라이언트의 구현체 목록입니다.  

![clients](./clients.png) 


Sync strategies, State Pruning라는 새로운 단어가 등장합니다. 이 단어들을 [Geth](https://github.com/ethereum/go-ethereum)의 동기화 과정을 통해 알아보겠습니다.

## Geth의 Sync Strategies(동기화 전략)

[Geth Command Line Option](https://geth.ethereum.org/docs/interface/command-line-options) 
```
--syncmode # ("fast", "full", "snap" or "light") (default: snap)
```

Geth Command Line Option Geth의 Sync Strategies는 `Fast` 와 `Full` 로 나눠지는 것을 알 수 있습니다. (그외에도 Snap, Light 동기화 전략이 있습니다) 

<br/>

Full Sync Mode는 모든 블록 데이터(headers, transactions, receipts)를 다운받아 모든 트랜잭션을 재실행하며, 검증하고 직접 모든 블럭의 State를 만들어 나가는 전략입니다. 하지만 제네시스 블록부터 모든 트랜잭션을 재실행해나가는 것은 엄청나게 오랜 시간이 소요됩니다. (현재 매우 강력한 시스템에 7~8일이 소요됩니다)

<br/>

Fast Sync Mode는 Full Sync처럼 모든 블록 데이터(headers, transactions, receipts)를 다운받습니다. 하지만 트랜잭션을 직접 실행하지 않고 PoW 검증(블록 헤더 검증)을 할 뿐입니다. 

트랜잭션을 직접 실행하지 않으면 어떻게 State를 만들까요? 답은 간단합니다. 

최신 State(Head - 64, `pivot point` )를 다른 노드에게 직접 다운로드 받습니다  (동기화 과정을, 다른 노드에 거의 의존하지 않는 Full Sync에 비해 불안전합니다)

<br/>

한번 동기화에 성공하면 Full Sync Mode로 전환됩니다. (즉 동기화 성공 후, 트랜잭션을 직접 실행하며 State를 만들어 나갑니다) 즉 Sync Mode란 ‘실행’에 관한 전략이 아닙니다. 말 그대로 ‘동기화’에 대한 전략입니다 (어떻게 안정적으로, 빠르게 이더리움 네트워크에 참여할 것인지 동기화 전략을 선택할 수 있습니다)

### Q. 최신 블록에 (임의의 개수)64개 뒤쳐진 상태로 동기화가 진행되지 않습니다.

- 블록데이터를 모두 다운받았기 때문에 동기화 완료되었다는 생각은 **착각**입니다. 아직 동기화 진행중입니다.
- 아직 State Trie 다운로드가 완료되지 않았습니다. (이 작업이 블록 데이터 다운로드보다 더 오랜 시간이 소요됩니다.)
- 매 블록마다 지속적으로 변경되는 state trie를 지속적으로 동기화해야 합니다. (심지어 다운로드하는중에 필요없어지기도 합니다) 끝없는 `Imported state entries [...]` 로그를 통해 확인 가능합니다.
- 엄청난 양의 Disk Write 작업이기 때문에 성능이 좋은 SSD를 사용해야 합니다. (HDD라면... 😢)


### Geth의 State 정리(Pruning) 전략

[Geth Command Line Option](https://geth.ethereum.org/docs/interface/command-line-options) 
```bash
--gcmode # garbage collection mode 
         # ("full", "archive") (default: "full")
```

Pruning Mode는 Pruning을 하는 전략이고, Archive Mode는 반대로 Pruning을 하지 않는 전략입니다.

Pruning이란 쉽게 말해 이더리움 State Trie의 `Garbage collection`입니다. (상태 정리)

예를들어 특정 Account의 “1년전” Balance가 필요한 경우가 있을까요? 대부분 그렇지 않습니다. 

이렇게 필요가 없어진 State Trie를 정리하는 작업이 Pruning입니다. (Trade Off)

사실 Pruning을 진행했어도 Full Sync Mode라면, 블록 데이터를 통해 과거 상태를 다시 만들 수 있습니다. 다만 매우 힘든 작업이 될 것입니다 (아래에서 설명합니다)

Pruning이 어려운 작업인 이유
- 블록 재구성(더 높은 누적난이도를 가지는 체인이, 낮은 체인을 덮어버리는 현상)에 대비해 N개의 State를 메모리에 보유하고 있어야합니다. 
- 이론적으로는 블록 재구성의 위험을 감수하지 않을 만큼 오래된 상태 데이터를 쉽게 삭제할 수 있지만 결과적으로 어렵습니다.
-  오래된 Tree의 Root가 오래되어 삭제할 수 있는지 여부를 쉽게 결정할 수 있지만, 이전 상태의 깊은 단말(leaf) 노드가 여전히 새로운 항목에서 참조되는지 여부를 파악하는 것은 비용이 많이 듭니다.

> 왜 `—-gcmode=full`이 Pruning을 진행하는 옵션일까요?
> 개인적으로 생각하기엔 full은 모든걸 저장할 것 같은 이름인데 말이죠... GC를 Full로 돌린다는 의미일라나요...
    

## 다시 노드의 타입으로

앞서 말했듯 노드(실행중인 클라이언트)는 크게 3가지 타입으로 구분된다고 했습니다. (Full Node, Light Node, Archive Node) 이제 우리는 어떤 옵션으로 Geth를 실행시켜야 Full Node로, 또는 Archive Node로 만들 수 있는지 알게 되었습니다.



### Q. [8,000,000  블록부터 archive mode를 실행하고자 한다면 어떻게 해야하나요?](https://github.com/ethereum/go-ethereum/issues/24063)
- `syncmode=full gcmode=full` 으로 genesis 블록 부터 실행하고 8만번째 블록에 도달하면 노드를 멈추고 `gcmode=archive` 로 변경하여 실행하면 됩니다.
- 이렇게 하면 8M부터는 모든 State를 삭제하지 않고, 저장하게 됩니다.
- 다만 이렇게 임의로 멈추는것은 이론적으로만 가능하니까 Geth Code를 수정하여 8M 블록에서 정지하도록 해야합니다.

### Q. [`gcmode=full` 로 실행했지만 archive Node로 전환하고 싶습니다. 즉, 전체 State를 다운로드 하고 싶습니다](https://github.com/ethereum/go-ethereum/issues/24115)

- 과거 State에 즉시 접근(immediately accessible)하기 위해서 `gcmode=archive` 로 실행했어야 합니다. (gcmode default value는 “full”)
- `syncmode=full, gcmode=full` 로 실행했다면, Earlier Point부터 다시 계산하여 필요한 State를 만들 수 있습니다. 
- Geth는 약 2시간 마다 State를 flush() 합니다. (reexec parameter)
- 다만 fast/snap Sync Mode로 실행했다면 Pivot Point 이전의 블록에 대한 상태를 얻을 수 없습니다.
- Geth 종료 후 `gcmode=archive` 를 통해 다시 실행했다면, 그 이후의 상태 데이터는 메모리 상에서 제거되는게 아니라, disk에 flush()됩니다

### Q. `NoPruning`  옵션과 `gcmode` 옵션의 차이점이 무엇일까요?

- 차이가 없습니다. 둘다 똑같이 Pruning에 관한 설정입니다.
    - Archive Mode 옵션: NoPruning = True, gcmode = archive
    - Pruning Mode 옵션: NoPruning = False, gcmode = full
- 여담이지만, 전 `NoPruning` 이라는 이름이 너무 헷갈립니다.

----

<br/>

[이더리움 상태(State)란 무엇일까?](/blockchain/ethereum-state/) 추천합니다.

모든 내용은 요약되었기 때문에, 아래 링크를 통해 자세한 내용을 읽어보는 것을 추천합니다. 

### 출처

- [Nodes And Clients](http://ethereum.org)
- [Geth FAQ](https://geth.ethereum.org/docs/faq)
- [Ethereum full node vs archive node](https://www.quicknode.com/guides/infrastructure/ethereum-full-node-vs-archive-node)
- [What Comprises an Ethereum Fullnode Implementation?](https://medium.com/amentum/what-comprises-an-ethereum-fullnode-implementation-a9113ce3fe3a)
- [fast synchronization algorithm](https://github.com/ethereum/go-ethereum/pull/1889)

### 추가학습

- [Running Ethereum Full Node on a RaspberryPi 4](https://kauri.io/#communities/Ethereum%20Node%20Runners/running-an-ethereum-full-node-on-a-raspberrypi-4-/)