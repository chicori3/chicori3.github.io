---
title: 'Github Action을 사용해보자'
categories:
  - 삽질
tags:
  - CI/CD
  - GitHub
toc: true
toc_sticky: true
---

회사에서 혼자 맡은 작은 프로젝트를 배포해봤습니다.

처음으로 로컬이 아닌 클라우드 서버에서 돌아가는 제 프로젝트를 보니 재밌더라고요.

하지만 문제가 있었습니다.

## 번거로운 수동 배포

개발자는 게을러야 한다고 생각합니다.

직접 gradle의 bootJar를 실행하고, 생성된 jar를 직접 클라우드 서버로 옮기고, 서버에서 직접 스크립트를 실행시키는 것은 개발자스럽지 않다고 생각했습니다.

때문에 코드가 반영이 되면 자동으로 빌드, 배포가 되도록 해보자 했고 깃허브를 사용한 프로젝트이다 보니

좀 간편한 **"Github Action"**을 사용해봤습니다.

## 깃허브 액션

[GitHub Actions Documentation](https://docs.github.com/en/actions)

깃허브 액션은 빌드, 테스트 그리고 배포를 자동화할 수 있는 CI/CD 플랫폼이라고 합니다.

리눅스, 윈도우, MacOS 가상 서버를 지원하며 원하는 환경으로 필요한 설정들을 구성할 수 있습니다.

깃허브 액션은 주요한 5개의 개념이 있습니다.

Events

-   깃허브 액션을 실행시키는 조건, 이벤트 트리거의 역할을 합니다
-   특정 브랜치로 push가 되었거나, 이슈가 생겼을 때 등의 이벤트가 발생하면 깃허브 액션이 실행되도록 할 수 있습니다. 

Workflow

-   하나 이상의 작업을 실행할 수 있는 자동화 프로세스
-   event에 의해 실행됩니다

Jobs

-   하나 이상의 step으로 구성될 수 있는 실행 단위입니다
-   job은 기본적으로 병렬로 실행되며 step들은 순차적으로 실행됩니다

Actions

-   복잡한 step들을 사용하기 편하도록 정의한 커스텀 애플리케이션
-   필요하다면 마치 라이브러리처럼 가져와서 사용할 수 있습니다

Runner

-   Workflow를 실행하는 서버입니다
-   각 러너는 가상 머신에서 하나의 job을 실행합니다

## 깃허브 액션 YML 작성하기

```
name: Workflow 이름
on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
```

대충 이름과 이벤트를 설정해줍니다.

저의 경우에는 main 브랜치로 push 또는 PR 시 실행되도록 작성했습니다.

```
jobs:
  build:
    runs-on: ubuntu-latest
```

이벤트가 발생하면 작업을 수행할 job입니다.

빌드 및 배포가 목적이기 때문에 build로 작성했고 사용될 가상 머신은 Ubuntu로 지정했습니다.

```
 steps:
      - uses: actions/checkout@v3

      - name: Build Project
        run: |
          chmod 777 ./build.sh
          chmod +x gradlew
          ./build.sh
      
      - name: SCP Jar
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.NAME }}
          password: ${{ secrets.PASSWORD }}
          port: ${{ secrets.PORT }}
          source: "./app.jar"
          target: "/app"
          strip_components: 1

      - name: Start Service
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.NAME }}
          password: ${{ secrets.PASSWORD }}
          port: ${{ secrets.PORT }}
          script: |
            systemctl start 프로젝트
```

Github Actions의 checkout 액션을 사용하여 최신 상태의 코드를 내려받습니다.

Build Project는 제가 미리 작성한 스크립트를 실행합니다.

이때 생성된 jar 파일을 저희 서버 클라우드로 전송해야 하는데 이 때 scp-action을 사용할 수 있습니다.

[GitHub - appleboy/scp-action](https://github.com/appleboy/scp-action)

전송된 jar 파일을 실행하기 위해선 ssh로 접근하여 미리 작성한 리눅스 서비스를 실행시켜줘야 했습니다.

ssh-action은 ssh에 접근할 수 있도록 정의된 action이며 각각 사용하시는 호스트, 이름, 비밀번호, 포트 등을 작성하시면 됩니다.

[Github - appleboy/ssh-action](https://github.com/appleboy/ssh-action/actions)

## 결과

![](https://user-images.githubusercontent.com/40778768/198823099-e219ae4f-e460-4108-9484-1e3a09b28188.png){: .align-center}
*수많은 삽질 끝에 성공*

처음 사용해보는 기술에다가 수많은 오타가 있었지만 끝내 성공해냈습니다.

Github Action은 따로 설치 필요 없이 사용할 수 있고 저보다 나은 개발자들이 미리 작성해놓은 Action을 사용할 수 있기 때문에
쉽게 배워 써먹을 수 있어 좋은 것 같습니다.

> 참고  
> [드림코딩 엘리 - 깃허브 액션](https://www.youtube.com/watch?v=iLqGzEkusIw&t=564s)
