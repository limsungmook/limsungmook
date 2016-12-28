---
layout: single
title:  "2년 된 Spring 소스코드를 Spring Boot 로 바꾼 후기"
date:   2016-12-28 16:40:23 +0900
tags: spring-boot spring-legacy
published: true
---

cafe24 에서 사용하는 가상 호스팅이 해킹으로 먹통이 된 김에 OS포맷을 했다. 
포맷하고 서버를 다시 띄우려니 tomcat도 설치해야 하고 이것저것 너무 귀찮은게 많다..
Spring 도 2년 된 3.1.3 버전이고 그때 당시 설정을 죄다 xml 로 해 놓은 터라 나중에 읽어보니 아주 가독성이 떨어졌다. 
DB Query 도 Hibernate Criteria 로 날렸는데.. 개인 서비스가 뭐 얼마나 복잡한 쿼리가 있겠나. JPA의 Method Query 로 변경되겠다 싶어 작업을 했다.


spring-boot 4.1.3 으로 변경을 했는데 생각보다 간단하게 끝났다. pom.xml 도 spring 관련 dependency 들이 정리되고 jackson 같은 넘들도 spring-boot 에 기본 내장되어 있어 
아주 간소하게 pom.xml 이 변해버렸다. 패키징 방식도 war 에서 jar 로 변경!

xml 에 넣었던 DataSource 도 application.properties 에 빼고 내친김에 Maven Profile 로 xml 이 분기되던 설정들을 application-{profile}.properties 로 넣어버렸다.
이렇게 되니 Runtime 에 -D 옵션으로 정말 간단하게 property 를 변경할 수 있게 되어 정말 좋아졌다.
 
xml 은 다 삭제하고 JavaConfiguration 으로 변경했다. 근데 boot 에서 자동으로 다 설정해줘서 추가한 넘은 Security 와 JSP 설정밖에 없다.

패키징이 Executable Jar 로 되니 정말 편해졌다. 서버에 jre 만 설치되어 있으면 ./deploy.jar 나 java -jar deploy.jar 로 간단하게 실행할 수 있다. 
 jsp 파일은 resources/META-INF/resources/WEB-INF/jsp 같이 resources/META-INF/resources 안에 있어야 한다. 인터넷에 있는 webapp/WEB-INF/jsp 설정으로 하면 mvn spring-boot:run 으로 실행했을 땐 잘 될지 몰라도
  패키징 된 .jar 파일을 그대로 실행하면 jsp 파일을 못 찾는 이슈가 있다.
  
 모든 Repository Class 들을 Repository Interface 로 변경했다. JPA 는 정말 막강한 것 같다. 이전엔 Paging 처리도
손수 구현했었는데(좀 간지나는 패턴으로 했었는데..) Pageable 인터페이스로 기본 제공하니 Paging 에 사용되었던 유틸 클래스나 커스텀 제네릭 레파지토리도 제거할 수 있게 되었다.

그 외에 자잘한 작업들 좀 해줬어야했지만 그래도 정말 간단하게 마이그레이션이 되었다. spring-boot 가 나와 정말 행복하다 ㅠㅠ 
