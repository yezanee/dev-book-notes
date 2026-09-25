## 2-1 Hello API 자바스크립트 구현

**목표와 구성**

- 쿼리 파라미터로 이름을 받아 `Hello, 이름!`을 돌려주는 HTTP API를 서버리스 스택으로 만든다. 먼저 자바스크립트로 구현하고, 이후 타입스크립트로 다시 작성하며 차이를 비교하는 흐름이다.
- 구성 요소는 세 가지다. HTTP 이벤트를 받는 API Gateway, 응답을 만드는 Lambda, Lambda 로그를 모으는 CloudWatch Logs.
- 웹 콘솔로 직접 자원을 만들고 연결할 수도 있지만 번거롭기 때문에 Serverless Framework를 사용한다.

**프로젝트 생성**

- `sls create --template aws-nodejs --name hello-api-js --path hello-api-js`로 템플릿에서 프로젝트를 만들면 `handler.js`와 `serverless.yml`이 생긴다.
- `serverless.yml`은 스택을 관리하는 선언 파일로 클라우드 서비스, 지역, 런타임, 함수와 이벤트, 권한, 기타 자원을 선언한다. `handler.js`는 여기에 선언된 `hello` 함수의 소스 코드다.
- 템플릿은 자주 바뀌며, 선언 예시가 `#` 주석으로 들어 있다.

**serverless.yml 주요 속성 10가지**

1. `service`: CloudFormation에 등록될 스택 이름. `-name` 인자로 설정된다.
2. `frameworkVersion`: 잘못된 버전의 Serverless Framework 사용을 막기 위해 명시한다.
3. `provider`: 클라우드 업체와 서비스 전반 설정. AWS 외에 GCP, Azure 등도 지원하며 여기서는 `name: aws`.
4. `runtime`: Lambda 런타임(`nodejs14.x`). `functions` 블록에서 함수별로 재정의할 수 있다.
5. `stage`: 배포 스테이지. dev에서 테스트 후 production으로 배포하는 식으로 구분한다. 기본값은 dev.
6. `region`: 배포 지역. 한국 대상이면 `ap-northeast-2`. 기본값은 `us-east-1`.
7. `iam.role.statements`: Lambda가 AWS 자원에 접근하기 위한 권한. IAM 문법을 따르며, runtime과 달리 함수별로 정의할 수 없다. 함수별 분리가 필요하면 `serverless-iam-roles-per-function` 플러그인을 쓴다.
8. `environment`: `process.env`로 접근하는 환경 변수. 함수별로도 지정 가능하다.
9. `functions`: 함수의 진입점(handler)과 이벤트를 연결하는 블록. `handler.hello`는 `handler.js`의 `hello` export를 뜻한다. 이벤트 종류로 httpApi, websocket, s3, schedule, sns, stream, cloudwatchEvent, cloudwatchLog, alb 등이 있고 httpApi는 path와 method를 지정한다.
10. `resources`: CloudFormation 선언을 넣는 곳. S3 버킷 같은 자원을 스택에 함께 포함해 관리할 수 있다.
- 속성이 매우 많으므로 전체 설정 예제를 참고하는 게 좋고, 최근에는 `serverless.ts`로 VSCode 자동 완성을 받을 수 있다.

**handler.js 코드 해설**

- 템플릿 코드는 HTTP 요청을 받으면 메시지와 함께 event를 그대로 응답한다.
- `'use strict'`는 선언하지 않은 변수 사용을 막아 오타로 인한 문제를 방지한다.
- `module.exports`는 Node.js 모듈 문법으로, `require("./handler").hello`로 접근 가능하며 이 경로를 `functions.hello.handler`에 적은 것이다.
- 화살표 함수는 this 바인딩 차이가 있지만 여기서는 코드를 짧게 쓰려는 용도다.
- HTTP 이벤트 처리 Lambda는 `event`를 받아 `Promise<{statusCode, body, ...}>`를 반환한다. 과거에는 callback을 썼지만 지금은 Promise 반환을 선호하며, `async`로 선언하면 값을 그대로 반환해도 런타임이 Promise로 감싼다.
- `statusCode` 200은 정상 처리, `body`는 string이어야 하므로 JSON은 `JSON.stringify`로 문자열화한다.
- Lambda는 S3, SQS 등 다른 이벤트로도 실행되며, 이벤트 소스에 따라 event 형태와 반환값 형태가 달라진다. 템플릿 코드는 반환값 형태로 HTTP용임을 알 수 있지만 실제로는 이벤트가 연결되어 있지 않다.

**이벤트 연결과 지역 설정**

- 주석 처리된 `events.httpApi`를 복구해 `path: /hello`, `method: get`으로 설정하면 `GET /hello`로 함수를 실행할 수 있다.
- 기본 지역 us-east-1은 서울에서 멀어 응답이 느리므로 `provider.region`을 `ap-northeast-2`로 지정한다.

**배포 (`sls deploy`)**

- CloudFormation으로 자원을 할당하므로 간단한 예제도 약 2분 걸린다. 할당 대상은 API Gateway, Lambda, 스택 메타데이터용 S3, CloudWatch, IAM.
- 스택 이름은 service와 stage를 합친 `hello-api-js-dev`.
- 자원은 11개로, API Gateway나 Lambda 같은 큰 단위가 아니라 세부 단위로 할당되어 생각보다 많다. 목록은 다음과 같다: LambdaFunction, LambdaPermission, LambdaVersion, LogGroup, HttpApi, HttpApiIntegration, HttpApiRoute, HttpApiStage, IAM Role, ServerlessDeploymentBucket(S3).

**코드 업로드 방식**

- Lambda 코드는 압축 해제 기준 최대 250MB이며, 압축 코드를 S3에 올리고 Lambda가 가져오게 한다.
- 3MB 미만이면 인라인 코드로 들어가 콘솔 편집기에서 바로 수정할 수 있고, S3에서 불러오는 시간이 줄어 기동이 조금 더 빠르다. 예제는 인라인으로 배포된다.

**호출과 event 구조**

- 주소 형식은 `https://API_ID.execute-api.AWS_REGION.amazonaws.com/PATH`. API_ID는 배포 시 자동 생성된다.
- curl 응답에서 HTTP 이벤트(input) 구조를 볼 수 있다: version 2.0, routeKey, rawPath, rawQueryString, headers, requestContext(accountId, apiId, domainName, http.method/path/sourceIp 등, requestId, stage, time), isBase64Encoded.
- 쿼리 파라미터나 인증 정보를 얻으려면 이 구조에 익숙해야 하는데, 직접 출력해 보는 것보다 타입스크립트의 자동 완성이 훨씬 편하다.

**로그 확인 (`sls logs -f hello`)**

- 기본으로 최근 10분 로그를 보여준다.
- 첫 번째 호출에만 `init` 시간이 찍힌다. 첫 기동에는 초기화가 필요하고, 이후 요청은 초기화된 Lambda를 재사용하기 때문이다.
- 코드가 커질수록 init이 길어져 첫 요청 지연이 늘어나므로 Lambda는 가볍게 작성하고 초기화 시간을 주기적으로 모니터링해야 한다.

**본래 목표 구현**

- `event.queryStringParameters.name`으로 name을 읽는다. 파라미터가 없으면 `queryStringParameters` 자체가 undefined이고, name이 없으면 name이 undefined이므로 두 경우 모두 404 Not Found를 반환한다. 있으면 `Hello, name!`을 body로 반환한다.
- 코드만 바뀐 경우 `sls deploy function -f hello`로 Lambda 코드만 갱신해 훨씬 빠르다(약 1초). 단 API 주소를 보여주지 않으므로 필요하면 `sls info`를 쓴다.
- 결과: 파라미터 없이 호출하면 Not Found, `?name=lacti`면 `Hello, lacti!`.

**삭제 (`sls remove`)**

- 약 30초 소요되며 할당한 모든 자원을 회수한다.
- 서버리스 스택은 안 쓰면 비용이 거의 없지만 배포 가능 자원 개수에 한도가 있으므로 테스트 후 바로 지우는 게 좋다. S3처럼 저장 용량에 비용이 드는 경우는 더욱 그렇다.

## 2-2 Hello API 타입스크립트 구현

**도입 배경**

- 처음 서버리스를 할 때 가장 생소한 건 Lambda에 전달되는 이벤트와 반환 타입을 모른다는 점, 그리고 Serverless Framework 옵션을 매번 찾아봐야 한다는 점이다.
- `serverless.yml`은 정적 파일이라 오타로 시간을 낭비하기 쉽고 동적으로 값을 만들기 어렵다.
- 타입스크립트로 해결되는 것: (1) event의 필드를 타입 정보로 쉽게 알 수 있음, (2) `serverless.ts`로 선언을 재사용 가능하게 정리하고 옵션에 빠르게 익숙해질 수 있음.

