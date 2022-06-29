---
title: "[AWS] Deploying Full Stack App with AWS Elastic Beanstalk & S3"
excerpt:
published: true
categories:
    - AWS

toc: true
toc_sticky: true

date: 2022-06-30
last_modified_at: 2022-06-30
---

## Deploying Java REST API Backend to AWS Elastic Beanstalk

이전과 같은 방법으로 AWS Elastic Beanstalk에 배포하므로 생략

## Building React Frontend Code for AWS Deployment

React 어플리케이션의 API_URL을 위에서 AWS Elastic Beanstalk에 등록한 REST API 서버로 바꿔주자.

```
//export const API_URL = 'http://localhost:5000'
export const API_URL = 'http://rest-api-in28minutes-qa.kvajmpmiiu.us-east-1.elasticbeanstalk.com'
export const JPA_API_URL = `${API_URL}/jpa`
```

React 어플리케이션을 npm run build 명령을 통해 빌드하자. 빌드가 완료되면 build 폴더가 생긴다.

## AWS Simple Storage Service - S3

정적 콘텐츠를 배포하는 방법에는 여러가지가 있다. 다른 대부분의 클라우드 서비스에서는 정적 콘텐츠를 배포하는 nginx 서버가 있거나, 자신만의 웹 서버를 만들어 거기에 정적 컨텐츠를 올린다. AWS 에서는 S3를 통해 가능하다. S3 는 Simple Storage Service의 약자로 클라우드에서 확장 가능한 저장소이다.
Bucket이라는 저장소에 저장되는데 우리가 Elastic Beanstalk에 배포한 jar, war 파일등이 저장되어있다. 아카이브 파일, 로그 등의 파일을 저장하고 싶을 때 S3를 사용하면 된다.

S3는 정적 컨텐츠를 제공하는 웹 서버의 기능으로써도 사용할 수 있다. S3는 높은 내구성을 제공한다. S3에 객체를 저장하면 99.9999...%의 내구성을 제공한다.
따라서 어떤 환경이던지 데이터가 유지될 수 있다. 또한 높은 가용성을 제공한다.

S3는 지역에 관계되지 않는 글로벌 서비스이다. 사용자는 지역을 특정하지 않아도 S3의 서비스를 이용할 수 있다.

## Deploying React App to AWS S3 Static Website

S3 bucket을 생성하자

![1](../../images/aws/Screenshot%202022-06-29%20at%2023.11.39.jpg)

bucket name 은 unique 해야함

만든 bucket에 아까 빌드한 build 폴더안의 파일들을 업로드

![1](../../images/aws/Screenshot%202022-06-29%20at%2023.15.08.jpg)

이 정적 파일들을 웹 서버에서 이용하기 위해 Properties -> Static website hosting 에서 설정하자.

![1](../../images/aws/Screenshot%202022-06-29%20at%2023.18.04.jpg)

![1](../../images/aws/Screenshot%202022-06-29%20at%2023.20.17.jpg)

생성된 웹사이트 엔드포인트에 가보면 403 에러가 나온다.

업로드한 파일들은 보안되어져있어 아무나 접근이 가능하지 않다. 따라서 우리가 만든 bucket을 pulic으로 만들어 줘야한다. Permission 탭에 가면 bucket public access 설정이 가능하고 bucket policy에서 모든 bucket의 policy 설정이 가능하다.

Block public access 가 기본적으로 설정되어있는데 해제하고 bucket policy를 설정하자

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AddPerm",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::full-stack-frontend-qa-neo/*"
        }
    ]
}
```

이제 웹사이트 엔드포인트에 접근하면 정상적으로 접근이 가능해진다.

<script src="https://utteranc.es/client.js"
        repo="chojs23/comments"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
