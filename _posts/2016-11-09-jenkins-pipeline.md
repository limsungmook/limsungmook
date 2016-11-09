---
layout: single
title:  "Jenkinsfile 을 이용한 젠킨스 Pipeline 설정"
date:   2016-11-09 16:40:23 +0900
tags: jenkinsfile pipeline jenkins ci workflow
published: true
---

github 의 오픈소스들을 보면 많은 repository 에서 `.travis.yml` 를 통해 지속적인 통합 및 배포 를 진행하는 것을 볼 수 있다.
Jenkins 에서 이런저런 Job 설정을 통해서만 빌드 및 배포를 했던 나로써는 통합 및 배포 설정이 소스코드 안에 들어있는 것에서 적잖은 충격을 받았었다.
통합 및 배포(CI & CD)에 대한 설정이 소스코드에 있으면 장점이 많다. 예전엔 젠킨스에 접속해 job들을 일일이 클릭하며 확인했었는데 아래의 내용을 소스코드로 Fix하고 공유 할 수 있게 된다.
 
 * 소스코드 빌드 방법(maven profile 등에 대한 설정)
 * 테스트 및 정적분석 프로세스
 * 스테이지 별 배포 프로세스
 
위 사항들에 대한 문서화가 소스코드에서 이루어지게 되고, 이 문서는 실제로 작동하는 살아있는 문서가 될 수 있다.

지금까지는 새로운 빌드 환경이 생기면 젠킨스에서 Job 을 복사해서 사용하다보니 설정의 중복이 발생했고, 중복은 당연하게도 설정 변경 시 매우 큰 귀찮음은 물론 장애 가능성도 안겨주었다.

Jenkins 는 버전 1.642.3 부터 소스코드로써 빌드 및 배포에 대한 설정을 관리하는 방식 제공하는데, 이를 [Pipeline][pipeline] 이라 한다. 코드로 정리하는 것에 대한 가치는 [Pipeline as Code][pipeline-as-code] 에 정리되어 있다.

Pipeline 은 소스코드에 포함된 [Jenkinsfile] 을 통해 관리할 수 있다. 이 글에서는 기존 Job 스타일에서 Jenkinsfile 로 옮겨가는 시작점에 대해 기술해본다.
 
## 준비

기본적인 빌드가 가능한 Jenkins(버전 2.x 이상 권장) 를 준비한다. 여기서 젠킨스는 Git plugin, JDK, Maven 등 Pipeline 없이 기존 방식으로는 빌드할 수 있을 정도의 세팅이 완료된 젠킨스를 뜻한다. 
여기선 젠킨스 관리 -> Global Tool Configuration 에서 Maven 을 M3(maven 3)로 설정해놓았다
![Maven install as M3]({{ site.url }}/assets/2016-11-09/M3_maven_install.png)

## 설치 

젠킨스 관리 -> 플러그인 관리에서 `Pipeline Plugin` 을 설치한다.
  
## 설정

젠킨스에서 `Pipeline` Job 을 만든다.  
![Create pipeline job]({{ site.url }}/assets/2016-11-09/create_pipeline_job.png)

Job 을 만든 후에 복잡하게 설정을 할 필요는 없다. 통합 및 배포에 대한 Workflow 는 소스코드 안에 존재한다.

![Write github url and Jenkinsfile path]({{ site.url }}/assets/2016-11-09/setup_job.png)

Pipeline Definition 엔 Pipeline Script 도 존재하는데, 이 설정으로 하면 소스코드 안에 Jenkinsfile 을 만들 필요 없이 잡 설정 내의 Inline 으로 스크립팅을 할 수 있다.
이 예제에선 소스코드 내의 Jenkinsfile 로 진행한다.
여기서 Jenkinsfile.groovy 를 사용한 것을 볼 수 있는데 젠킨스에서 구현한 Pipeline as Code 는 Groovy Script 를 통해 실행된다.
공식 홈페이지에서 권장하는 파일 이름은 Jenkinsfile 이지만 IDE 의 지원을 받아 스크립트를 원활하게 작성하려면 .groovy extension 이 붙어있는게 편해 실제 프로젝트에선 이렇게 진행하고 있다.

이제 Jenkins 설정은 완료 되었다! 이제 파이프라인의 Spec 이 정의된 Jenkinsfile.groovy 을 만들면 된다.

## Pipeline as Code

이 예제에선 소스코드 통합 시 가장 흔하게 발생할 수 있는 두 가지 요구사항을 충족해보고자 한다.

* 빠른 배포 파일 생성을 위해 Test 를 Skip 한다.
* Job마다 Profile 을 선택적으로 적용한다.
 
위 두 요구사항과 기본적인 Build Pipeline 을 정의한 스크립트는 아래와 같다.

