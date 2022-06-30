---
title: "[AWS] Deploying Containerized Java Applications with AWS Elastic beanstalk"
excerpt:
published: false
categories:
    - AWS

toc: true
toc_sticky: true

date: 2022-06-30
last_modified_at: 2022-06-30
---

## maven docker plugin

빌드할 프로젝트의 pom.xml 파일에 플러그인 추가

```
<build>
		<plugins>

			...

			<!-- Docker -->
			<plugin>
				<groupId>com.spotify</groupId>
				<artifactId>dockerfile-maven-plugin</artifactId>
				<version>1.4.10</version>
				<executions>
					<execution>
						<id>default</id>
						<goals>
							<goal>build</goal>
							<!-- <goal>push</goal> -->
						</goals>
					</execution>
				</executions>
				<configuration>
					<repository>in28min/${project.artifactId}</repository>
					<tag>${project.version}</tag>
					<skipDockerInfo>true</skipDockerInfo>
				</configuration>
			</plugin>
		</plugins>
	</build>
```

이 플러그인을 추가하면 도커파일을 관리할 수 있게된다.

Dockerfile

```
FROM openjdk:8-jdk-alpine
VOLUME /tmp
EXPOSE 5000
ADD target/*.jar app.jar
ENV JAVA_OPTS=""
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar /app.jar" ]
```

dockerfile에 어떻게 컨테이너화 시킬지 작성하면 이 플러그인은 mvn package 명령을 실행했을 때 도커파일 명령들을 실행하고 도커 이미지를 만든다.

<script src="https://utteranc.es/client.js"
        repo="chojs23/comments"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
