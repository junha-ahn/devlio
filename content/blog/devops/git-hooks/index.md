---
title: pre-commit, husky 없이 Git Hooks 사용하기
date: "2023-07-24T16:05:50.831Z"
description: "아무것도 설치하기 싫다고..."
category: devops
draft: false
---

# 목차

1. 그냥 설치하기 싫었다
2. 메세지 컨벤션 체크하기 (prepare-commit-msg)
3. IDE에서 자동으로 세팅하기 (Devconainer, Gradle)
4. git actions에서도 사용하기 
5. Lint 해보기 (pre-commit)

# 그냥 설치하기 싫었다

Git hooks는 사용하고 싶고, `pre-commit`이나 `husky`는 **설치하고 싶지 않았다**

어떻게 해야할까?

# 커밋 메세지 컨벤션 체크하기

prepare-commit-msg hook을 사용해 커밋 메세지 컨벤션을 체크한다

```shell
#!/bin/sh
commit_msg_file="$1"
commit_msg_type="$2"
commit_msg="$3"

if [ "$commit_msg_type" = "-m" ]; then
  echo "$commit_msg" > "$commit_msg_file"
fi

commit_msg=$(cat "$commit_msg_file")

# 정규표현식 파일을 참조한다
commit_msg_regex=$(cat "$(dirname "$0")/../../.github/commit-regular.txt")

if ! echo "$commit_msg" | grep -Eq "$commit_msg_regex"; then
  echo "Invalid commit message format."
  exit 1
fi
```
> [prepare-commit-msg](https://github.com/f-lab-clone/ticketing-service/blob/main/.github/hooks/prepare-commit-msg)

<br/>

간단히 말해 해당 파일을 `.git/hooks/prepare-commit-msg` 위치에 작성하면, Git이 알아서 `git commit` 마다 **컨벤션 체크가 동작**한다!


```javascript
(feat|fix|refactor|style|docs|test|chore):.{1,50}(\n.{1,72})?$
```
> [commit-regular.txt](https://github.com/f-lab-clone/ticketing-service/blob/main/.github/commit-regular.txt)

<br/>

위 파일은 정규표현식으로 향후 커밋 메세지를 체크할때 사용한다 
- 어떤 의미인지는 해당 [템플릿](https://github.com/f-lab-clone/ticketing-service/blob/main/commit-msg-template.txt)을 통해 확인 가능하다
- 이렇게 별도 파일로 구성한 이유는 해당 정규표현식을 `Git Actions`를 통한 CI 작업에도 사용하기 때문이다


# IDE에서 자동으로 세팅하기 (Devconainer, Gradle)

해당 파일을 모든 개발자가 셋업 하기에는 너무 귀찮은 일이다.

특정 IDE 마다 다르겠지만 대부분 훅을 제공해 여러 설정을 자동으로 세팅할 수 있다.


## Devcontainer

`devcontainer`는 `Codespace` 개발환경 세팅을 위한 규칙이라고 생각하면 편하다!

가장 간단한 예시로는 레퍼지토리에 `devcontainer` 규칙에 맞는 파일을 정의하면, 어떤 누구나 코드스페이스를 실행할때 설정해놓은 Ubuntu Image 위에 Java, Docker가 설치되며 Extenstion도 자동으로 설치된다!

```json
{
    // ...
   "postCreateCommand": "bash ./.devcontainer/postCreateCommand.sh",
}
```
> [devcontainer.json](https://github.com/f-lab-clone/ticketing-service/blob/main/.devcontainer/devcontainer.json)

```shell
cp .github/hooks/prepare-commit-msg .git/hooks/prepare-commit-msg
chmod +x .git/hooks/prepare-commit-msg
```
> [postCreateCommand.sh](https://github.com/f-lab-clone/ticketing-service/blob/main/.devcontainer/postCreateCommand.sh)

<br/>

`postCreateCommand.sh` 을 이용해 쉘 스크립트 파일을 `.git/hooks`로 복사한다

## Gradle

인텔리제이를 포함해서 모든 에디터에서 공통이다.

```java
val installLocalGitHook = tasks.register<Copy>("installLocalGitHook") {
    from("${rootProject.rootDir}/.github/hooks")
    into(File("${rootProject.rootDir}/.git/hooks"))

    eachFile {
        mode = "755".toInt(radix = 8)
    }
}

tasks.build {
    dependsOn(installLocalGitHook)
}
```
> [build.gradle.kts](https://github.com/f-lab-clone/ticketing-service/blob/main/build.gradle.kts)

<br/>

해당 파일은 코틀린으로 작성되었으며 특정 Gradle 버전마다 다른 형식을 사용할 수 있다

단점: `graldew build` 실행시 세팅되기 때문에 **만약 빌드를 안하면 세팅되지 않는다!**

# Git Actions에서도 사용하기

위 커밋 컨벤션 체크 쉘 스크립트 파일을 그대로 이용했다

```yaml
name: Commit Message Check

on: [push]

jobs:
  commit_message_check:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Read Commit Regular Expression
        id: read_commit_regex
        run: |
          commit_msg_regex=$(cat ./.github/commit-regular.txt)
          echo "Commit Regular Expression: $commit_msg_regex"
          echo "::set-output name=commit_regex::$commit_msg_regex"

      - name: Validate Commit Message
        run: |
          commit_msg=$(git log --format=%B -n 1 ${{ github.sha }})
          echo "Commit Message: $commit_msg"
          commit_regex="${{ steps.read_commit_regex.outputs.commit_regex }}"
          if [[ ! "$commit_msg" =~ $commit_regex ]]; then
            echo "Invalid commit message format."
            exit 1
          else
            echo "Commit message is valid."
          fi
```
> [commit-msg-check.yml](https://github.com/f-lab-clone/ticketing-service/blob/main/.github/workflows/commit-msg-check.yml)


# Lint 해보기

## Local pre-commit Hook
```shell
./gradlew ktlintCheck --daemon

status=$?

echo "$status"

if [ "$status" = 0 ] ; then
    echo "> Completed ktlint hook."
    exit 0
else
    echo "> Error ktlint hook."
    echo "============================"
    exit 1
fi
```
> [pre-commit](https://github.com/f-lab-clone/ticketing-service/blob/main/.github/hooks/pre-commit)

## Git actions

```yaml
 lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Ktlint check
        uses: gradle/gradle-build-action@749f47bda3e44aa060e82d7b3ef7e40d953bd629
        with:
          arguments: ktlintCheck
```
[gradle-build-test-lint.yml](https://github.com/f-lab-clone/ticketing-service/blob/main/.github/workflows/gradle-build-test-lint.yml#L103)

<br/>

`Ktlint`를 예시로 사용했다!