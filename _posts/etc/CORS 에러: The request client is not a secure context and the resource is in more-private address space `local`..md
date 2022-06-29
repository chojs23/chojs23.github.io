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

이를 해결하기 위해서는 브라우져 설정에서, 위와 같은 제한을 해제해주어야 한다.

크롬의 경우: chrome://flags/#block-insecure-private-network-requests 에 들어가서 설정 disabled
엣지의 경우: edge://flags/#block-insecure-private-network-requests 에 들어가서 설정 disabled

<script src="https://utteranc.es/client.js"
        repo="chojs23/comments"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