### 2-2-1 템플릿으로 시작

- `sls create --template aws-nodejs-typescript`로 만들면 훨씬 많은 파일이 생긴다: `serverless.ts`, `src/functions/hello/`(handler.ts, index.ts, mock.json, schema.ts), `src/libs/`(api-gateway.ts, handler-resolver.ts, lambda.ts), tsconfig.json, tsconfig.paths.json 등.
- 웹팩, 루트 기준 참조를 위한 tsconfig.paths, JSON 검증용 schema, 미들웨어 패키지 middy가 미리 설정되어 있다.
- 좋은 기능이지만 설명할 게 너무 많아 이 템플릿은 쓰지 않고, 자바스크립트 코드에 타입스크립트와 웹팩 설정을 단계별로 추가한다.

### 2-2-2 타입스크립트 적용

- `hello-api-js`를 복사해 `hello-api-ts`를 만든다.
- 이전에는 전역 설치한 Serverless Framework만 써서 package.json이 없었지만, 이번에는 의존성이 필요하므로 `npm init -y`로 생성한다.
- `npm install --save-dev typescript @types/node@14`로 설치한다. Node.js 타입은 런타임 버전(14)에 맞추고, 컴파일 시에만 필요하므로 devDependencies에 둔다. 이후 `npx tsc`로 컴파일러를 실행한다.
- 타입스크립트는 자바스크립트의 상위 집합이라 확장자만 바꿔도 되지만 좋은 방법은 아니다. `handler.ts`로 바꿔 컴파일하면 원래보다 훨씬 복잡한 JS가 나오는데, async를 지원하지 않는 낮은 버전 런타임에 맞춰 변환했기 때문이다.
- Lambda는 Node.js 14를 쓰므로 낮은 버전용 코드는 불필요하고, 코드가 많아지면 첫 기동이 길어질 수 있다. `npx tsc --init`으로 tsconfig.json을 만들고 target을 ES2019로 맞춘다.

**tsconfig.json 옵션**

1. `target`: 변환할 자바스크립트 표준 버전(ES2019).
2. `module`: 모듈 방식. Node.js 14에서 동작하도록 commonjs.
3. `strict`: 정적 검사 항목을 늘려 타입 문제를 실행 전에 발견.
4. `esModuleInterop`: 여러 모듈 사양 간 호환. 다양한 라이브러리 사용 시 켜두는 게 좋다.
5. `skipLibCheck`: 외부 패키지 타입 검사 생략으로 개발 시 자원 절약.
6. `forceConsistentCasingInFileNames`: 대소문자 구분 없는 파일시스템(윈도우 등)에서 잘못된 참조 방지.
- 전체 옵션은 typescriptlang.org/tsconfig 참고.

**타입 적용**

- tsconfig.json이 있는 상태에서 `npx tsc`를 실행하면 strict 때문에 `TS7006: Parameter 'event' implicitly has an 'any' type` 오류가 난다. 다만 치명적 오류는 아니라 handler.js는 생성되고, 내용이 기존 JS와 같아 설정이 의도대로 적용됐음을 확인할 수 있다.
- `npm install --save-dev @types/aws-lambda`를 설치하고 `APIGatewayProxyHandlerV2` 타입을 사용한다. API Gateway에서 HTTP 이벤트를 받는 함수용 타입이다.
- 변경점 3가지: (1) TS가 strict를 검사하므로 `use strict` 삭제, (2) `module.exports.hello` 대신 `export const hello`, (3) `aws-lambda`에서 타입을 import해 hello의 타입으로 지정.
- 함수 타입은 인자와 반환값 타입으로 정해지므로 event와 반환값 타입이 추론되고 event 멤버 자동 완성이 동작한다(body, cookies, headers, pathParameters, queryStringParameters, rawPath, requestContext, routeKey, stageVariables, version 등).

**패키징 문제**

- 구분을 위해 `service`를 `hello-api-ts`로 바꾼다.
- `sls package` 후 zip을 풀어보면 실행에 필요한 handler.js뿐 아니라 handler.ts, tsconfig.json, package.json, package-lock.json, `node_modules/@types` 등이 함께 들어 있다.
- 첫 기동 최적화를 위해 실행에 필요한 코드만 넣어야 한다. 의존 패키지에는 다른 모듈 형태용 배포판, 원본, README 등이 섞여 있고, @types 같은 컴파일 타임 전용 패키지는 빌드 후 제외해야 한다.
- `serverless.yml`의 package 설정으로 포함/제외를 명시할 수 있지만 번거롭기 때문에 웹팩을 사용한다.

### 2-2-3 웹팩 적용

- 웹팩은 결과물을 번들링하는 도구로, 필요한 코드만 추려 파일 형식별로 빌드하고 chunk로 묶어 최소화(minify)나 난독화를 돕는다. 로더를 추가해 여러 형식을 지원하며 ts-loader로 타입스크립트를 처리한다.
- 웹팩 4부터 zero-configuration이 대부분 가능하지만, Serverless Framework와 쓰려면 `serverless-webpack` 플러그인과 `webpack.config.js`가 필요하다.
- `sls plugin install --name serverless-webpack`으로 설치하면 `plugins` 블록에 추가되고, serverless-webpack과 webpack이 devDependencies에 들어간다. 버전 미지정 시 최신이 설치되며 2022년 8월 기준 각각 5.8.0, 5.74.0.
- 상세 정보와 TS 예제는 serverless-heaven/serverless-webpack GitHub 저장소 참고.
- `npm install --save-dev ts-loader` 설치 후 `webpack.config.js` 작성.

**webpack.config.js 옵션**

- 파일 자체는 JS로 작성한다(import 대신 require). TS로도 가능하지만 플러그인이 JS 설정을 기본 가정하므로 추가 설정이 필요하다. 전체 옵션은 webpack.js.org/configuration.
- `mode`: `slsw.lib.webpack.isLocal`이면 development, 아니면 production. isLocal은 `sls invoke local`이나 serverless-offline으로 로컬 테스트할 때 설정된다.
- `entry`: 빌드 시작점. 함수를 하나의 파일로 묶을 수도, `package.individually`로 함수별로 나눌 수도 있어 진입점이 Serverless 설정에 따라 바뀌므로 `slsw.lib.entries`를 쓴다.
- `devtool`: 소스맵 생성 옵션. 상용 빌드는 정보가 많은 `source-map`, 개발 빌드는 재빌드가 빠른 `eval-cheap-module-source-map`을 주로 쓴다. 소스맵 활용 패키지 때문에 결과물이 커지므로 첫 기동을 극단적으로 최적화해야 하면 아예 생성하지 않기도 한다. 이 예제는 source-map 사용.
- `resolve`: 처리할 확장자 지정(.mjs, .json, .ts, .js).
- `output`: 관례대로 `.webpack` 디렉터리에 `[name].js`로 생성, `libraryTarget`은 Node.js 표준인 commonjs2(commonjs에 export 문법이 추가된 형태).
- `target`: 실행 환경. Lambda의 Node.js이므로 node.
- `module`: `.ts`에 ts-loader 적용, `node_modules`는 제외.

**결과**

- `sls package` 시 웹팩이 handler.ts를 ts-loader로 컴파일하고 minify한 뒤 zip으로 묶는다.
- zip 안에는 handler.js(312B)와 handler.js.map(857B)만 남는다. 타입 정보 파일을 빌드 단계에서만 쓰고 제외했기 때문이며, 이후 다른 패키지를 써도 필요한 코드만 포함되어 배포 코드 양을 최적화할 수 있다.

### 2-2-4 source-map 적용

- devtool로 .map 파일을 만들어도 .js에는 디버깅 정보가 없어, 둘을 함께 읽는 디버깅 상황이 아니면 .map을 활용하지 못한다. 예를 들어 Lambda에서 예외 스택 트레이스가 원본이 아닌 변환된 코드 기준으로 찍혀 원인 파악이 어렵다.
- `npm install --save source-map-support`(런타임 의존성)로 설치하고 handler.ts에 `import "source-map-support/register"`를 추가한다.
- `webpack.config.js`에 `stats: "normal"`을 넣으면 빌드 시 파일별 용량 등 지표를 볼 수 있다.
- 결과: handler.ts는 444B인데 handler.js는 33.3KiB로 JS 예제(312B)의 약 100배다. source-map-support(19.8KiB)와 source-map(99.4KiB)이 그대로였다면 약 120KiB였겠지만 웹팩이 필요한 코드만 포함했다.
- 그래도 첫 기동에 불리하므로 상황에 따라 스택 트레이스를 포기하고 이 패키지를 안 쓰는 선택도 가능하다.

