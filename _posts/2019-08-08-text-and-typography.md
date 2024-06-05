---
title: platform-api 에 checkstyle 플러그인 적용하기
description:
date: 2024-05-29 +0900
categories: [Workspace]
tags: [checkstyle, platform-api]
---

> platform-api 에 컨벤션이 없어 가독성이 떨어지고 코드를 이해하기 어려워 Checkstyle 플러그인 적용
{: .prompt-tip }

## Java 에서 주로 사용하는 린트
- [ ] SonarLint
- [X] CheckStyle


참고 : [**CheckStyle vs SonarLint | What are the differences?**](https://stackshare.io/stackups/checkstyle-vs-sonarlint)
<br>
#### Checkstyle을 선택한 이유
- platform-api 의 가장 큰 문제점은 **컨벤션**. 보안 취약점이나 코드 스멜은 나중에 생각할 문제!
- 컨벤션과 맞지 않는 부분 html 파일로 리포트 기능
- gradle task 로 로컬에서 포멧 적용 가능 (쉽게 돌려볼 수 있다는 점)
- 린트 적용 강제화 가능


## 작업 순서
>네이버의 코드 포멧 및 validation-rule 적용 <br>
> <https://github.com/naver/hackday-conventions-java/tree/master/rule-config>
{: .prompt-tip }

#### 1. Code Style 적용
- rule-config/naver-intellij-formatter.xml 다운로드
- IntelliJ IDEA > setting > Editor > Code Style > Java > Schema > Import Schema 에 적용
  - tabs and indents 변경 확인 후 적용

#### 2. IntelliJ에서 CheckStyle 플러그인 설치 및 설정
- IntelliJ IDEA > settings > checkstyle
  - 설정에 사용할 파일을 아래 디렉토리에 저장
![screenshot1](/assets/docs/workspace/checkstyle_1.png){: width="500"}

#### 3. pom.xml 에 의존성 추가
- 문서의 설명에 따라 아래의 값들을 추가함
- failOnViolation : 규칙에 어긋나는 파일이 있으면 빌드 실패(이미 너무 많은 부분에서 발생하고있어 false 로 해둠)
- outputDirectory : 리포트 파일 위치 지정

```xml
<build>
  <pluginManagement>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-checkstyle-plugin</artifactId>
        <version>3.2.1</version>
        <configuration>
          <configLocation>./checkstyle/naver-checkstyle-rules.xml</configLocation>
          <propertyExpansion>suppressionFile=./checkstyle/naver-checkstyle-suppressions.xml</propertyExpansion>
          <failOnViolation>false</failOnViolation>
          <outputDirectory>target/checkstyle</outputDirectory>
        </configuration>
        <dependencies>
          <dependency>
            <groupId>com.puppycrawl.tools</groupId>
            <artifactId>checkstyle</artifactId>
            <version>10.16.0</version>
          </dependency>
        </dependencies>
      </plugin>
    </plugins>
  </pluginManagement>

  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-checkstyle-plugin</artifactId>
    </plugin>
  </plugins>
</build>
```

#### 4. intellij commit 설정 변경하기
![screenshot1](/assets/docs/workspace/checkstyle_2.png){: width="500"}
- 커밋 전 변경 파일에 대해 scan
- 커밋을 못하게 제한하진 않았지만 되도록 수정하고 반영하기로!

## **참고**
- 설정 변경 내용
  - maxLineLen 200으로 변경함
  - 대문자 약어 : DAO, VO, AWS (2024/05/27 기준)
  - EOF : LF로 통일
- 강제화 : 현재 적용X
  - git hook 의 pre-commit 사용해서 적용 가능
  - 현재 warning 이 지나치게 많고, 지나친 강제는 불편을 초래할 수 있다고 생각해서 설정하지 않음
- 모든 패키지에 reformat code한 결과
  - ..어쩜좋아
  
![screenshot1](/assets/docs/workspace/checkstyle_3.png){: width="500"}

