---
title: RawTransaction은 어떻게 만들어질까?
date: "2021-11-29T16:16:39.869Z"
description: "정말 Nonce가 바뀌면 TXID도 바뀔까?"
category: blockchain
draft: false
---

`nonce`가 변하면 `txid`도 변한다는것은 많이 들어본 이야기다.

하지만 `SDK` 를 사용해 개발을 하다보니 어느새 정말 바뀌는지 혼란스러웠다.

`rawTransaction`이 어떻게 만들어지는지 알아보자

```javascript
import ethers from 'ethers.js'

async function sendTransaction(
    wallet: ethers.Wallet, 
    to: string, 
    value: ethers.BigNumber
  ): string {

    const gasPrice = await getGasPrice()
    const gasLimit = getGasLimit()
    
    const transactionRequest = { gasPrice, gasLimit, to, value }
    const fullTx = await wallet.populateTransaction(transactionRequest)
    const signed = await wallet.signTransaction(fullTx)

    const result = await provider.sendTransaction(signedTx)

    return result.hash
}
```

위는 `ethers.js` 모듈을 사용해서 `transaction`을 전송하는 과정이다. 

그런데 어디에도 `nonce`는 보이지 않는다. 정말 `txid`가 변할까?


## ethers.js

사실 `SDK` 내부 구현에서 대부분의 작업을 처리해주고 있기 때문에 혼란스러움을 느낀 것이다.

<br/>