### 2-2-5 serverless.ts 사용

- 함수 코드뿐 아니라 외부 환경 설정도 타입의 도움을 받는 방법이다. 정적 YAML과 달리 로직을 넣을 수 있어 환경 변수 기반 분기나 AWS CDK로 자원 정의도 가능하다.
- Serverless Framework는 yml→ts 변환 기능이 없으므로 onlineyamltools 같은 웹 서비스나 yq로 JSON 변환 후 serverless.ts에 붙여넣는다.
- `npm install --save-dev @serverless/typescript`로 선언 타입을 설치한다.
- yml과 ts가 함께 있으면 yml이 우선이므로 ts 작성 후 yml을 삭제해야 한다.
- `@serverless/typescript`의 `AWS` 타입으로 `config`를 선언하고 `export = config`. VSCode Format Document로 JS 객체 형태로 정리할 수 있다.
- 이점: 자동 완성과 타입 검증, 그리고 각 함수에서 functions 설정을 export해 serverless.ts에서 import하는 방식이 가능해 선언과 구현의 괴리가 줄고 응집도가 높아지며 누락을 알아차리기 쉽다.
- `npm install --save-dev ts-node`가 필요하며, 설치 후 `sls package`가 잘 되면 설정이 올바른 것이다.

## 2-3 상용 서비스 고려

**흐름과 장점**

- 사용자 요청 → API Gateway → 연결된 Lambda 실행 → 결과를 API Gateway가 응답, 실행 중 로그는 CloudWatch Logs로 전달된다.
- 인프라를 크게 신경 쓰지 않아도 개발과 상용 환경 모두 잘 동작한다: 인증서 없이 https, 서버 확장 고민 없이 요청량에 따라 Lambda 동시 실행, 디스크 고갈 걱정 없는 로그, 배포 시 서비스 중단 없음(갱신 순간까지 이전 버전이 처리).
- 향후 NAME 에러 처리와 테스트, 스택 확장에 집중할 수 있다.
- 한계: 요청량에 따라 요금이 빠르게 증가하고, 서비스마다 한도(quotas)가 있다. 모르고 있으면 어느 날 트래픽 증가에 대응하지 못하므로 한도를 알아야 한다.

### 2-3-1 서비스 한도

- 한도 내에서는 자동 확장을 위한 인프라 고민이 필요 없지만, 그렇기에 기본 한도를 알아야 한다. 공식 문서는 방대하니 필요할 때 조금씩 익힌다.
- 기본 확인 대상(괄호는 조절 가능 여부):
    - API Gateway 요청: 10,000요청/초 (계정-지역), 5,000요청 버킷 (가능)
    - API Gateway 최대 요청 시간: 30초 (불가)
    - API Gateway 최대 요청 크기: 10MB (불가)
    - Lambda 동시 실행 수: 1,000 (가능)
    - Lambda 최대 요청 크기: 6MB (불가)
    - Lambda 최대 수행 시간: API Gateway 통합 시 29초 (불가)
    - CloudWatch Logs 로그 스트림 생성 요청: 50요청/초 (계정-지역) (가능)
    - CloudWatch Logs 로그 스트림 업로드 요청: 스트림마다 5요청/초 (가능)
    - CloudWatch Logs 최대 로그 크기: 256KB (불가)
    - CloudWatch Logs 최대 요청 크기: 1MB (불가)
- 전체 내용은 API Gateway, Lambda, CloudWatch Logs 각각의 할당량 문서에서 확인한다.
- 문서가 지역별 값을 명시하지 않거나 미국 기준이라 서울과 다를 수 있고, 글로벌 배포 시 지역별 한도 차이로 문제가 날 수 있다. 정확한 값은 서비스 한도 페이지에서 확인하되, 그 페이지엔 설명이 없으니 먼저 문서로 의미를 익혀야 한다.

### 2-3-2 API Gateway의 한도

- 토큰 버킷 알고리즘: 일정 시간마다 토큰이 채워지고 처리 단위마다 소모, 부족하면 거절한다. 기본값은 초당 10,000요청 허용, 버킷 5,000. 즉 5,000 버킷에 1ms마다 10요청씩 회복된다고 보면 된다.
- 예시:
    1. 1ms에 10요청씩 일정하면 모두 성공.
    2. 첫 1ms에 10,000요청이 몰리면 5,000만 처리되고 나머지는 429 Too Many Requests.
    3. 첫 1ms에 5,000, 101ms에 1,000이 오면 모두 성공. 100ms 동안 1,000토큰이 회복되기 때문.
- 이 한도는 계정-지역 단위라 같은 지역의 모든 API Gateway가 공유한다. 한 서비스 요청이 폭주하면 다른 서비스들도 429를 받는다.
- 대응: API별 throttle, 필요 시 서포트 티켓으로 상향 요청. 10,000요청/초는 쉽게 도달하므로 상용 서비스마다 계정을 분리하고 지속 모니터링으로 필요할 때 바로 티켓을 만들 수 있어야 한다.

### 2-3-3 API Gateway 통합 Lambda의 한도

- **실행 시간**: Lambda를 직접 호출하거나 SQS, S3 이벤트로 실행하면 최대 900초지만, API Gateway 통합 시 30초 내에 끝나야 한다. 30초 근처에서 끝나면 초과될 수 있어 29초 내 완료를 권장한다.
- **요청 크기**: 반대로 Lambda 때문에 API Gateway가 제한되기도 한다. API Gateway는 10MB까지 받지만 Lambda는 6MB까지라 실질 한도는 6MB. 더 큰 요청은 S3에 업로드한 뒤 Lambda가 임시 파일로 불러와 처리한다.
- **동시 실행**: 인스턴스를 동시에 최대 1,000개(스레드 풀 1,000개에 비유). 요청 하나에 10ms라면 인스턴스 하나가 초당 100건, 1,000개면 초당 100,000건 처리.
- 이 역시 계정-지역 단위라 여러 스택이 1,000개를 나눠 쓴다. 서비스마다 계정을 분리하고 동시 실행 수치를 모니터링해 사용량을 예측하는 게 좋다.
- 한도를 넘으면 이후 실행은 실패하므로 미리 상향 티켓을 생성해야 한다.

### 2-3-4 CloudWatch Logs의 한도

- 로그를 이벤트마다 즉시 보내지 않고 수행이 끝난 뒤 배치로 업로드한다. Lambda에 대응하는 로그 그룹 밑의 로그 스트림에 `PutLogEvents` API로 전달하며, 로그 그룹은 Lambda 배포 시 함께 생성된다.
- 로그 이벤트는 최대 256KB, PutLogEvents 요청은 최대 1MB. 즉 console.log 메시지 하나가 256KB를 넘을 수 없고, 그런 메시지가 4개를 넘으면 여러 번 호출된다.
- 요청은 스트림당 초당 5회. 256KB 로그를 한 번에 총 20번(배치당 4개 × 초당 5회)을 넘게 남기면 초과분은 업로드되지 못할 수 있다.
- 동시 실행되는 Lambda는 각각 다른 로그 스트림에 기록하므로 스트림 생성 자체도 고려해야 한다. 계정-지역 내 초당 50개 생성이 허용되어, 동시에 기동되는 50개 Lambda까지는 새 스트림을 할당할 수 있다.
- **인스턴스 재사용**: 두 번째 호출이 빠른 이유도 이것이다. 1ms짜리 함수를 초당 20번 수행해도 로그가 모두 전달되는데, 수행마다 바로 전송하지 않고 여유가 생길 때 모아서 올리기 때문이다. 재사용된 인스턴스의 로그는 같은 스트림에, 새 인스턴스의 로그는 새 스트림에 남는다. 즉 인스턴스 초기화 시 스트림이 정해지고 이후 비동기로 모아서 전송하는 구조로 추측된다.
- 그래서 로그 전송 제한을 투명하게 계산하기 어렵다. 필요한 로그만 남도록 정리해 유실을 막고, 필요하면 한도 상향 티켓을 요청한다.

### 2-3-5 운영 전략

- AWS가 한도를 두는 이유: 실수나 공격으로 과도한 요청이 들어올 때 무한히 확장하면 의도치 않은 과도한 비용이 생기므로, 적당한 수준으로 두고 티켓으로 조정하게 한다.
- 조정 불가 한도(실행 시간, 로그 전송 등)도 대부분 문제가 되지 않는다. HTTP 처리에 30초 이상 걸리면 안 되고, 무의미한 로그를 여과 없이 모으는 건 의미도 없고 비용만 든다.
- 마세라티 문제: 초반부터 백만 명 동시 접속을 걱정하는 것. 확장성, 효율성, 가용성보다 빠른 출시가 중요하며 필요할 때 고민하면 된다.
- 서버리스는 인프라 문제가 어느 정도 해결된 상태에서 시작하므로 비즈니스 로직에 집중할 수 있다. 처음부터 대규모 트래픽은 무리지만 모니터링하다가 필요한 시점에 한도를 올리면 높은 트래픽에도 대응할 수 있다.

