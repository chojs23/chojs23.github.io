---
title: "CORS 에러: The request client is not a secure context and the resource is in more-private address space `local`."
excerpt:
published: true
categories:
    - etc

toc: true
toc_sticky: true

date: 2022-06-30
last_modified_at: 2022-06-30
---

![1](../../images/etc/Screenshot%202022-06-29%20at%2023.50.18.jpg)

AWS S3에 올린 정적 웹사이트가 로컬에서 실행중인 서버에 요청을 보낼 때 CORS 에러가 발생하였다. 서버에서 CORS를 허용해주었음에도 에러가 발생하였는데, origin 보다, 더 낮은 수준의 네크워크로 요청을 보내는 경우, 위와같이 에러를 발생한다.

CORS 에러는 브라우저의 동일 출처 정책(Same-Origin Policy)을 위반하면 발생하는 에러인데, 여기서 동일 출처란 프로토콜, 호스트(도메인), 포트가 모두 같은 것을 말한다. 'http://mypage:8080'에서 요청을 보낸다고 했을 때, 동일 출처 정책을 위반한 경우는 아래와 같다.

-   https://mypage:8080/v1/api/product/list로 요청을 보낸 경우 > 프로토콜 불일치
-   http://yourpage:8080/v1/api/product/list로 요청을 보낸 경우 > 호스트(도메인) 불일치
-   https//mypage:8090/v1/api/product/list로 요청을 보낸 경우 > 포트 불일치

이를 해결하기 위해서는 브라우져 설정에서, 위와 같은 제한을 해제해주어야 한다.

크롬의 경우: chrome://flags/#block-insecure-private-network-requests 에 들어가서 설정 disabled
엣지의 경우: edge://flags/#block-insecure-private-network-requests 에 들어가서 설정 disabled

> 참고 : https://nankisu.tistory.com/70
> https://wicg.github.io/private-network-access/#intro

<script src="https://utteranc.es/client.js"
        repo="chojs23/comments"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
