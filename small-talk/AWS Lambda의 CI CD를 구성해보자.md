## AWS Lambda의 CI CD를 구성해보자

## 1. 개요

지난 번의 [포스팅에](https://github.com/G0RANI/eye-opener/blob/main/small-talk/AWS%20Lambda%EB%A5%BC%20%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC%20DQL%EC%97%90%20%EC%9E%88%EB%8A%94%20%EB%A9%94%EC%84%B8%EC%A7%80%EB%A5%BC%20SQS%EB%A1%9C%20redrive%20%ED%95%B4%EB%B3%B4%EC%9E%90.md) 이어 작성한다. </br>
Lambda에 대하여 CI/CD가 필요함을 느꼈고 AWS codecommit과 CodePipeline을  이용해 관리를 하기로 한다.</br>

## 2. CodeCommit과 CodeDeploy

### 2.1 CodeCommit 이란?

AWS CodeCommit는 프라이빗 Git 리포지토리를 호스팅하는 확장성이 뛰어난 관리형 소스 제어 서비스입니다. 코드를 저장하기 위한 리포지토리를 생성하기만 하면 됩니다. 프로비저닝 및 확장할 하드웨어나 설치, 구성, 운영할 소프트웨어가 없습니다. AWS CodeCommit의 가져오기 요청, 분기 및 병합 기능을 사용하면 협업하여 코드 작업을 할 수 있습니다. 코드 검토 및 피드백이 기본적으로 포함되는 워크플로를 구현하고, 특정 분기를 변경할 수 있는 사용자를 제어할 수 있습니다.

출처 : https://aws.amazon.com/ko/codecommit/features/

### 2.2 CodePipeline 이란?

AWS CodePipeline은 소프트웨어를 릴리스하는 데 필요한 단계를 모델링, 시각화 및 자동화할 수 있게 해주는 지속적 전달 서비스입니다. AWS CodePipeline을 사용하여 코드 빌드, 사전 프로덕션 환경으로의 배포, 애플리케이션 테스트 및 프로덕션으로 릴리스를 비롯한 전체 릴리스 프로세스를 모델링합니다. 그러면 AWS CodePipeline이 정의된 워크플로우에 따라 코드 변경이 있을 때마다 애플리케이션을 빌드, 테스트, 배포합니다. 파트너 도구 및 자체 사용자 지정 도구를 릴리스 프로세스 중 원하는 단계에 통합하여 포괄적이며 지속적 전달 솔루션을 형성할 수 있습니다.

출처 : https://aws.amazon.com/ko/codepipeline/faqs/

## 3. 배포 과정

1. 코드를 작성하여 repository에 commit & push
2. CodePipeline에서 변경을 감지하여 코드를 가지고 온다.
3. 가지고 온 코드를 CodeBuild 에서 build
4. CodeBuild에서 buildspec.yml 파일의 aws cli를 이용하여 반영

## 4. 작업 목록

* Lambda
  * 이건 저번 포스팅에 작성을 하였다.
  * CodeBuild를 이용하기 위해 buildspect.yml 파일 작성 필요
* CodeCommit
  * repository 생성
  * branch 생성
* CodeDeploy
  * Group 생성
* CodeBuild 생성
* CodePipeline 생성
  * CodeCommit 과 연결


## 5. 구성

### 5.1 CodeCommit 생성

<img src="https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/1.png?raw=true" alt="1.png" style="zoom:50%;" />

<img src="https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/2.png?raw=true" alt="2.png" style="zoom:50%;" />

<img src="https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/3.png?raw=true" alt="3.png" style="zoom:50%;" />

생성 후 생성한 코드와 repository 연결을 하고 develop branch를 생성을 하였다.

![4.png](https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/4.png?raw=true)



### 5.2 CodeBuild 생성

<img src="https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/5.png?raw=true" alt="5.png" style="zoom:50%;" />

빌드 생성

![7.png](https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/7.png?raw=true)

소스 공급자를 CodeCommit으로 선택해주어야 한다.

<img src="https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/8.png?raw=true" alt="8.png" style="zoom:50%;" />

buildspec을 사용하기에 선택, root에 buildspec.yml 파일을 생성하여 작성해준다.

<img src="https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/17.png?raw=true" alt="17.png" style="zoom:50%;" />



### 5.4 CodePipeline 생성

<img src="https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/9.png?raw=true" alt="9.png" style="zoom:50%;" />

파이프라인 생성

<img src="https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/10.png?raw=true" alt="10.png" style="zoom:50%;" />

<img src="https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/11.png?raw=true" alt="11.png" style="zoom:50%;" />

소스 공급자에 CodeCommit 선택 후 리포지토리, 브랜치도 선택해준다.

<img src="https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/12.png?raw=true" alt="12.png" style="zoom:50%;" />

빌드 공급자에 CodeBuild 선택

<img src="https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/13.png?raw=true" alt="13.png" style="zoom:50%;" />

배포는 buildspec.yml에서 AWS cli로 진행하였기 때문에 넘어간다.

<img src="https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/14.png?raw=true" alt="14.png" style="zoom:50%;" />

최종 모습, 파이프라인 생성을 한다.



## 6. 결과

파이프 라인을 생성하면 즉시 진행 되고 결과를 확인 할 수 있다.

<img src="https://github.com/G0RANI/eye-opener/blob/main/image/10_96f99d18/15.png?raw=true" alt="15.png" style="zoom:50%;" />



## 7. 정리

간단하게 Lambda 코드를 콘솔에서가 아닌 CodeCommit으로 관리 하여 소스 코드의 변동을 감지하여 배포까지 하는 파이프라인을 생성해보았다.

여기서 추가로 더 진행 할 수 있는 부분이라 하면 이정도가 있다 한다.

- **배포 트래픽 조절**

배포 중에 Lambda 함수에 대한 트래픽을 제어하고 조절할 수 있다. "배포 구성 구성" 단계에서 "서비스 역할의 최소 요청 비율" 및 "서비스 역할의 최대 요청 비율" 옵션을 구성하여 배포 동안의 트래픽 비율을 조정할 수 있다.

- **환경 변수 관리**

Lambda 함수에 대한 환경 변수를 관리하고 업데이트할 수 있다. 배포 구성 구성 단계에서 "환경 변수" 섹션에서 Lambda 함수에 대한 환경 변수를 구성하고, 배포 시에 환경 변수 값을 업데이트할 수 있다.

- **알림 구성**

배포 상태에 대한 알림을 구성하여 배포 프로세스를 모니터링할 수 있다. AWS CloudWatch 이벤트 또는 Amazon SNS를 사용하여 배포 시작, 완료 또는 실패와 같은 이벤트에 대한 알림을 설정할 수 있다.

- **롤백 구성**

배포 중에 문제가 발생한 경우, 롤백 기능을 사용하여 이전에 성공한 상태로 Lambda 함수를 롤백할 수 있다. 배포 구성 구성 단계에서 "롤백 구성"을 설정하여 롤백을 활성화하고, 롤백 시 Lambda 함수를 복원할 소스를 구성한다.

- **고급 배포 구성**

롤링 업데이트 대신 다른 배포 방식을 선택하여 배포 속도 및 전략을 구성할 수 있다. "배포 구성 구성" 단계에서 "고급 배포 구성"을 선택하고, 배포 그룹에 대한 세부적인 구성 옵션을 조정할 수 있다.



이중에서 환경 변수와 알림 그리고 롤백 구성은 필수적으로 더 추가 할 예정이다.

다음 내용은 해당 내용을 추가 하는 내용이 될 수 있도록 해보겠다.