## 2-4 모니터링

- CloudWatch의 지표와 로그로 정상 동작 여부, 오류 시점의 로그, 요청량에 따른 확장 판단을 할 수 있고, 지표 조건으로 경보를 걸어 메일이나 SMS로 알릴 수 있다.
- Lambda 로그 그룹은 자동 생성되지만 API Gateway 액세스 로그처럼 별도 옵션이 필요한 로그도 있다.

### 2-4-1 Lambda의 로그 확인

- 코드에서 반환하는 메시지를 `console.info`로 출력하고 `sls deploy function -f hello`로 갱신, curl로 몇 번 호출 후 몇 초 기다린 뒤 `sls logs -f hello`로 보면 INFO 레벨로 찍힌다.
- console의 적절한 레벨 함수를 고르면 그대로 출력된다. 별도 로깅 라이브러리도 쓸 수 있지만 첫 기동에 영향이 없도록 가벼운 것이나 웹팩으로 불필요한 node_modules가 빠지는 것을 골라야 하며, 대부분 기본 console이 가장 좋다.
- **관리 콘솔**: sls logs는 스트림을 지정할 수 없어 동시 실행 로그를 찾기 어렵다. CloudWatch → 로그 그룹 → `/aws/lambda/hello-api-ts-dev-hello` → 로그 스트림 목록(마지막 이벤트 시간 표시)으로 들어가며, 대상이 많으면 Search all로 필터링 후 해당 스트림으로 이동한다. 스트림 안에서 입력 필드와 시간 버튼으로 검색할 수 있다.
- **VSCode AWS Toolkit**: AWS EXPLORER 탭 → CloudWatch Logs → 로그 그룹 우클릭 → View Log Stream → 스트림 선택 시 편집창에 이벤트 목록이 뜨고, VSCode 검색 기능을 쓸 수 있어 개발 중에는 sls logs보다 편하다.
- **비용 주의**: 로그 조회 API 자체는 과금이 없지만 전송 트래픽에 EC2 데이터 송신 비용이 발생하므로 대량 로그를 너무 자주 조회하면 예상 못한 요금이 나올 수 있다.

### 2-4-2 API Gateway의 로그 확인

- 간단한 설정으로 HTTP 요청 액세스 로그를 CloudWatch Logs에 남길 수 있다. 단 공식 문서상 다음 경우에는 지표와 로그가 남지 않을 수 있다:
    1. 413 Request Entity Too Large
    2. 429 Too Many Requests가 과도하게 발생
    3. 사용자 지정 도메인에 연결된 API가 없어 4XX 발생
    4. 내부 오류로 5XX 발생
- `serverless.ts`의 `provider.logs.httpApi`를 true로 하거나 `format`으로 원하는 형태를 지정한다.
- 기본 형식은 요청 ID, IP, 요청 시간, HTTP 메서드, routeKey, 응답 코드, 프로토콜, 응답 길이를 JSON으로 남긴다.
- 대부분 액세스 로그는 큰 의미가 없고 정상 여부나 트래픽 확인엔 지표가 더 유용하다. 하지만 인증을 추가하면 Authorizer Lambda가 요청을 거절할 때 그 수행 로그가 CloudWatch Logs에 남지 않아 디버깅이 어렵다. 이때 `$context.error.message`나 `$context.authorizer`를 액세스 로그에 남기면 큰 도움이 된다(4장에서 상세히 다룸).
- 이 변경은 특정 함수가 아닌 provider 수준이므로 `sls deploy function -f`로는 반영되지 않고 `sls deploy`로 스택 전체를 배포해야 한다.
- `sls logs`는 함수에 연결된 로그만 조회하므로 액세스 로그는 볼 수 없다. 로그 그룹 이름은 `/aws/http-api/hello-api-ts-dev`(서비스 이름에 stage를 붙인 형태)이며, 콘솔이나 AWS Toolkit으로 Lambda 로그와 같은 방법으로 조회한다. 책에서는 Toolkit 방식만 소개한다.

### 2-4-3 Lambda의 지표

- 로그는 함수가 의도대로 동작했는지, 어떤 상태와 오류였는지 볼 때 쓰고, 전반적으로 올바르게 동작하는지와 한도 상향 필요성은 수행 지표로 검토한다.
- 지표는 총 3가지 유형이며 Lambda 기능이 확장될 때마다 늘어난다. 모두 항상 유용하진 않지만 한 번 보고 기억해두면 필요할 때 쓸 수 있다.

### 2-4-4 Lambda의 호출 지표 (이후 내용 계속)

- 호출 결과에 따라 0 또는 1의 값을 가지며(성공이면 Errors 0, 오류면 1), 그래서 Sum 통계로 봐야 한다.
- **Invocations**: 성공 실행과 함수 오류 실행을 포함한 실행 횟수. 호출 요청이 제한(throttle)되거나 다른 호출 오류가 나면 기록되지 않으며, 요금이 청구되는 요청 수와 같다.
- **Errors**: 함수 오류가 발생한 호출 수. 코드의 예외와 Lambda 런타임 예외(시간 초과, 구성 오류 등)를 포함한다. 오류율은 Errors ÷ Invocations. 타임스탬프는 오류 발생 시점이 아닌 함수 호출 시점을 반영한다.
- **DeadLetterErrors**: 비동기 호출에서 배달 못한 편지 대기열(DLQ)로 이벤트를 보내려다 실패한 횟수. 권한 오류, 잘못 구성된 리소스, 크기 제한 등으로 발생한다.
- **DestinationDeliveryFailures**: 비동기 호출에서 Lambda가 대상(destination)에 이벤트를 보내려다 실패한 횟수. 권한 오류, 잘못 구성된 리소스, 크기 제한 등으로 발생한다.
- **Throttles**: 제한된 호출 요청 수. 모든 인스턴스가 처리 중이고 확장할 동시성이 없으면 `TooManyRequestsException`으로 추가 요청을 거부한다. 제한된 요청과 기타 호출 오류는 Invocations나 Errors에 잡히지 않는다.
- **ProvisionedConcurrencyInvocations**: 프로비저닝된 동시성에서 함수 코드가 실행된 횟수.
- **ProvisionedConcurrencySpilloverInvocations**: 프로비저닝된 동시성을 모두 쓰는 중이라 표준 동시성에서 실행된 횟수.

**어떤 지표를 주로 보나**

- 보통 Invocations, Errors, Throttles를 본다. 얼마나 실행됐고, 그중 얼마나 에러였고, 가용 Lambda가 없어 몇 건이 실패했는지 알 수 있기 때문이다.
- Errors가 높으면 함수에 문제가 있는 것이므로 로그로 추적한 뒤 수정본을 배포하거나 정상 동작하던 이전 버전을 배포한다.
- Throttles는 잔여 인스턴스가 없어 실행이 거절된 것, 즉 이미 서비스 장애가 났다는 뜻이다. 이 지표가 보이기 전에 한도 상향을 요청하는 게 바람직하다.
- 그런데 Invocations는 재사용된 인스턴스까지 합산하므로 동시 실행 수치를 정확히 알기 어렵다. 한도를 사전에 모니터링하려면 동시성 지표의 ConcurrentExecutions를 써야 한다.
- DeadLetter 기능은 호출 실패 시 재시도해 신뢰성을 보장하는 기능이다. 웹 서비스용 Lambda는 Lambda 자체 재시도보다 HTTP 요청 수준 재시도로 개발하므로, AWS 자원 간 이벤트를 엮는 Lambda의 신뢰성을 확보하는 경우가 아니면 DeadLetterErrors와 DestinationDeliveryFailures는 안 봐도 된다.
- 프로비저닝된 동시성은 첫 기동 지연을 줄이려 미리 준비된 Lambda를 확보하는 기능이다. 하지만 웹 요청 수준의 첫 기동 지연은 100ms 수준이고 대부분은 요청이 많아 이미 warm-up 상태일 것이므로 안 써도 될 수 있으며, 그렇다면 관련 두 지표도 볼 필요가 없다.

### 2-4-5 Lambda의 성능 지표

