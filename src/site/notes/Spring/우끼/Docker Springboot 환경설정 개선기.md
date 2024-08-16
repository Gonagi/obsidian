---
{"dg-publish":true,"permalink":"/spring//docker-springboot/","tags":["gardenEntry"]}
---


# 문제 상황
### Dockerfile
``` Dockerfile
FROM openjdk:21

ARG JAR_FILE=build/libs/usedStuff-0.0.1-SNAPSHOT.jar app.jar

WORKDIR /usedStuff

COPY ${JAR_FILE} app.jar

ENTRYPOINT ["java","-jar","/usedStuff/app.jar"]
```
- 기존 코드는 다음과 같이 작성되어 있었는데  소스 코드에 변경이 있을때마다 `./gradlew build` 명령을 수동으로 실행하여 JAR 파일을 생성해야하는게 귀찮았다.
---
# 1. Docker 환경에서의 직접 빌드
### Dockerfile.dev
``` Dockerfile
# 빌드 스테이지
FROM openjdk:21-jdk-slim AS builder
WORKDIR /pposonggil

# Gradle Wrapper 복사
COPY gradlew .
COPY gradle gradle
RUN chmod +x ./gradlew

# 의존성 파일 복사 및 다운로드
COPY build.gradle .
COPY settings.gradle .
RUN ./gradlew --no-daemon dependencies

# 소스 코드 복사 및 애플리케이션 빌드
COPY . .
RUN ./gradlew --no-daemon clean build -Pprod

# 실행 스테이지
FROM openjdk:21-jdk-slim
COPY --from=builder /pposonggil/build/libs/*.jar /pposonggil/app.jar
ENTRYPOINT ["java", "-jar", "/pposonggil/app.jar"]
```
- #### 멀티스테이지 빌드 적용
	- 첫번째 FROM(작업공간)에서 build한 후, 두번 째 FROM(작업공간)에서 jar 파일을 복사해 이미지를 만든다.
	- 불필요한 파일들(의존성, 파일, 빌드 도구, ...)을 최종 이미지에서 제외할 수 있어 이미지 크기를 줄일 수 있다.
- `openjdk:21`은 xargs를 지원해 주지 않아 `RUN ./gradlew --no-daemon clean build -Pprod`에서 `xargs not available` 에러를 냈다.
	- `openjdk:21-jdk-slim`으로 이미지를 변경하였다.
### docker-compose.dev.yml 일부
``` docker-compose
pposonggil:
  build:
    context: ./Backend/pposonggil
    dockerfile: Dockerfile.dev
  container_name: pposonggil
  expose:
    - 8080
  environment:
    ...
  depends_on:
    mysql:
      condition: service_healthy
  networks:
    - front-network
    - back-network
    - osrm-network
```
- #### 결과
	- Docker 이미지 내에서 직접 빌드하여 JAR 파일을 생성하고 실행할 수 있도록 했다.
	- 빌드 및 실행 : `docker-compose -f docker-compose.dev.yml up -d --build`
	- 종료 : `docker-compose -f docker-compose.dev.yml down -v`
- #### 문제점
	- 캐시를 활용하지 못하고 코드가 변경될때마다 ==서버를 재시작 해야했다.==
	- 의존성을 다시 다운받고, 빌드 하는 과정이 너무 길었다.
		![Pasted image 20240814160904.png|300](/img/user/Spring/%EC%9A%B0%EB%81%BC/Pasted%20image%2020240814160904.png)