"gasPrice, nonce, gasLimit, chainId를 채우고 반환한다." 
> from [populateTransaction](https://docs.ethers.io/v5/single-page/#/v5/api/signer/-%23-Signer-populateTransaction) 문서

<br/>

즉, (객체 초기화 당시) 설정한 provider에게 `TransactionCount`를 조회 한다. 명시적으로 입력하지 않은 `nonce`가 해당 메소드를 통해 채워지는 것이다.



```javascript
// ethers.js populateTransaction method
if (tx.nonce == null) {
  tx.nonce = this.getTransactionCount("pending") 
}
```
> from [ethers.js github](https://github.com/ethers-io/ethers.js/blob/master/packages/abstract-signer/src.ts/index.ts#L293)

코드에서 account의 `transactionCount`를 조회하는것을 확인 가능하다.

<br/>

정리하자면, `nonce`를 명시적으로 입력하지 않았지만 `sdk` 메소드를 통해 서명 전 필요한 값을 모두 채웠다.

<br/>

'복잡성'을 라이브러리 속으로 감추고, 로직을 구현했기 때문에 혼란이 발생했다.


## 그렇다면 서명전에 필요한 값은 무엇일까?

> 지금부터 나오는 내용은 "최신 정보"가 아니다. 예를들어 `EIP-1559` 변경사항은 포함되어 있지 않다.

<br/>

<마스터링 이더리움> 6장을 일부 수정한 내용이다.

<br/>

필요한 정보는 아래와 같다.
- nonce
- gas price
- gas limit
- recipient (to address)
- value (payment)
- data (contract invocation)
- v, r, s (EOA의 ECDSA 디지털 서명의 세가지 구성 요소)

<br/>

> from address가 존재하지 않는 이유는, EOA의 공개키를 v, r, s 구성요소로 부터 알아낼 수 있기 때문이다. 또한 공개키 복구 과정은 설명하지 않는다. 

> v 값에는 chainID 값이 포함된다.

## 그렇다면 어떻게 rawTransaction이 만들어지는걸까?


"트랜잭션은, 필요한 데이터를 포함하는 RLP 인코딩 체계를 사용하여 시리얼라이즈된 바이너리 메세지다."

1. `[nonce, gasPrice, gasLimit, to, value, data, chainID, 0, 0]`의 9개 필드를 포함하는 트랜잭션 데이터 구조 생성
2. RLP로 인코딩된 트랜잭션 데이터 구조의 시리얼라이즈된 메시지를 생성
3. 시리얼라이즈된 메시지의 Keccak-256 해시를 계산
4. EOA의 개인키로 해시에 서명하여 ECDSA 서명을 계산
5. ECDSA 서명의 계산된 v, r, s 값을 트랜잭션에 추가
6. 서명이 추가된 트랜잭션 데이터를 RLP 인코딩해 시리얼라이즈하여 전파

<br/>

위 내용을 좀 더 자세히 설명해보겠다. (From [KlaytnDocs](https://ko.docs.klaytn.com/klaytn/design/transactions/basic#rlp-encoding-example))


아래는 1번 ~ 5번까지의 내용이다.

```javascript
const message = rlp([
  nonce, gasPrice, gas, to, value, input, chainid, 0, 0
])
const hash = keccak256(message)
const signature = sign(hash, <private key>)
```

<br/>

아래는 6번의 내용이다.

```javascript
const rawTransaction = rlp([
  nonce, gasPrice, gas, to, value, input, v, r, s
])
const txHash = keccak256(rawTransaction)
```

<br/>

실제로 `caver.js`를 이용해서 txHash를 뽑는 코드
- 위와 동일하게 `rawTransaction` 값의 `keccak256` 해시값을 계산한다.
```javascript
const txhash = caver.utils.keccak256(tx.getRLPEncoding())
```

<br/>

그렇다면 `rawTransaction`을 왜 본 기억이 없을까? (정확히 말하면 자주 보진 않는다.)

<br/>

간단하다. 보통 보기 쉽게 출력해주기 때문이다. (Node의 특정 API, 라이브러리 등)

심지어 `RPC` 콜을 할때도 `rawTransaction`만을 조회하는 API는 별도로 존재한다.

<br/>

예를들어 일반 사용자가 자주 접하는 메타마스크나 또는 이더스캔은 사용자가 보기 쉽게 정보를 재가공해서 보여준다.

또한 Blockchain Client 없이 `https://etherscan.io/getRawTx?tx={txhash}` 주소로 rawTransaction을 쉽게 확인가능하다.

## 마무리

> 이더리움은 글로벌 싱글톤 상태 머신이며, 트랜잭션은 이 상태 머신을 움직여서 상태를 변경할 수 있도록 만든다.

<br/>

이렇게 만들어진 트랜잭션을 네트워크에 전파하고, 블럭에 포함되면 결국 이더리움 싱글톤 상태가 수정된다.

<br/>

얼마전 "트랜잭션 서명의 검증 과정"에서 문제가 발생하여 클레이튼 네트워크가 36시간 동안 장애가 생겼엇다. 

해당 이슈를 첨부하고 글을 마무리하겟다. [Klaytn 메인넷(Cypress)의 11/13 블록 생성 장애 복구 과정 및 원인 분석](https://medium.com/klaytn/incident-report-klaytn-%EB%A9%94%EC%9D%B8%EB%84%B7-cypress-%EC%9D%98-11-13-%EB%B8%94%EB%A1%9D-%EC%83%9D%EC%84%B1-%EC%9E%A5%EC%95%A0-%EB%B3%B5%EA%B5%AC-%EA%B3%BC%EC%A0%95-%EB%B0%8F-%EC%9B%90%EC%9D%B8-%EB%B6%84%EC%84%9D-c8760ed30a05)

## 출처 및 참고

- <마스터링 이더리움>
- [클레이튼 일반 트랜잭션 문서](https://ko.docs.klaytn.com/klaytn/design/transactions/basic#rlp-encoding-example)
- [트랜잭션 해시(txid)에 대한 오해](https://brunch.co.kr/@nujabes403/15)
- [이더리움 트랜잭션 분석](https://medium.com/santonychoi/%EC%9D%B4%EB%8D%94%EB%A6%AC%EC%9B%80%EC%9D%98-%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98-%EB%B6%84%EC%84%9D-part-2-6c2751d5775)