- 처리 세부에 대한 시간 정보를 제공한다. 예를 들어 Duration은 수행 시간(ms)이므로 Average나 Max로 본다.
- Duration은 통계 왜곡을 막기 위해 특이값을 제외하는 백분위수 통계를 지원하며, `pNN.NN` 형식(소수점 두 자리까지)으로 지정한다.
- **Duration**: 함수 코드가 이벤트를 처리하는 데 걸린 시간. 청구되는 호출 시간은 가장 가까운 ms로 반올림된 Duration 값이다.
- **PostRuntimeExtensionsDuration**: 함수 코드 완료 후 런타임이 확장(Extensions) 코드를 실행하는 데 쓴 누적 시간.
- **IteratorAge**: 스트림에서 읽는 이벤트 소스 매핑에서 마지막 레코드의 경과 시간. 스트림이 레코드를 받은 시점부터 이벤트 소스 매핑이 함수로 보낸 시점까지다.
- 대부분 Duration으로 충분하다. 단 이 값에는 첫 기동 시 인스턴스가 첫 실행을 준비하는 시간이 포함되지 않는다.
- Lambda 확장을 쓰면 실행 시간 부담을 PostRuntimeExtensionsDuration으로 측정한다. 확장은 직접 개발하기도 하고, Lumigo나 Epsagon 같은 파트너사 모니터링 도구 설정 과정에서 쓰이기도 한다.
- IteratorAge는 Kinesis나 SQS처럼 스트림 기반 입력을 만드는 서비스와 연동할 때 쓴다. 예컨대 SQS 이벤트를 일정 개수 모아 배치 처리하는 Lambda를 이벤트 소스 매핑으로 구성했을 때 연결이 잘 유지되는지 확인하는 용도다.

### 2-4-6 Lambda의 동시성 지표

- 특정 시각에 동시에 처리 중인 인스턴스 수를 제공한다. 개별 함수뿐 아니라 계정-지역 내 모든 함수 대상으로도 제공되어, 각 함수나 전체가 동시성 제한에 얼마나 근접했는지 볼 수 있다. 수치 지표이므로 Max 통계로 본다.
- **ConcurrentExecutions**: 이벤트를 처리 중인 인스턴스 수. 지역의 동시 실행 할당량이나 함수의 예약된 동시성 한도에 도달하면 추가 호출이 제한된다.
- **ProvisionedConcurrentExecutions**: 프로비저닝된 동시성에서 이벤트를 처리 중인 인스턴스 수. 프로비저닝된 동시성이 있는 별칭이나 버전을 호출할 때마다 현재 개수를 보낸다.
- **ProvisionedConcurrencyUtilization**: 버전이나 별칭에서 ProvisionedConcurrentExecutions를 할당된 총 프로비저닝 동시성으로 나눈 값. 예를 들어 .5면 50% 사용 중이다.
- **UnreservedConcurrentExecutions**: 예약된 동시성이 없는 함수들이 처리 중인 이벤트 수.
- 동시성은 계정-지역 한도 안에서 관리되며, 예약된 동시성(reserved-concurrency)으로 함수 단위 관리가, 그 안에서 미리 인스턴스를 준비하는 프로비저닝된 동시성(provisioned-concurrency)으로 추가 관리가 가능하다.
- 예약 없이 하나의 풀에서 모든 함수를 관리하면 ConcurrentExecutions만 봐도 충분하다. 함수별로 예약한다면 각 함수의 ConcurrentExecutions로 예약 범위 내 사용량을, UnreservedConcurrentExecutions로 예약 없는 함수들이 제한에 부딪히지 않는지를 확인한다. 프로비저닝이 충분히 활용되는지와 부족하지 않은지는 ProvisionedConcurrentExecutions와 ProvisionedConcurrencyUtilization으로 본다.

### 2-4-7 Lambda의 동시성

- 요청이 이전 처리 완료 후 순차적으로 오면 인스턴스가 재사용되지만, 처리 중에 새 요청이 오면 새 인스턴스가 할당된다. 계정-지역의 최대 동시 실행 한도 때문에 무한히 늘지는 않는다.
- 계정-지역 한도이므로 여러 Lambda를 가진 스택에서는 각 함수가 한도를 어떤 전략으로 나눠 가질지 정해야 한다. 이를 위해 두 기능이 있다.
    1. 예약된 동시성: 지정한 Lambda의 최대 실행 가능 한도를 보장한다.
    2. 프로비저닝된 동시성: 예약된 동시성 내에서 warm-up된 인스턴스 수를 보장해 첫 기동 지연을 없앤다.
- 공식 문서 그림 설명(범례: 함수 동시성, 예약된 동시성, 프로비저닝된 동시성, 예약되지 않은 동시성, 한도로 제한된 부분)
    - 예약된 동시성 예제: 예약이 설정된 my-function-PROD와 my-function-DEV는 Other functions와 다른 구간에서 실행된다. 다른 함수가 아무리 많이 실행돼도 자기 한도는 영향을 받지 않는 독립 구간을 갖는다.
    - 프로비저닝된 동시성 예제: 그 구간 안에서 실행되는 함수는 이미 기동된 인스턴스를 재사용하는 것처럼 동작해 첫 기동 지연이 없다. my-function-PROD는 대부분 프로비저닝 구간에서 실행되어 빠르게 처리하고, 가끔 요청이 몰리면 일부가 예약 구간 내에서 새 인스턴스를 기동한다.
- 이 값들은 가용성과 효율성에 맞춰 지속 모니터링으로 관리해야 한다. 프로비저닝은 추가 요금이 있으니 효율과 요금 사이에서 적당한 수준을 찾는다.
- Lambda 중심 스택을 상용에 쓸 계획이면 동시성을 구체적으로 알아둘 것을 권하며, 공식 문서 두 개(Lambda 함수 동시성 관리, Lambda 함수 규모 조정)를 추천한다.

### 2-4-8 Lambda의 지표 확인

- Lambda 관리 콘솔에서 `hello-api-ts-dev-hello` 선택 후 모니터링 탭으로 가면 기본 지표 화면이 나온다.
- 구성과 용도는 다음과 같다.
    - Invocations: 얼마나, 잘 호출되는지
    - Duration: 각 실행에 걸리는 시간
    - Error count and success rate: 성공인지 실패인지
    - Throttles: 동시성 제한으로 실행이 실패했는지
    - Concurrent executions: 동시 실행 수로 한도 상향 필요 여부 판단
    - Async delivery failures, IteratorAge: 이벤트 소스 매핑 시 확인하는 지표로, 이번 구현과 무관해 아무것도 표시되지 않는다.

### 2-4-9 API Gateway의 지표 확인

- 어떤 항목이 있는지는 공식 문서로 확인하는 게 가장 좋고, 지금 다 쓰지 않더라도 정리해두면 나중에 도움이 된다. API Gateway는 1분마다 CloudWatch로 지표를 보낸다.
- **4XXError**: 기간 내 클라이언트 측 오류 수. Sum은 총 개수, Average는 4XX 오류율(총 4XX ÷ 총 요청).
- **5XXError**: 기간 내 서버 측 오류 수. Sum과 Average의 의미는 4XX와 같다.
- **CacheHitCount**: API 캐시에서 처리한 요청 수. Sum은 총 적중 수, Average는 캐시 적중률.
- **CacheMissCount**: 캐싱 활성 시 백엔드에서 처리한 요청 수. Sum은 총 누락 수, Average는 캐시 누락률.
- **Count**: 기간 내 총 API 요청 수. SampleCount 통계가 이 지표를 나타낸다.
- **IntegrationLatency**: API Gateway가 요청을 백엔드로 릴레이한 때부터 백엔드 응답을 받을 때까지의 시간(ms).
- **Latency**: 클라이언트 요청 수신부터 클라이언트에 응답을 반환할 때까지의 시간. 통합 지연 시간과 기타 API Gateway 오버헤드가 포함된다.
- 활용: 오류 확인은 4XXError와 5XXError, 전체 응답 수 비교는 Count. Lambda Duration으로 수행 시간을 가늠할 수 있지만 Authorizer나 기타 초기화까지 포함한 전체 시간은 IntegrationLatency나 Latency로 본다. 캐시를 쓰면 CacheHitCount와 CacheMissCount로 캐시 효율을 모니터링한다.
- Serverless Framework 기본 설정에서는 API Gateway 지표가 비활성화되어 있으므로 `serverless.ts`의 `provider.httpApi.metrics`를 true로 설정하고 `sls deploy`로 반영한다.
- Lambda와 달리 API Gateway 관리 콘솔에는 기본 지표 화면이 없으므로 CloudWatch 대시보드를 구성해 봐야 한다.

### 2-4-10 CloudWatch 대시보드 구성

