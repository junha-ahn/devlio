---
title: 자바스크립트에서 큰 숫자 처리하기 (실제 입금처리 문제사례)
date: "2023-12-17T06:43:51.660Z"
description: "Bignumber.js 쓰세요. 제발"
category: nodejs
draft: true
---

실제 업무 중 만났던 두 가지 케이스에 대해 간단하게 작성 해본다.

## 1. `1,999,999.999999999983222784` 

KLAY 입금을 처리 하는 중 `1,999,999.999999999983222784 KLAY` 입금 트랜잭션이 여러 Javascript 로직을 타면서 `2,000,000 KLAY` 입금으로 처리되어버렸다.

예를들면 아래와 같은 로직의 문제였을 것이다. 
```javascript
Math.floor("1999999.999999999983222784") // 2,000,000
```

## 2. `279,223,813,037,930,000`

> [Number.MAX_SAFE_INTEGER](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/MAX_SAFE_INTEGER) 범위를 넘는다. 

해당 입금건도 실제와 다른 값(`279,223,813,037,929,984`)이 DB에 저장되었다. 


문제 코드는 마지막 DB Insert 부분에 Javascript Number를 사용했기 때문에 발생했다.

```javascript
// import 'sequelize'

const bigAmount = new BigNumber(tx.amount)
  .multipliedBy(scaleFactor)
  .integerValue(BigNumber.ROUND_DOWN);
await transferHistory.create({
  ...
  amount: bigAmount.toNumber(),
})
```

## 결론 

Javascript로 큰 수를 다룰때는 절대 기본 Number 자료형을 사용하지 말자. 안전하게 String 형식으로 숫자 데이터를 처리하자. (`bignumber.js` 등의 라이브러리)

Bignumber.js를 사용하면서도 위 사례와 같이 일부 연산 후 Number 자료형을 사용할 수 있는데, 처음부터 끝까지 놓치는 부분 없이 String을 사용해야 한다.

그리고 소수점의 경우 시스템의 최소자리수를 정하고 무조건 내림처리한다. (입금의 경우 반올림, 올림을 사용할 수 없다. 즉 없는 돈을 만들어 줄 수는 없다)
