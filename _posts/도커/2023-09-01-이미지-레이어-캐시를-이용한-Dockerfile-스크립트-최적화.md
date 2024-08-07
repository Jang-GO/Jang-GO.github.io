---
layout: post
title:  "이미지 레이어 캐시를 이용한 Dockerfile 스크립트 최적화"
date:   2023-09-01 15:24:06 +0900
categories: 도커
excerpt_separator: <!--more-->
toc : ture
---
도커파일 스크립트 최적화에 대해 알아봅니다.
(참고: '도커 교과서'책)<br>
<!--more-->
이미지에 애플리케이션이 구현된 자바스크립트 파일이 들어있다고 생각해보자. 잎 파일을 수정하고 이미지를 다시 빌드하면 새로운 이미지 레이어가 생긴다. 도커의 이미지 레이어가 특정한 순서대로만 배치된다고 가정한다. 그래서 이 순서 중간에 있는 레이어가 변경되면 변경된 레이어보다 위에 오는 레이어를 재사용할 수 없다.
기존 이미지의 애플리케이션 내용을 수정하고 다시 빌드하여 이미지 history를 보자.
![이미지](/assets/도커/이미지%20레이어%20캐시를%20이용한%20도커파일%20스크립트%20최적화.png)
<br>
새로운 레이어가 만들어진 것을 볼 수 있다.
도커파일 스크립트의 인스트럭션은 각각 하나의 이미지 레이어와 1:1로 연결된다. 그러나 인스트럭션의 결과가 이전 빌드와 같다면, 이전에 캐시된 레이어를 재사용한다. 이런 방법으로 똑같은 인스트럭션을 다시 실행하는 낭비를 줄일 수 있다. 
<br>
도커는 캐시에 일치하는 레이어가 있는지 확인하기 위해 해시값을 이용한다. 해시값은 Dockerfile 스크립트의 인스트럭션과 인스트럭션에 의해 복사되는 파일의 내용으로부터 계산되는데, 기존 이미지 레이어에 해시값이 일치하는 파일이 없다면 캐시 미스가 발생하고 해당 인스트럭션이 실행된다. 한번 인스트럭션이 실행되면 그다음에 오는 인스트럭션은 수정된 것이 없더라도 모두 실행된다.
<br>
위 history에서 app.js파일이 수정됐으므로 COPY인스트럭션은 실제로 실행 될 것이다. 그 다음 CMD 인스트럭션은 변경된 것이 없지만 COPY 인스트럭션이 실행됐으므로 함께 실행된다.
<br>
이러한 연유로 Dockerfile 스크립트의 인스트럭션은 잘 수 정하지 않는 인스트럭션이 앞으로 오고 자주 수정되는 인스트럭션이 뒤에 오도록 배치해야 한다. 이렇게 해야 캐시에 저장된 이미지 레이어를 되도록 많이 재사용 할 수 있다. 이미지를 공유하는 과정에서 시간은 물론이고 디스크 용량, 네트워크 대역폭을 모두 절약할 수 있는 방법이다. 
<br>
기존 Dockerfile 스크립트를 최적화 해보자.
```
// 기존
FROM diamol/node

ENV TARGET="google.co.kr"
ENV METHOD="HEAD"
ENV INTERVAL="3000"

WORKDIR /web-ping
COPY app.js .

CMD["node", "/web-ping/app.js"]
```
```
// 수정
FROM diamol/node

CMD ["node","/web-ping/app.js"]

ENV TARGET="google.co.kr" \
    METHOD="HEAD" \
    INTERVAL="3000"

WORKDIR /web-ping
COPY app.js
```
7단계의 인스트럭션이 수정 후 5단계로 줄긴 했으나 결과는 동일하다. 그러나 앞에서 말한대로 app.js를 수정하고 빌드해보면 마지막 단계를 제외하고는 모든 레이어를 캐시에서 재사용한다.
<br>
이번 장에서 배운 내용중 중요한 것은 두 가지다. 첫 번째는 Dockerfile 스크립트의 최적화다. 두 번째는 다른 환경에도 애플리케이션을 배포할 수 있도록 이식성 있는 이미지를 만드는 것이다. 이 두가지는 Dockerfile의 인스트럭션을 잘 배치하고 애플리케이션의 설정값을 컨테이너에서 받고록 하는 방법으로 달성할 수 있다. 결과적으로 좀 더 빨리 이미지를 빌드할 수 있으며 테스트 환경에서 검증된 이미지를 운영 환경에 그대로 적용할 수 있다.