- 여러 함수를 가진 스택이 여럿이면 Lambda 콘솔에서 하나씩 보는 게 번거롭고, API Gateway처럼 기본 화면이 없는 경우도 있어 CloudWatch 대시보드에 필요한 지표를 등록해 관리한다.
- 지표 탐색기로도 볼 수 있지만 여러 지표를 모아 보는 대시보드가 지속 모니터링에 더 도움이 된다.

**만드는 과정**

1. CloudWatch 대시보드 메뉴에서 대시보드 생성, 이름은 HelloAPI(허용 문자는 0~9A~Za~z-_).
2. 위젯 종류는 탐색기, 행, 누적 면적, 번호, 막대, 파이, 텍스트, 로그 테이블, 경보 상태 등이 있고 여기서는 가장 간단한 행(line)을 쓴다.
3. 데이터 원본으로 지표 또는 로그를 고른다. 로그를 고르면 CloudWatch Logs Insights 쿼리 결과로 그래프를 만든다. 여기선 지표를 선택.
4. 계정에서 지표가 수집되는 모든 AWS 자원이 나온다(ApiGateway, Lambda, 로그, S3 등). 서비스를 많이 운영할수록 항목이 많아진다.
5. API Gateway에 들어가면 ApiId, Method, Resource, Stage, 전체 API 기준으로 고를 수 있다. Serverless Framework 배포라면 대부분 ApiId로 충분하다. 응답이 올바른지 보려고 4xx, 5xx, 개수(Count)를 추가한다.
6. 기본 통계는 5분 평균이라 전체 요청 대비 비율로 보인다. 합계로 바꾸려면 그래프로 표시된 지표 버튼에서 통계와 기간을 수정한다. 지표별 최소 수집 단위보다 짧게 잡으면 무시된다. API Gateway는 1분마다 보내므로 30초나 10초로 설정해도 1분 그래프와 같다.
7. 위젯 생성 버튼으로 추가.

**지연 시간 위젯**

- 시간축 지표라 오류 위젯과 따로 추가한다. ApiId에서 IntegrationLatency와 Latency를 선택하고, 복제 버튼으로 각 지표를 2개씩 만들어 최대와 평균을 본다.
- 둘은 축 규모 차이가 커서 최대는 왼쪽 Y축, 평균은 오른쪽 Y축을 쓰도록 설정한다.

**Lambda 위젯**

- 함수별로 오류(Errors)와 호출(Invocations)을 함께 보는 위젯, 기간(Duration)의 최대와 평균 위젯을 추가할 수 있다.
- 전체 함수 기준 ConcurrentExecutions나 UnreservedConcurrentExecutions의 최대 위젯으로 최대 실행 한도 문제를 모니터링한다.

**마무리와 활용**

- 위젯 크기를 조절하고 대시보드 저장. 테스트 호출 후 1분 이상 기다리면 지표가 채워진다. 제목, 범례 수정이나 더 나은 시각화 위젯 추가로 다듬을 수 있다.
- Lambda 개별 함수 위젯은 제목에 마우스를 올리면 나오는 버튼으로 그 시간대의 CloudWatch Logs로 바로 이동할 수 있다(이 시간 범위에서 로그 보기). Error 지표로 에러를 인지한 뒤 바로 해당 시점 로그를 필터링해 추적할 때 유용하다.

### 2-4-11 CloudWatch 경보 설정

- 대시보드는 상황실 모니터에 띄워두고 볼 때 좋지만 매순간 볼 수는 없으므로, 이상치 발생 시 메일로 알리는 경보를 추가한다.
- CloudWatch 경보는 지표가 조건을 만족하면 AWS SNS로 이벤트를 보낸다. 예: API Gateway 4XX/5XX가 특정 수치를 넘거나 Lambda ConcurrentExecutions가 일정 값에 도달할 때. CloudWatch의 경보 메뉴에서 설정한다.
- 대시보드와 달리 CloudWatch Logs Insights는 경보에 쓸 수 없다.

**지표 선택**

- Lambda의 ConcurrentExecutions를 선택한다. 1분에 한 번 전송되므로 한도 내에서 빠르게 반응하도록 1분 최댓값을 근거로 설정한다.
- 경보 구성 화면에서 그래프와 함께 지표 이름, 통계, 기간을 다시 고를 수 있다.

**조건 설정**

- 임계값 유형: 특정 값과 비교하는 정적, 표준편차 대역과 비교하는 이상 탐지. 여기선 ConcurrentExecutions가 100을 넘으면 경보가 울리도록 정적, 보다 큼, 100으로 설정한다.
- 경보를 알릴 데이터 포인트: 평가 기간 내 위반 데이터 포인트 수(첫 칸)와 평가 기간(둘째 칸). 예를 들어 1분 집계 4XX 경보라면 즉시 알리기보다 2/5로 두어 5번 집계 중 2번 위반 시 알리는 게 효율적일 수 있다. 여기선 바로 알리도록 1/1.
- 누락된 데이터 처리: 누락, 양호, 무시, 불량 중 선택. 보통 미집계는 의도치 않은 상황일 가능성이 커서 누락이나 불량으로 다룬다. 하지만 ConcurrentExecutions는 함수가 한 번도 안 돌면 수집되지 않으므로, 접속이 드문 서비스라면 양호로 간주해야 한다. 이 예제도 테스트 외엔 호출이 없으니 양호로 설정한다.

**알림 설정**

- SNS로 경보 이벤트를 다른 서비스와 연계한다. 경보 상태뿐 아니라 정상이나 데이터 부족 상태도 알릴 수 있다.
- 기존 SNS 없이 이메일 알림만 만들려면: SNS 새 주제 생성, 고유한 이름 입력(경보를 받을 그룹 이름이면 됨), 수신 이메일 입력. 예제는 Default_CloudWatch_Alarms_Topic.
- 알림 추가 버튼으로 더 추가할 수 있고, 한 번 만든 주제는 이후 경보에서 기존 SNS 주제로 재사용한다.
- 이름과 설명 입력(예: LambdaConcurrentLimits, Lambda 동시성 한도 경보). 이 내용이 이메일로 가므로 문제 상황을 식별하기 쉽게 자세히 적는다.
- 검토 화면에서 경보 생성. 조건을 만족하지 않으니 정상 상태로 유지되며, 작업 칸에는 알림 작업 1개가 표시된다.
- 이메일로 받으려면 AWS Notifications 제목의 메일에서 Confirm subscription 링크로 인증해야 한다.

**부하 테스트로 경보 확인**

- 부하 생성기 autocannon(`npm install -g autocannon`)으로 동시 128개 연결을 60초 동안 보낸다(`autocannon -c 128 -d 60 URL`).
- 결과 예시: 지연 시간 평균 약 13.83ms, 초당 요청 평균 약 8,935건, 60초간 약 536k 요청.
- 약 1분 기다리면 경보 상태로 바뀌고, 항목을 누르면 그래프와 상태 업데이트 이후 작업 내역을 보여준다.
- 경보 메일에는 경보 이름, 설명, 상태 변화(INSUFFICIENT_DATA에서 ALARM), 원인(최근 데이터 포인트 128.0이 임계값 초과), 시각, 계정, ARN이 담긴다.
- 정상 상태 알림도 설정했다면 잠시 후 해제되면서 정상 복귀 메일이 온다.

## 2-5 비용 계산

- 예제 스택은 API Gateway, Lambda, CloudWatch로 구성되며 모두 프리티어가 있어 소규모 요청에선 비용이 없다.
- 실사용량은 예측을 벗어나 정확한 계산은 어렵지만 요청량을 추정하면 규모는 추산할 수 있다. 가격은 지역과 세부 항목마다 다르고 프리티어 항목도 서비스마다 다르므로 공식 가격 페이지(API Gateway, Lambda, CloudWatch)를 자세히 읽어야 한다.

### 2-5-1 API Gateway 비용

- HTTP API 외에 기능이 더 많은 REST API, WebSocket API가 있다. 여기선 HTTP API만 보며, HTTP API는 호출 가격만 있다.
- 월별 요청 수 기준 요금(백만 건당)
    - 첫 1백만 건: 무료(프리티어)
    - 처음 3억 건: 1.23USD
    - 3억 건 이상: 1.11USD
- 예: 하루 1백만 건씩 30일이면 첫 1백만 건을 뺀 2,900만 건 × 1.23 = 35.67USD. 시간으로 환산하면 약 0.0495USD로, EC2 t4g.medium(2vCPU 4GB, 0.0416USD/시간)보다 약간 비싸다. 하지만 그 사양으로 하루 1백만 건을 받는 고가용 게이트웨이를 운영하기 쉽지 않다는 점에서 이 규모에선 나쁘지 않다.
- 프리티어 1백만 건은 1분에 약 23건이 한 달 내내 들어와도 무료인 수준이라, 개인 프로젝트나 사내 도구를 저렴하게 만들기에 괜찮다.

