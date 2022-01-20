---
title: Geth SIGKILL 비정상 종료
date: "2021-07-19T07:29:07.967Z"
description: "데이터 어디갔니?"
category: 'os'
draft: false
---

어느날 팀에서, Geth 업데이트 중 **블록데이터가 날라가는 일**이 발생했다.

```bash
Head state missing, repairing
```
- 재시작 후 만난 메세지

## 원인

지금까지 Geth를 Pm2라는 노드 패키지를 통해 관리했었다.

문제는 `pm2 stop <pid>` 명령어의 종료 방식이었다.

[Pm2 Document](https://pm2.keymetrics.io/docs/usage/signals-clean-restart/)를 확인해보면 

```
First a SIGINT a signal is sent to your processes, signal you can catch to know that your process is going to be stopped. 
If your application does not exit by itself before 1.6s (customizable) 
it will receive a SIGKILL signal to force the process exit.
```

간단히 말해 `pm2 stop`은 프로세스에 SIGINT 신호를 날리고, 1.6초 후 강제종료 신호인 `SIGKILL`를 날린다..

> SIGKILL은 잔인한 녀석이라 작업완료까지의 자비를 주지 않고 프로세스를 죽인다

여기서 문제가 발생했다. 

메모리나 디비가 비정상 종료되어 데이터가 많이 날라가버렸다. 
- 물런 항상 발생하는 문제는 아니다.

## 해결 방법 

일단 Geth의 **안전한 종료 방법**은 `SIGINT` 시그널이다. 

1. pm2 를 사용하지 않고 `kill -INT <pid>` 명령어를 통해 직업 종료하는 방법
2. pm2 `kill_timeout` 를 매우 길게 세팅하여 여유있게 종료하는 방법

## 추가 정보

`21.01.20` 추가 작성

원인은 사실, Config를 잘못 주입 후 실행해서 생긴 문제라고 추가 추측 (chainID 관련)

그럼에도 글 내용 자체는 의미가 있으니 삭제하지 않겠습니다.

## 참고
- [Is it safe to kill geth with SIGTERM?](https://ethereum.stackexchange.com/questions/10566/is-it-safe-to-kill-geth-with-sigterm)
- [geth rewinds chain on every startup](https://ethereum.stackexchange.com/questions/43231/geth-rewinds-chain-on-every-startup/45133)