---
layout: post
title: "S3 정적 사이트 배포"
author: Jason
featured: true
excerpt: AWS S3에서 React App 배포 자동화를 위해 필요한 내용을 번역해보았습니다.
---

> 아래 글은 [출처](https://gist.github.com/bradwestfall/b5b0e450015dbc9b4e56e5f398df48ff)의 글을 [저자](https://gist.github.com/bradwestfall)의 허락을 받아 번역한 내용입니다.

**아래에서 다룰 내용**
- S3에서 정적 웹 사이트 호스팅
- 웹 사이트는 SPA 일 수 있습니다 (모든 `Requests`가 `index.html`로 전달되야하는 경우)
- 무료 AWS SSL 인증서
- CDN 무효화를 사용한 배포


## 참고자료
- https://stormpath.com/blog/ultimate-guide-deploying-static-site-aws
- https://miketabor.com/host-static-website-using-aws-s3/
- http://www.mycowsworld.com/blog/2013/07/29/setting-up-a-godaddy-domain-name-with-amazon-web-services.html
- https://www.davidbaumgold.com/tutorials/host-static-site-aws-s3-cloudfront/#make-an-s3-bucket

## S3 버킷
- 도메인 이름으로 S3 버킷을 만듭니다 (예: `biz.lever.me`).
- **속성**에서 정적 웹 사이트 섹션을 클릭하십시오.
  - **이 버킷을 사용하여 웹 사이트 호스팅합니다**를 클릭하고 **인덱스 문서** 항목에`index.html`을 입력하십시오.
  - `http://biz.lever.me.s3-website.ap-northeast-2.amazonaws.com`과 같은 `Endpoint`가 생성됩니다.
- 그런 다음 **권한** 탭을 클릭 한 후 **버킷 정책**을 클릭하십시오. 아래 정책을 입력하십시오 :

```json
{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Sid": "AllowPublicRead",
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::biz.lever.me/*"
        }
    ]
}
```

`BUCKET_NAME(Resource)`을 확인 후 입력해주세요.
> 참고 : 버킷 이름을 반드시 도메인 이름 일 필요는 없습니다. 여러 글에서 그것이 필요하다고 읽었지만 그렇지 않습니다. AWS에서 와일드 카드 도메인을 사용하는 경우 와일드 카드 도메인을 사용할 때 도메인 이름에 점(.)이 표시되지 않는다는 것을 읽었습니다. 따라서 버킷의 이름을 무엇이든 지정할 수 있지만 와일드 카드 도메인을 사용하지 않는 경우 점(.)을 사용하면 작동합니다.

`index.html`을 업로드하면 `Endpoint`를 방문 할 수 있습니다

## CloudFront
- CloudFront 섹션으로 이동하여 **Create Distribution**을 클릭 한 다음 RTMP가 아닌 **Web**을 생성하십시오.
- **Origin Domain Name**에서 S3에서 이전에 생성한 버킷을 선택합니다.
- 이 지침의 순서는 SSL 인증서가 아직 설정되지 않은 것으로 가정합니다. 따라서 SSL에 관한 설정으로 아무것도하지 마십시오
- **Compress Objects Automatically**: `Yes`를 선택하십시오.
- **Alternate Domain Names (CNAMEs)**: 이 버킷에 해당하는 도메인 이름을 입력합니다.
- **Default Root Object**: `index.html`을 입력하십시오.
- **Create Distribution** 버튼을 눌러 생성합니다. 다음 화면에서 각 항목을 표 형식으로 보여줍니다. 방금 만든 것은 몇 분 동안 `진행 중` 상태로 진행됩니다.

배포판에는`dpo155j0y52ps.cloudfront.net`과 같은 도메인 이름이 있습니다. 이것은 DNS에 중요합니다 (아래 참조). 이 도메인을 복사하십시오.
  
## Route 53
- **호스트 영역**을 클릭하십시오.
- **레코드 세트**를 클릭하여`A` 레코드를 만듭니다.
  - 이것은 `biz.lever.me` 도메인이 CloudFront를 가리키는 레코드로 만듭니다.
  - **이름**에 `biz`를 입력합니다.
  - **별칭**을 선택하고 필드에 CloutFront 도메인을 붙여 넣습니다.
    - 이것은 `[some-random-number].cloudfront.net`과 같아야합니다. CloudFront 배포를 클릭하면 일반 탭에 `도메인 이름`레이블이 있습니다.
  - `레코드 세트 저장`을 클릭하십시오.

## HTTPS
AWS 콘솔에서 **Certificate Manager**로 이동하여 도메인 및 모든 하위 도메인에 대한 인증서를 요청하십시오.
이메일 또는 DNS를 통해 인증서를 확인해야합니다. 이메일로 확인하는 경우 AWS는 퍼블릭 DNS
소유자 정보를 검색하고 찾은 이메일을 최대 3 개까지 사용합니다 (도메인 소유권 정보가 퍼블릭 인 경우).
그러나 공개되지 않더라도 AWS는 이들을 사용합니다 (선택할 수 없음)

- `administrator@biz.lever.me`
- `hostmaster@mywebsite.com`
- `postmaster@mywebsite.com`
- `webmaster@mywebsite.com`
- `admin@mywebsite.com`

회사에서 `webmaster@`를 사용하는 경우 앱의 수명이 1000년이 되었을 것입니다(?).

**.io TLD의 경우 참고**: http://docs.aws.amazon.com/acm/latest/userguide/troubleshoot-iodomains.html

DNS를 통해 확인하기로 선택한 경우 AWS는 Route 53 DNS에 일부 CNAME 레코드를 추가하도록 요청하지만
인증서 관리자 내에서 (각 도메인 및 하위 도메인에 대해) 이를 수행할 수 있는 바로 가기 버튼이 있다는 것이 좋습니다.

확인이 완료되고 인증서가 "발급"되면 CloudFront로 돌아가서 이 도메인의 배포를 편집 할 수 있습니다.

- 배포를 클릭하고 다음 페이지 (일반 탭)에서 ** 편집 **을 클릭하십시오.
- **사용자 정의 SSL 인증서** 확인란을 선택하십시오.
- 인증서를 선택하고 저장하십시오. 텍스트 필드 모양은 인증서를 선택하기 위해 클릭하면 드롭 다운 메뉴입니다.
- 양식 작성이 완료되면 ** Behaviors ** 탭을 클릭하고 있어야하는 유일한 레코드를 편집하십시오.
- **HTTP를 HTTPS로 리디렉션**을 선택합니다. 저장을 클릭하십시오

## SPA
웹 사이트가 SPA 인 경우 파일이없는 경우에도 서버 (이 경우 S3)에 대한 모든 요청이 무언가를 반환하는지 확인해야합니다.
이것은 React (React Router가 있는)와 같은 SPA가 모든 요청에 ​​대해`index.html` 페이지를 필요로하기 때문에 `Not Found`
페이지와 같은 것들이 프론트 엔드에서 처리된다는 것입니다.

CloudFront로 이동하여 SPA 설정을 적용 할 배포를 클릭하십시오.
**오류 페이지** 탭을 클릭하고 새 오류 페이지를 추가하십시오. 다음 필드로 양식을 채우십시오.

- **HTTP Error Code**: 404
- **TTL**: 0
- **Custom Error Response**: Yes
- **Response Page Path**: `/index.html`
- **HTTP Response Code**: 200

## 배포
배포를 위해 CloudFront CDN의 파일이 변경되지 않았 음을 고려해야합니다.
새 파일을 S3에 업로드 할 경우 CDN의 `Edge` 서버에 배포되지 않으므로 웹 사이트가 업데이트되지 않습니다.
[자세히 읽기](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/ReplacingObjectsSameName.html).

CDN에서 파일을 무효화하려면 CloudFront의 **Invalidations** 기능을 사용해야합니다.
[자세히 읽기](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html).

AWS 콘솔의 CloudFront 배포 관리에는 **Invalidations**에 대한 탭이 있습니다.
모든 S3 파일을 무효화하기 위해 무효화 (`/*`값으로)를 수동으로 작성할 수 있습니다.
여기서 무효화 레코드는 일회성 무효화이며 새 파일을 배포 할 때마다 새로운 무효화를 수행해야합니다.

무효화와 함께 배포하려면 먼저 [AWS-CLI](https://aws.amazon.com/cli)를 설치해야합니다.
또한 액세스 키 및 비밀 액세스 키를 가진 AWS의 IAM 사용자가 있다고 가정합니다.

설치를 테스트하려면 다음을 수행하십시오.

```sh
aws --version
```

aws-cli를 구성하십시오.

```sh
aws configure --profile PICK_A_PROFILE_NAME
```

> `프로파일`을 사용하여 AWS-CLI를 구성하는 것이 가장 좋습니다. CLI를 사용하여 특정 시점에 여러 AWS 계정을 관리 할 수 ​​있기 때문입니다.
> `PICK_A_PROFILE_NAME`을 프로파일명으로 바꿔야합니다(무엇이든 가능).

다음 값을 입력하십시오:
```sh
AWS Access Key ID [None]: [AWS 액세스 키]
AWS Secret Access Key [None]: [AWS 비밀 액세스 키]
Default region name [None]: ap-northeast-2
Default output format [None]: json
```

그러면 `~/.aws/credentials`에 항목이 저장됩니다. AWS에 적합한 리전을 입력해야합니다. `json` 대신`text`로 응답을 받을 수도 있습니다.

컴퓨터의 기본값을 설정하려면 (모든 프로파일이 사용할 수 있도록) 지역 및 형식에 대한 마지막 두 가지 질문을 생략 할 수 있습니다.
기본 프로파일은`~/.aws/config`에 있습니다. 프로파일에서 region과 output을 생략하려면 `~/.aws/config`에 다음과 같이 존재해야합니다.

```ini
[default]
output = json
region = ap-northeast-2
```

이제 "실험적인" CloudFront 명령을 수행해야하므로 다음을 수행합니다.

```sh
aws configure set preview.cloudfront true
```

`~/.aws/config`에 더 많은 레코드가 생깁니다.

배포를 위해 지금 설정해야합니다.
Run:
```sh
aws s3 sync --acl public-read --profile YOUR_PROFILE_NAME --delete build/ s3://biz.lever.me
```

- `YOUR_PROFILE_NAME`을 자신의 것으로 바꾸십시오. 또한 이것은 업로드하려는 폴더가`build`라고 가정합니다.
- 이 명령은
  - 업로드 된 모든 파일이 공개용인지 확인 (`--acl public-read`)
  - 로컬 AWS 프로파일 (`--profile YOUR_PROFILE_NAME`)의 자격 증명을 사용하고 있는지 확인하십시오.
  - 로컬에 존재하지 않는 기존 S3 객체를 모두 제거합니다 (`--delete`).

배포가 확인되고 성공한 후 무효화해야합니다.

```sh
aws cloudfront --profile YOUR_PROFILE_NAME create-invalidation --distribution-id YOUR_DISTRIBUTION_ID --paths '/*'
```

- `YOUR_PROFILE_NAME` 및`YOUR_DISTRIBUTION_ID`를 자신의 것으로 바꾸십시오. 배포 ID는 AWS 콘솔의 CloudFront seciton에서 찾을 수 있습니다.
- 무효화가 작동하면 배포를 클릭 한 후 **Invalidations** 탭에서 무효화 기록을 볼 수 있습니다.

더 쉽게하려면`package.json`에 추가하십시오 :

```json
{
  "scripts": {
    "deploy": "aws s3 sync --acl public-read --profile XYZ --delete  build/ s3://biz.lever.me && npm run invalidate",
    "invalidate": "aws cloudfront --profile XYZ create-invalidation --destribution-id XYZ --paths '/*'"
  }
}
```

`XYZ`는 교체해야 할 부분입니다. 이제 `npm run deploy`를 실행하여 배포 한 다음 무효화 할 수 있습니다

건배!