### 2-5-2 Lambda 비용

- 요청(횟수)과 시간(GB-초)으로 구성된다. 메모리 크기를 고를 수 있으므로 수행 시간에 메모리를 곱한 GB-초로 계산한다.
    - 요청: 1백만 건당 0.20USD
    - 시간: GB-초당 0.0000166667USD
- 메모리는 128MB~10GB 사이에서 1MB 단위로 지정하고, 요금은 최소 1ms 단위로 계산한다. 별도 지정이 없으면 1,024MB를 쓴다.
- 메모리별 1ms당 요금
    - 128MB: 0.0000000021USD
    - 512MB: 0.0000000083USD
    - 1,024MB: 0.0000000167USD
    - 1,536MB: 0.0000000250USD
    - 2,048MB: 0.0000000333USD
    - 3,072MB: 0.0000000500USD
    - 4,096MB: 0.0000000667USD
    - 8,192MB: 0.0000001333USD
    - 10,240MB: 0.0000001667USD
- 측정되는 실행 시간에는 첫 기동 지연도 포함된다. 처리 15ms에 초기화 120ms면 135ms로 계산하고, 인스턴스가 재사용되면 이후엔 15ms만 발생한다.
- 프리티어: 매달 1백만 건 호출과 400,000GB-초 무료. 1GB 함수가 400,000초 실행할 수 있는 양으로, 한 달 동안 1초에 약 150ms씩 무료 처리 가능하다. 요청당 15ms라면 1GB 함수를 1초에 10번씩 한 달 내내 무료로 쓸 수 있다.
- 다만 1초에 10번이면 한 달 약 2,600만 호출이라 호출 비용 프리티어는 넘는다. 1백만 건당 0.2USD이므로 5.2USD가 나오며, EC2로는 t3.nano와 t3.micro 사이, Lightsail 5USD/월 모델과 비슷한 수준이다.

### 2-5-3 Lambda와 EC2의 가격 비교

- Lambda의 CPU는 메모리에 비례한다. 1,769MB에서 vCPU 1개 온전히, 최대 10,240MB에서 vCPU 6개. 공식 문서에 명시되진 않았지만 1,024MB면 vCPU 1개를 약 2/3 시분할로 쓴다고 한다. 그래서 기본값 1,024MB Lambda는 계산 집약보다 IO 집약 작업에 유리하다고 한다.
- 성능 비교가 어려운 이유: `/proc/cpuinfo`로 보면 Intel Xeon 2.50GHz인데 정확히 일치하는 EC2 CPU가 없다. Lambda를 서비스하는 Firecracker microVM 문서엔 C3나 T2 CPU 템플릿을 지원한다고 되어 있으나 그 사양과도 맞지 않는다. Firecracker가 동일 사양으로 맞추려 설정한 값으로 보이며, 동일 성능 EC2를 찾을 수 없어 정확한 성능 비교는 불가능하다.
- 그래서 한 달 Lambda 비용을 먼저 계산하고 시간 단위로 환산해 비슷한 비용의 EC2를 찾은 뒤, 그 EC2로 같은 일을 했을 때 최적화 여지가 있는지 보는 게 낫다.
- 예시: 요청당 15ms, 1GB Lambda가 1시간에 1백만 건 호출(프리티어 없음 가정)
    - Lambda 호출: 0.2USD/1백만 × 1백만 = 0.2USD
    - Lambda 시간: 0.0000000167USD/ms × (15ms × 1백만) = 0.2505USD
    - API Gateway 요청: 1.23USD/1백만 × 1백만 = 1.23USD
- AWS가 소수점 셋째 자리로 반올림하므로 1.681USD/시간이며, 이는 1.728USD/시간인 EC2 온디맨드 c5.9xlarge와 비슷하다. c5.9xlarge는 36vCPU 72GB로 c5.xlarge(4vCPU 8GB) 9대 규모이고, 이 정도면 시간당 1백만 요청 이상도 문제없다.
- EC2를 직접 쓰면 서버 자원, 인증서, 배포 등 인프라 작업이 늘지만, 요청량이 많을 땐 인프라 비용 면에서 EC2가 유리할 수 있다.

### 2-5-4 Lambda의 메모리와 CPU의 관계

- GB-초로 과금하니 메모리를 줄이면 싸다고 생각하기 쉽지만, CPU가 메모리에 비례하므로 틀리는 경우가 있다.
- 1,769MB 미만에서는 시분할로 vCPU를 제어한다(1,024MB면 약 0.67 vCPU).
- 시간 가격이 메모리에 거의 비례하므로 CPU 양까지 고려하면 가격 차이가 거의 없다. 1,024MB로 1초 걸리는 작업이 512MB에서 2초 걸리면 각각 0.0000167USD와 0.0000166USD로 사실상 같다.
- 하지만 DB 접근이나 네트워크 조회 같은 IO가 있으면 CPU를 안 쓰고 기다리는 구간이 생긴다. 상황별 비교는 다음과 같다.
    - 계산 집약, 1,024MB(0.67 vCPU): CPU 1초, IO 0초, 총 1초, 0.0000166USD
    - 계산 집약, 512MB(0.33 vCPU): CPU 2초, IO 0초, 총 2초, 0.0000167USD
    - IO 혼합, 1,024MB: CPU 0.5초, IO 0.5초, 총 1초, 0.0000166USD
    - IO 혼합, 512MB: CPU 1초, IO 0.5초, 총 1.5초, 0.00001245USD
- 즉 IO 대기가 섞이면 512MB가 비용 면에서 더 효율적이다.
- 반대로 계산 집약적이고 멀티 코어를 잘 쓰는 작업이면, 한 인스턴스 안에서 최대한 병렬 처리해 외부 공유 메모리를 통한 동기화 비용을 줄일 수 있다. 1,769MB, 1vCPU Lambda 6개로 병렬 처리하면 Redis 같은 외부 공유 자원과 네트워크 동기화 비용이 들지만, 10,240MB, 6vCPU 하나로 처리하면 더 빠르고 비용도 절약된다.
- 현실은 더 복잡하다. CPU와 IO 시간을 정확히 재기 어렵고, 6개 vCPU로 끝나는 병렬 작업도 사실상 없다. 그래서 초반엔 기본값 1,024MB 정도로 두고 CloudWatch 지표를 보며 메모리를 조정하는 게 바람직하다.
- AWS Compute Optimizer로 적합한 메모리를 추천받을 수 있다. 1,792MB보다 작은 메모리의 Lambda에 대해 지난 14일 수행 결과를 보고 추천하며, 그 기간 호출이 50회 미만이면 근거 부족으로 추천하지 못한다.

### 2-5-5 CloudWatch 비용 계산

- 기능이 많은 만큼 비용 항목도 다양하며, 여기선 예제에서 쓴 지표, 대시보드, 로그만 본다.
- 해상도: 1분 단위 표준 분해능과 1초 단위 고분해능. 고분해능은 1초 단위로 저장해 1, 5, 10, 30, 60초 배수 기간으로 조회하고 10초, 30초 단위 경보도 가능하지만 더 비싸다.

**프리티어**

- 지표: 기본 모니터링 지표(5분 간격), 세부 모니터링 지표 10개(1분 간격)
- 대시보드: 월별 최대 50개 지표를 제공하는 대시보드 3개
- 경보: 경보 지표 10개(고분해능 경보 제외)
- 로그: 5GB 데이터(수집, 아카이브 스토리지, Logs Insights 쿼리로 스캔한 데이터)

**로그**

- API Gateway나 Lambda 지표는 표준 분해능이라 한두 개 스택을 기본 모니터링하는 건 프리티어 범위다.
- 그런데 Lambda 로그는 보존 기간을 지정하지 않으면 평생 보존되어 의도치 않게 5GB를 넘을 수 있다.
- 로그 비용
    - 수집(데이터 수집): GB당 0.76USD
    - 스토어(아카이브): GB당 0.0314USD
    - 분석(Logs Insights 쿼리로 스캔한 데이터): GB당 0.0076USD
- 작은 서비스는 문제가 안 되지만 대량 로그의 상용 서비스에선 보관료가 크게 나올 수 있으니 불필요한 로그를 줄이는 게 좋다.
- `serverless.ts`의 `provider.logRetentionInDays`로 보존 기한을 지정한다. 14일로 두면 그 이전 로그는 자동 삭제된다.
- 오래된 로그는 CloudWatch Logs에 두기보다 S3로 내보내 Elasticsearch와 연동하거나 S3 Glacier로 보내 저렴하게 영구 보관하는 게 낫다.