- #### 참고
	- [Dockerfile 내 RUN ./gradlew bootJar 실행시 'xargs not available' 에러](https://velog.io/@mj3242/Dockerfile-%EB%82%B4-RUN-.gradlew-bootJar-%EC%8B%A4%ED%96%89%EC%8B%9C-xargs-not-available-%EC%97%90%EB%9F%AC))
	- [도커파일-피드백](https://www.inflearn.com/community/questions/1169281/%EB%8F%84%EC%BB%A4%ED%8C%8C%EC%9D%BC-%ED%94%BC%EB%93%9C%EB%B0%B1)
___ 
# 2. Layerd JAR
### Layer를 이용한 Dockerfile.prod
``` Dockerfile
# 빌드 스테이지
FROM openjdk:21-jdk-slim AS builder
WORKDIR pposonggil
ARG JAR_FILE=build/libs/*.jar
COPY ${JAR_FILE} application.jar
RUN java -Djarmode=layertools -jar application.jar extract

# 실행 스테이지
FROM openjdk:21-jdk-slim
WORKDIR pposonggil
COPY --from=builder /pposonggil/dependencies/ ./
COPY --from=builder /pposonggil/spring-boot-loader/ ./
COPY --from=builder /pposonggil/snapshot-dependencies/ ./
COPY --from=builder /pposonggil/application/ ./
ENTRYPOINT ["java", "-Dspring.profiles.active=prod", "-Duser.timezone=Asia/Seoul", "org.springframework.boot.loader.launch.JarLauncher"]
```
- #### Layerd JAR
	- JAR 파일을 여러 레이어로 분리하여 런타임에 모듈화 및 커스터마이징할 수 있도록 하는 기술
	- Java9 이상, Spring boot 2.3.0 이상
	- Spring 3.2.X 이후부터는 `org.springframework.boot.loader.JarLauncher`가 아닌 `org.springframework.boot.loader.launch.JarLauncher`를 써야 한다.
- #### 장점
	- 애플리케이션을 더 빠르게 시작할 수 있다.
	- 필요한 모듈만 로드하므로 메모리 사용량이 줄어들고, 배포 및 업데이트가 쉬워진다.
- #### 구성![Pasted image 20240814161437.png](/img/user/Spring/%EC%9A%B0%EB%81%BC/Pasted%20image%2020240814161437.png)
	- application : 애플리케이션 소스 코드
	- snapshot-dependencies : 프로젝트 클래스 경로에 존재하는 스냅샷 종속성 jar 파일
	- dependencies : 프로젝트 경로에 존재하는 라이브러리 jar 파일
	- spring-boot-loader : jar 로더와 런처
- #### Docker에서의 Layered JAR![Pasted image 20240814161739.png](/img/user/Spring/%EC%9A%B0%EB%81%BC/Pasted%20image%2020240814161739.png)
	- 자주 변경되는 레이어를 상위 계층에 둬야 한다.
		- 하위 계층이 변경되면 상위 계층도 재구성함
### docker-compose.prod.yml
``` docker-compose
pposonggil:
  build:
    context: ./Backend/pposonggil
    dockerfile: Dockerfile.prod # 여기만 다름
  container_name: pposonggil
  expose:
    - 8080
  environment:
    ...
  depends_on:
    mysql:
      condition: service_healthy
  networks:
    - front-network
    - back-network
    - osrm-network
```
- #### 결과
	-  애플리케이션 소스 코드만 변경 했을때 필요한 모듈만 로드하므로 메모리 사용량이 줄어들고 빠르다.
	- 속도가 많이 빨라졌다.
	 ![Pasted image 20240814164211.png](/img/user/Spring/%EC%9A%B0%EB%81%BC/Pasted%20image%2020240814164211.png)
	- 빌드 및 실행 : `./gradlew build` + `docker-compose -f docker-compose.prod.yml up -d --build`
	- 종료 : `docker-compose -f docker-compose.prod.yml down -v`
- #### 문제점
	- 코드가 변경될때마다 다시 빌드해야했다.(`./gradlew clean build -Pprod`)
	- 직접 빌드한 후, ==서버도 재시작해야 했다.==
- #### 참고
	- [도커파일-피드백](https://www.inflearn.com/community/questions/1169281/%EB%8F%84%EC%BB%A4%ED%8C%8C%EC%9D%BC-%ED%94%BC%EB%93%9C%EB%B0%B1)
	- [Gradle, Layered Jar 그리고 Dockerbuild 최적화](https://velog.io/@ssol_916/Gradle-Layered-Jar-%EA%B7%B8%EB%A6%AC%EA%B3%A0-Dockerbuild-%EC%B5%9C%EC%A0%81%ED%99%94)
	- [Spring Boot 3.2.X:JarLauncherPath](https://medium.com/viascom/spring-boot-3-2-x-jarlauncher-path-a3656f8e69b4)
	- [Reusing Docker Layers with Spring Boot](https://www.baeldung.com/docker-layers-spring-boot)
---
# 3. auto reload
- 1, 2번 방식 모두 서버가 실행되고 있을 때 jar 파일을 변경해도 바로 반영이 안된다.
	- 항상 `docker-compose up -d --build`로 서버를 재시작 해야 했다.
### build.gradle
``` gradle
dependencies {
	developmentOnly 'org.springframework.boot:spring-boot-devtools'
}

task updateLib(type: Copy) {
	duplicatesStrategy = DuplicatesStrategy.EXCLUDE
	from configurations.compileClasspath into "libs/"
	from configurations.runtimeClasspath into "libs/"
}

task buildAndReload {
	dependsOn build
	mustRunAfter build // buildAndReload must run after the source files are built into class files using build task
	doLast {
		new File(".", ".restart").text = "${System.currentTimeMillis()}" // update trigger file in root folder for hot reload
	}
}
```
- #### updateLib 정의
	- 중복된 파일 제외
	- 컴파일, 런타임 클래스 경로에 있는  파일들을 `libs`로 복사
- #### buildAndReload 정의
	- 빌드 후 애플리케이션을 자동으로 리로드하기 위해 `.restart` 파일 갱신
### Dockerfile.dev
``` Dockerfile
# 빌드 스테이지
FROM openjdk:21-jdk-slim
WORKDIR /usr/src/pposonggil
COPY . /usr/src/pposonggil
VOLUME /tmp

RUN chmod +x entrypoint.sh && ./gradlew updateLib
EXPOSE 8080
CMD [ "sh" , "entrypoint.sh" ]
```
- 현재 디렉터리에 있는 모든 파일들을 `usr/src/pposonggil`에 복사
- `build.gradle`에 정의한 `updateLib`으로 필요한 라이브러리 업데이트 후, entrypoint.sh에 실행 권한을 준 뒤 실행
### entrypoint.sh
``` sh
start_server() {
(sleep 30; ./gradlew buildAndReload --continuous -PskipDownload=true -x Test) &
./gradlew bootRun -PskipDownload=true
}
start_server
```
- 백그라운드에서 buildAndReload를 수행하여 변경사항을 감지하고 실시간으로 빌드(패키지 다운로드, 테스트 제외)를 수행한다.
- 생성한 jar 파일 실행
### aplication.yml
``` application.yml
# devtools  
  devtools:  
    restart:  
      enabled: true # restart 활성  
      additional-paths: . # 추가 경로 설정  
      trigger-file: .restart # 트리거 파일로서 해당 파일이 수정되면 서버가 재시적됨  
    livereload:  
      enables: true # livereload 활성화
```
### docker-compose.dev.yml
``` docker-compose.yml
pposonggil:
  build:
    context: ./Backend/pposonggil
    dockerfile: Dockerfile.dev
  container_name: pposonggil
  volumes:
    - type: bind
      source: ./Backend/pposonggil
      target: /usr/src/pposonggil
  expose:
    - 8080
  environment:
    ...
  depends_on:
    mysql:
	  condition: service_healthy
  networks:
    - front-network
    - back-network
    - osrm-network
```
- `./Backend/pposonggil`에 있는 코드를 바인드 마운트 하여 소스코드를 변경 했을 때 컨테이너 안에 있는 코드도 같이 변경된다.
- #### 결과
	- `docker-compose up -f docker-compose.dev.yml -d --build`로 컨테이너 실행 후 코드를 수정하면, 변경사항을 감지하여 재빌드하고 `.restart`파일을 수정한다. 해당 파일이 트리거가 돼서 서버가 재시작된다.
	- Dockerfile의 `COPY . /usr/src/pposonggil` 때문에 이미지 크기가 늘어났지만, 매번 서버를 재시작 할 필요가 없는것이 매우 편리하다.
- #### 참고 
	- [spring boot 도커환경 auto reload 세팅](https://velog.io/@youngjun0627/spring-boot-%EB%8F%84%EC%BB%A4%ED%99%98%EA%B2%BD-auto-reload-%EC%84%B8%ED%8C%85)
	- [# docker spring boot live reload 개발 환경 구성(vs code)](https://velog.io/@jjh930301/docker-spring-%EA%B0%9C%EB%B0%9C-%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%84%B1vs-code)
---
# 결론
### 개발 환경(dev) → auto reload
- 코드를 변경해도 서버를 재시작 할 필요 없어 개발 효율성↑
### 배포 환경(prod) → Layered JAR
- 캐시 활용, 이미지 크기↓