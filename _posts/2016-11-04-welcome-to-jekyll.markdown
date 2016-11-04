---
layout: post
title:  "시에라(sierra) 업데이트 후 Oracle 로케일을 인식할 수 없습니다(locale not recognized) 에러가 났을 때 해결방법"
date:   2016-11-04 17:27:23 +0900
categories: oracle mac
---

OSX 를 시에라로 업데이트한 뒤로 Oracle Client(Java로 만들어진 클라이언트들) 로 접속할 때 마다 "로케일을 인식할 수 없습니다" 에러가 떴다. `-Duser.language=ko -Duser.country=KR` 를 JVM Option 으로 추가하라는 글이 있어 추가했더니 되기는 했는데.. IntelliJ 에서 메소드를 테스트할 때 마다 설정하기도 귀찮고 해서 결과를 찾다가 놀라운 해결책을 발견했다.
 
Mac에서 `시스템 환경 설정 -> 언어 및 지역 -> 지역` 에서 지역을 다른 지역으로 바꿨다가 다시 대한민국으로 바꿔주면 해결된다. 

![Change locale screenshot]({{ site.url }}/assets/2016-11-04/change_locale.png)

Sierra 업그레이드때 내부적으로 참고하는 Locale 이 초기화된 뒤 재할당이 안된 것일까? 다시 지역을 대한민국으로 바꾸면 말끔히 해결된다.