{% highlight groovy linenos %}

node {
    // Jenkins 파일에서 취급하는 파라미터들을 미리 정의한다.
    // 아래와 같이 미리 정의하면 Jenkins Job 이 Parametrized Job 이 되며 기본 변수들이 들어가게 된다
    properties(
            [
                    [$class: 'ParametersDefinitionProperty', parameterDefinitions:
                            [
                                    [$class: 'BooleanParameterDefinition', defaultValue: false, description: '테스트를 Skip 할 수 있습니다. 선택 시 테스트를 건너뛰고 체크아웃 - 빌드 - 아카이빙만 진행합니다', name: 'skipTests']
                                    , [$class: 'StringParameterDefinition', defaultValue: 'hsqldb', description: 'Spring Boot 에서 Active 할 Profile 들을 쉼표로 분리해서 입력하세요. 예) hsqldb,mysql', name: 'activeProfiles']
                            ]
                    ]])

    def mvnHome

    stage('Preparation') { // for display purposes
        echo "Current workspace : ${workspace}"
        // Get the Maven tool.
        // ** NOTE: This 'M3' Maven tool must be configured
        // **       in the global configuration.
        mvnHome = tool 'M3'
    }
    stage('Checkout') {
        // Get some code from a Git repository
        checkout scm
    }
    if (skipTests != true) {
        stage('Test') {
            sh "'${mvnHome}/bin/mvn' -Dspring.profiles.active=${activeProfiles} -Dmaven.test.failure.ignore -B verify"
        }
        stage('Store Test Results') {
            junit '**/target/surefire-reports/TEST-*.xml'
        }
    }
    stage('Build') {
        sh "'${mvnHome}/bin/mvn' -Dspring.profiles.active=${activeProfiles} -Dmaven.test.skip=true clean install"
    }
    stage('Archive') {
        archive '**/target/*.jar'
    }
    stage('Deploy') {
        echo "Deploy is not yet implemented"
    }
}

{% endhighlight %}

Pipeline 의 각 단계는 stage 로 나누어져 진행된다. 각 단계별로 실제 진행해야할 사항들을 넣으면 젠킨스 UI 에서 비주얼하게 보여진다.
4-10line 이 조금 특별하다고 볼 수 있는데, 위 코드를 사용하여 빌드를 진행하면 Job 자체에 Parameterized Build 옵션이 켜지게 된다. 
Job 자체에 따로 Environment 를 지정할 수도 있지만 이렇게 코드 자체에 Parameter를 넣도록 강제하면 소스코드와 젠킨스 사이의 갭이 줄어들게 된다.
이렇게 젠킨스로부터 Inject 된 변수들은 groovy 인스턴스로써 제어할 수 있게 된다.

위 Pipeline 은 skipTests 옵션이 켜져 있으면 test 를 skip 하도록 분기하고 Job 마다 Spring Boot Profile 을 따로 설정할 수 있도록 구성되어있다.
여기서 만든 파이프라인은 Preparation -> Checkout -> Test -> Store Test Results -> Build -> Archive -> Deploy 의 단계로 나누어져 있는데 필요하면 Analytics, Notification 과 같은 Stage 를 추가하면 된다.

## 실행

소스를 Checkout 받지 않은 상태인 처음 빌드엔 Build with Paramterers 가 나타나지 않는다.
일단 빌드를 한번 실패해야만 Build with Parameters 가 나타나 아래 화면으로 빌드할 수 있다.

![Build with parameters]({{ site.url }}/assets/2016-11-09/build_with_parameters.png)

빌드를 실행하면 최종 결과가 아래와 같이 나오게 된다

![Build Pipeline Result]({{ site.url }}/assets/2016-11-09/build_pipeline_result.png)

## 아쉬운 점

Jenkins 의 투박한 방식은 약간 아쉬움이 남는다.
예를 들면 위 4-10Line 의 Properties 를 지저분하게 넣는 것 외에는 소스코드로 Job 의 구조를 강제화할 방법이 보이지 않는다. 
그리고 일단 이렇게 설정을 하게 되면 Build with Parameters 옵션이 켜져 빌드할 때 Property 를 바꿀 수 있게 된다. Job 에서 빌드하는 방식을 고정하는 것을 좋아하는 나로써는 약간의 찝찝함을 남긴다.

Sample Code : [Github skeleton-ws-spring-boot](https://github.com/limsungmook/skeleton-ws-spring-boot)



[pipeline]: https://jenkins.io/doc/book/pipeline/overview/
[pipeline-as-code]: https://jenkins.io/solutions/pipeline/
[Jenkinsfile]: https://jenkins.io/doc/book/pipeline/jenkinsfile/