**지표**

- 지표 전송은 AWS 내부 네트워크라 별도 비용이 없지만 수집 개수에 따라 비용이 든다. 다만 각 자원의 기본 모니터링 지표는 프리티어라 무료다.
- PutMetricData API로 직접 수집하거나 EC2 세부 모니터링처럼 비용 경고가 표시되는 지표를 수집할 때만 과금되며, 세부 모니터링 지표도 10개까지는 무료다.
- 지표당 월 비용
    - 처음 10,000개: 0.30USD
    - 다음 240,000개: 0.10USD
    - 다음 750,000개: 0.05USD
    - 1,000,000개 이상: 0.02USD

**대시보드**

- 기본 모니터링 지표로 대시보드당 50개 이하면 3개까지 무료. 3개를 넘거나 50개 이상 지표를 쓰면 대시보드당 3USD.
- 예: HTTP API 함수 9개 각각에 API Gateway 4XX, 5XX, 지연 시간과 Lambda 기간, 호출, 오류까지 6개 지표를 넣으면 54개(9 × 6)라 3USD가 든다.

**경보**

- 표준 분해능 지표 기반 경보 10개까지 무료. 초과하거나 고분해능 경보를 쓰면 과금된다.
    - 표준 분해능(60초) 경보 지표당: 0.10USD
    - 고분해능(10초) 경보 지표당: 0.30USD
    - 복합 경보당: 0.50USD
- 결론: 규모 있는 서비스 전까지는 CloudWatch에서 별도 비용을 보기 어렵다. 기본 지표로 대시보드와 경보를 구성해도 몇 USD 수준이다. 단 CloudWatch Logs는 로그 양에 따라 급증할 수 있으니 그 부분만 유의하면 된다.

### 2-5-6 경보 이메일 전송 비용

- 경보가 발생하면 SNS 표준 주제로 이벤트가 가고 이메일 구독 작업이 실행된다. 이는 CloudWatch가 아닌 SNS 요금이다.
- SNS는 순서가 보장되는 FIFO 주제도 있고, 이메일 외에 모바일 푸시, SMS, SQS 등 다양한 엔드포인트를 지원하지만 여기선 이메일만 본다.
    - API 호출: 매월 첫 1백만 개 무료, 이후 1백만 개당 0.50USD
    - 이메일 전달: 알림 1,000개 무료, 이후 100,000개당 2.00USD
    - 데이터 전송: 월 최대 1GB 무료, 이후 GB당 0.126USD(다음 9.999TB/월)
- 팬아웃 구조에 SNS를 쓰는 게 아니라 경보 이메일만 보내는 정도라면 사실상 프리티어라 비용이 없다.

### 2-5-7 Hello API 비용 계산

- 가정: HTTP API가 Lambda를 실행, Lambda는 기본 1,024MB에 실행당 평균 15ms, 로그는 CloudWatch Logs, 기본 모니터링 지표로 대시보드와 경보 구성. 매일 1백만 요청, 한 달 30일.
- 월 비용
    - API Gateway 요청: 프리티어 1백만 건, 2,900만 건 × 1.23USD/백만 건 = 35.67USD
    - Lambda 호출: 프리티어 1백만 건, 2,900만 건 × 0.20USD/백만 건 = 5.8USD
    - Lambda 실행: 프리티어 400,000GB-초. 1GB Lambda가 3천만 번 15ms씩 실행하면 450,000,000GB-ms이고 프리티어 400,000,000GB-ms를 뺀 50,000,000GB-ms × 0.0000000167USD = 0.835USD
    - 모니터링: 무료. 별도 로그를 안 남기면 로그 저장 비용이 없고, 기본 지표와 50개 미만 지표 대시보드라 무료다. 요청이 과도하게 몰려 ConcurrentExecutions 경보가 울려도 한 달 1,000번을 넘지 않으면 경보와 알림 비용도 없다.
- 합계 42.305USD, 시간당 약 0.059USD.
- EC2 온디맨드로 보면 t3.medium(2vCPU, 288분 버스트, 4GB)이 월 37.44USD, c5.large(2vCPU 4GB)가 월 69.12USD라 그 중간 정도다. CPU 버스트 없는 c5.large에 잘 만든 서버면 하루 1백만 트래픽은 무리 없겠지만, 요청량에 따라 확장, 축소되지 않아 추가 인프라 관리 비용이 들고 고가용성을 위해 인스턴스를 최소 한 대 더 띄워야 한다. 이 수준에선 서버리스가 관리 비용뿐 아니라 인프라 비용도 더 저렴하다.
- 하루 1천만 요청이면 달라진다(프리티어 제외, API Gateway와 Lambda만).
    - API Gateway 요청: 3억 건 × 1.23USD/백만 건 = 369USD
    - Lambda 호출: 3억 건 × 0.20USD/백만 건 = 60USD
    - Lambda 실행: 45억 GB-ms × 0.0000000167USD = 75.15USD
    - 합계 504.15USD
- c5.large 약 7대를 쓸 수 있는 금액이며, 고가용성을 확보하고도 하루 천만 이상을 받을 서버를 구성할 수 있다. 이 경우 관리 측면은 서버리스가 유리하지만 비용 측면은 EC2 직접 구성이 유리하다.

## 2-6 정리

- API Gateway가 HTTP 요청으로 이벤트를 만들고, Lambda가 이를 처리해 응답을 만들어 API Gateway로 반환하는 서버리스 스택을 구축했다. `serverless.ts`와 `handler.ts`를 합쳐 약 30줄로 가용성 높은 웹 서비스를 빠르게 만들 수 있었다.
- 자바스크립트 대신 타입스크립트로 구현했고, 웹팩으로 런타임에 필요한 파일만 배포되게 했으며, Serverless Framework 선언도 타입스크립트로 작성해 타입 지원을 받도록 했다.
- 배포한 서비스로 AWS 자원의 한도와 비용을 알아봤고, 한도를 고려해 어떤 지표를 모니터링할지, 요청량별 비용이 얼마나 드는지 살펴봤다.
- 대부분의 웹 서비스는 HTTP API로 요청을 받아 함수를 수행하고 결과를 반환하는 구조이므로, 이 예제를 바탕으로 다른 작업도 쉽게 확장할 수 있고 상용 서비스 준비 시 고려할 점도 같은 방식으로 고민할 수 있다.

---

**현재 기준으로 달라졌을 수 있는 부분**

- `nodejs14.x` 런타임은 이미 지원 종료되어 새로 배포할 수 없으니, 실습할 때는 최신 Node.js 런타임과 그에 맞는 `@types/node` 버전을 쓰면 된다.
- Serverless Framework는 v4부터 로그인이 필요하고 일정 규모 이상 조직에는 유료 라이선스 정책이 생겼으며, 번들링도 웹팩 대신 esbuild 기반이 흔해졌다. 개념 이해엔 책 흐름 그대로 문제없지만, 실습 환경에서 템플릿이나 명령 결과가 다르게 나올 수 있다.
- Lambda 동시 실행 기본 한도 같은 수치는 신규 계정에서 더 낮게 시작하는 경우가 있어, 실제 값은 Service Quotas 콘솔에서 확인하는 게 정확하다.
- 모든 가격은 책 집필 당시 기준이라 지금은 달라졌을 수 있으니 실제 계산은 공식 가격 페이지를 확인해야 한다.
- 책에서 기본 메모리를 1,024MB라고 한 건 Serverless Framework의 기본값이다. AWS에서 Lambda를 직접 만들면 기본값은 128MB다.
- 책은 실행 시간에 초기화 시간이 포함된다고 설명하지만, 과거에는 관리형 런타임의 온디맨드 함수에서 INIT 단계가 사실상 과금되지 않는 경우가 많았다. 내가 알기로는 2025년 8월부터 INIT 단계도 일괄 과금하도록 바뀌었으니, 콜드 스타트가 잦은 서비스라면 비용 계산 때 이 부분을 확인해보는 게 좋다.
- AWS 프리티어 정책도 2025년 중반 신규 계정 대상으로 크레딧 방식이 도입되는 등 바뀌었다. Lambda나 CloudWatch처럼 상시 무료로 제공되던 항목이 지금 어떻게 적용되는지는 계정 기준으로 확인하는 게 정확하다.
- Elasticsearch는 AWS에서 OpenSearch Service로 이름이 바뀌었다.
- Epsagon은 Cisco에 인수된 뒤 서비스가 종료된 걸로 알고 있어서, Lambda 모니터링 파트너 도구를 찾는다면 다른 선택지를 보는 게 낫다.