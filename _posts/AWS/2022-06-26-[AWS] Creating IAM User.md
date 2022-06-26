---
title: "[AWS] Creating IAM User"
excerpt:
published: true
categories:
    - AWS

toc: true
toc_sticky: true

date: 2022-06-26
last_modified_at: 2022-06-26
---

## IAM User 란

AWS Identity and Access Management(IAM) 사용자는 AWS에서 생성하는 엔터티로서 AWS와 상호 작용하기 위해 해당 엔터티를 사용하는 사람 또는 애플리케이션을 나타냅니다. AWS에서 사용자는 이름과 자격 증명으로 구성됩니다.

## AWS 계정의 IAM 사용자 생성

1. AWS Management Console, AWS CLI 또는 Tools for Windows PowerShell에서나 AWS API 작업을 사용하여 사용자를 생성합니다. AWS Management Console에서 사용자를 생성하면 선택한 항목에 따라 1~4단계는 자동으로 처리됩니다. 프로그래밍 방식으로 사용자를 생성하는 경우, 각 단계를 개별적으로 수행해야 합니다.

2. 사용자에게 필요한 액세스 유형에 따라 사용자의 자격 증명을 생성합니다.

-   프로그래밍 방식으로 액세스: IAM 사용자가 API를 호출해야 하거나, AWS CLI 또는 Tools for Windows PowerShell을 사용해야 할 수 있습니다. 이 경우 해당 사용자의 액세스 키를 만드십시오(액세스 키 ID 및 보안 액세스 키).

-   AWS Management Console 액세스: 사용자가 AWS Management Console에서 액세스해야 할 경우, 에서 해당 사용자의 암호를 생성합니다. 사용자의 콘솔 액세스를 비활성화하면 사용자가 사용자 이름과 암호를 사용하여 AWS Management Console에 로그인하지 못합니다. 그렇더라도 사용자의 권한이 변경되거나 위임된 역할을 사용하여 콘솔에 액세스하는 것을 방지하지는 않습니다.
    <br>

    사용자가 필요한 자격 증명만 생성하는 것이 가장 좋습니다. 예를 들어, AWS Management Console을 통해서만 액세스해야 하는 사용자에게는 액세스 키를 생성해서는 안 됩니다.

3. 해당 사용자를 하나 이상의 그룹에 추가하여 필요한 작업을 수행할 수 있는 권한을 부여합니다. 권한 정책을 사용자에게 직접 연결하여 권한을 부여할 수도 있습니다. 하지만, 사용자를 그룹에 추가한 후 그 그룹에 연결된 정책을 통해 정책과 권한을 관리하는 것이 좋습니다. 권한 경계를 사용하여 일반적이지는 않지만 사용자에게 있는 권한을 제한할 수 있습니다.

## IAM 사용자 생성(콘솔)

AWS Management Console을 사용하여 IAM 사용자를 생성할 수 있습니다.

**한 명 이상의 IAM 사용자를 생성하려면(콘솔)**

1. AWS Management Console에 로그인하여 https://console.aws.amazon.com/iam/에서 IAM 콘솔을 엽니다.

2. 탐색 창에서 사용자(Users)와 사용자 추가(Add users)를 차례로 선택합니다.

3. 신규 사용자의 사용자 이름을 입력합니다. 이것은 AWS에 로그인할 때 사용하는 이름입니다. 여러 사용자를 동시에 추가하려면, 추가하는 각 사용자에 대해 [다른 사용자 추가(Add another user)]를 선택한 후 사용자 이름을 입력합니다. 한 번에 최대 10명까지 사용자를 추가할 수 있습니다.

4. 이 사용자 세트에게 부여할 액세스 권한의 유형을 선택합니다. 프로그래밍 방식 액세스나 AWS Management Console에 대한 액세스 또는 둘 다를 선택할 수 있습니다.

    - 사용자가 API, AWS CLI 또는 Tools for Windows PowerShell에 대한 액세스 권한이 필요한 경우, 프로그래밍 방식 액세스(Programmatic access)를 선택합니다. 이렇게 하면 각 사용자에 대한 액세스 키가 생성됩니다. 최종(Final) 페이지에 이르면 액세스 키를 보거나 다운로드할 수 있습니다.

    - 사용자에게 AWS Management Console에 대한 액세스 권한이 필요한 경우, AWS Management Console access(콘솔 액세스)를 선택합니다. 이렇게 하면 각 신규 사용자에 대한 암호가 생성됩니다.

5. 권한 설정 페이지에서 이 신규 사용자 세트에 권한을 할당하는 방식을 지정합니다. 다음 세 가지 옵션 중 하나를 선택합니다.

    - Add user to group(그룹에 사용자 추가). 이미 권한 정책을 보유한 하나 이상의 그룹에 사용자를 할당하고자 하는 경우, 이 옵션을 선택합니다. IAM에 계정의 그룹 목록이 연결된 정책과 함께 표시됩니다. 기존의 보안 그룹을 한 개 이상 선택하거나 그룹 생성을 선택하여 새 그룹을 만들 수 있습니다. 자세한 내용은 섹션을 참조하세요IAM 사용자의 권한 변경

    - 기존 사용자에서 권한 복사(Copy permissions from existing user). 기존 사용자의 그룹 멤버십, 연결된 관리형 정책, 포함된 인라인 정책 및 기존 권한 경계를 새로운 사용자로 모두 복사하려면 이 옵션을 선택합니다. IAM에 계정의 사용자 목록이 표시됩니다. 보유한 권한이 새로운 사용자의 요구 사항과 가장 근접하는 사용자를 선택합니다.

    - 기존 정책을 직접 연결합니다. 이 옵션을 선택하여 계정의 AWS 관리형 또는 사용자 관리형 정책 목록을 봅니다. 신규 사용자에게 연결하려는 정책을 선택하거나 정책 생성을 선택하여 새 브라우저 탭을 열고 완전히 새로운 정책을 생성합니다. 자세한 내용은 IAM 정책 생성 절차의 4단계 섹션을 참조하세요. 정책을 생성하면 탭을 닫고 원래 탭으로 돌아와 신규 사용자에게 정책을 추가합니다. 그 대신에 그룹에 정책을 연결한 다음, 사용자들을 적절한 그룹의 구성원으로 만드는 것이 바람직한 모범 사례입니다.

## IAM 사용자 로그인

IAM 사용자로 AWS Management Console에 로그인하려면 사용자 이름과 암호 이외에 계정 ID 또는 계정 별칭을 제공해야 합니다. 관리자가 콘솔에서 IAM 사용자를 생성한 경우, 계정 ID 또는 계정 별칭이 포함된 계정 로그인 페이지의 URL, 사용자 이름 등을 비롯한 로그인 자격 증명을 전송했을 것입니다.

https://_AWS-account-ID or alias_.signin.aws.amazon.com/console

<script src="https://utteranc.es/client.js"
        repo="chojs23/comments"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
