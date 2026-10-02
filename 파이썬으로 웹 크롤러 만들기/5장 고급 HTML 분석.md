## 5.1 BeautifulSoup으로 속성 기반 검색

대부분의 사이트는 CSS 스타일 적용을 위해 class, id 속성을 많이 쓰고, 스크레이퍼는 이를 기준으로 태그를 구별할 수 있습니다. 예제에서는 대사가 red, 인물 이름이 green 클래스로 되어 있어 이름만 뽑아낼 수 있습니다.

```python
nameList = bs.find_all('span', {'class': 'green'})
for name in nameList:
    print(name.get_text())
```

- `bs.태그명`은 첫 번째 태그 하나만, `find_all`은 조건에 맞는 전체를 찾음
- `get_text()`는 태그를 모두 제거하고 텍스트만 반환. 태그 구조는 탐색에 필요하므로 출력이나 저장 직전에만 호출하는 것이 좋음

### find()와 find_all()의 매개변수

`find_all(tag, attrs, recursive, text, limit, **kwargs)` 형태이고, `find`는 limit=1인 find_all과 같습니다. 실제로는 대부분 tag와 attrs만 씁니다.

- **tag**: 태그 이름 하나 또는 여러 개. `find_all({'h1','h2','h3'})`는 헤더 태그 전체를 찾음
- **attrs**: 속성 딕셔너리. 값을 여러 개 주면 그중 하나만 일치해도 찾음. `{'class': {'green','red'}}`
- **recursive**: True(기본값)면 자손 전체, False면 최상위 자식만 검색. 보통 기본값 유지
- **text**: 속성이 아닌 텍스트 내용으로 검색. `find_all(text='the prince')` 결과는 7개
- **limit**: 페이지 순서대로 앞에서 몇 개만 가져옴
- **kwargs**: `find_all(id='title', class_='text')`처럼 키워드로 속성 지정. class는 파이썬 예약어라 `class_`로 써야 함

keyword 방식은 간결하고 attrs 방식은 값 목록이나 복잡한 필터에 유리합니다. id는 페이지에 보통 하나뿐이므로 `find`를 쓰는 것이 적합합니다.

### BeautifulSoup의 네 가지 객체

- **BeautifulSoup**: 문서 전체 (예제의 `bs`)
- **Tag**: find, find_all 결과나 `bs.div.h1` 같은 탐색으로 얻는 태그
- **NavigableString**: 태그 안에 들어 있는 텍스트
- **Comment**: HTML 주석 `<!-- -->`

### 트리 이동

이름과 속성이 아니라 문서 안의 위치를 기준으로 찾을 때 사용합니다. 예제 페이지(page3.html)는 `table#giftList` 안에 제목 행과 상품 행들이 있는 쇼핑몰 구조입니다.

**자식과 자손**

- 자식은 바로 한 단계 아래, 자손은 여러 단계 아래까지 포함
- BeautifulSoup 함수는 기본적으로 자손을 대상으로 동작
- `.children`으로 테이블을 탐색하면 행 목록만, `.descendants`로 탐색하면 td, span, img까지 20개 넘는 태그가 나옴

```python
for child in bs.find('table', {'id': 'giftList'}).children:
    print(child)
```

**형제**

- `next_siblings`는 자기 자신을 제외한 다음 형제들을 반환
- 제목 행에서 호출하면 제목 행을 뺀 상품 행만 깔끔하게 얻을 수 있음
- `previous_siblings`는 앞쪽 형제들, `next_sibling`과 `previous_sibling`은 태그 하나만 반환

```python
for sibling in bs.find('table', {'id': 'giftList'}).tr.next_siblings:
    print(sibling)
```

**부모**

- `.parent`, `.parents`로 위로 올라감
- 아래 코드는 이미지 선택 후 부모 td로 올라가고, 그 앞 형제 td(가격 칸)의 텍스트 $15.00을 가져옴

```python
bs.find('img', {'src': '../img/gifts/img1.jpg'}).parent.previous_sibling.get_text()
```

`bs.tr`만으로도 동작하지만, 페이지 구조는 언제든 바뀔 수 있으므로 속성을 이용해 명확하게 선택하는 것이 안정적입니다.

## 5.2 정규 표현식

문자열이 정해진 규칙을 따르는지 판별하는 도구로, 전화번호나 이메일 같은 패턴 처리에 유용합니다. 예를 들어 `aa*bbbbb(cc)*(d|e)`는 a 1개 이상, b 정확히 5개, c 짝수 개, 마지막에 d 또는 e라는 네 가지 규칙을 한 줄로 표현한 것입니다.

**주요 기호**

- : 앞 요소가 0번 이상 (`a*b*`)
- `+`: 앞 요소가 1번 이상 (`a+b+`)
- `[]`: 괄호 안 문자 중 하나 (`[A-Z]*`)
- `()`: 그룹, 가장 먼저 평가됨 (`(a*b)*`)
- `{m,n}`: 앞 요소가 m번 이상 n번 이하 (`a{2,3}`)
- `[^]`: 괄호 안 문자를 제외한 문자 (`[^A-Z]*`)
- `|`: 둘 중 하나 (`b(a|i|e)d`)
- `.`: 임의의 문자 하나 (`b.d`)
- `^`: 문자열의 시작 (`^a`)
- `\`: 특수 문자를 원래 의미로 사용 (`\.`)
- `$`: 문자열의 끝 (`[A-Z]*[a-z]*$`)
- `?!`: 해당 위치에 그 문자가 오지 않음. 전체에서 배제하려면 `^`, `$`와 함께 사용

**이메일 정규식 만들기**

규칙을 단계별로 나누고 합칩니다.

- 아이디 부분: `[A-Za-z0-9._+]+`
- 반드시 @ 하나: `@`
- 도메인 이름: `[A-Za-z]+`
- 마침표: `\.`
- 최상위 도메인: `(com|org|edu|net)`

```
[A-Za-z0-9._+]+@[A-Za-z]+\.(com|org|edu|net)
```

정규식을 처음 만들 때는 목표 문자열의 구성을 단계별로 적어보고, 특히 맨 앞과 맨 뒤 조건을 신경 써야 합니다. 언어마다 구현이 조금씩 다르므로(파이썬은 펄 문법 기반) 헷갈리면 문서를 확인하세요.

## 5.3 정규 표현식과 BeautifulSoup

문자열을 받는 대부분의 BeautifulSoup 함수는 정규식도 받을 수 있습니다. 페이지의 img를 전부 가져오면 로고, 공백용 이미지 등이 섞이기 때문에 상품 이미지 경로 패턴으로 걸러냅니다.

```python
import re
images = bs.find_all('img', {'src': re.compile(r'\.\./img/gifts/img.*\.jpg')})
for image in images:
    print(image['src'])
```

페이지 위치나 레이아웃이 아니라 태그 자체를 식별할 수 있는 특징(여기서는 파일 경로)을 찾는 것이 핵심입니다.

## 5.4 속성에 접근하기

a 태그의 href나 img 태그의 src처럼 내용보다 속성값이 필요한 경우가 많습니다.

- `myTag.attrs`는 파이썬 딕셔너리를 반환
- `myImgTag.attrs['src']`로 원하는 속성값을 바로 꺼냄

## 5.5 람다 표현식

람다는 이름 없이 한 줄로 정의하는 함수입니다(`lambda n: n**2`). `find_all`에 태그 객체를 받아 True/False를 반환하는 함수를 넘기면, True로 평가된 태그만 반환됩니다.

```python
bs.find_all(lambda tag: len(tag.attrs) == 2)   # 속성이 정확히 2개인 태그
bs.find_all(lambda tag: tag.get_text() == "Or maybe he's only resting?")
```

두 번째 예시는 `find_all('', text='...')`로도 같은 결과를 얻습니다. 불리언만 반환하면 어떤 조건이든 쓸 수 있고 정규식과도 결합할 수 있어서, 람다만으로 BeautifulSoup의 다른 검색 기능 상당 부분을 대체할 수 있습니다.

## 5.6 닭 잡는 데 소 잡는 칼은 필요 없다

```python
bs.find_all('table')[4].find_all('tr')[2].find('td').find_all('div')[1].find('a')
```

이런 인덱스 체인은 사이트 구조가 조금만 바뀌어도 깨집니다. 복잡한 분석에 뛰어들기 전에 다음을 먼저 확인하라는 조언입니다.

- 원하는 정보 근처의 랜드마크(특정 CSS 속성, 특정 문자열을 가진 요소)를 찾아 기준점으로 삼기
- 원하는 태그를 직접 찾기 어렵다면 형제 태그를 찾고 `.parent`로 올라간 뒤 자식을 탐색하기
- 인쇄용 페이지나 HTML 구조가 더 단순한 모바일 버전 찾기
- script 태그나 자바스크립트 파일 안에 정리된 형태로 들어 있는 데이터 찾기
- 정보가 URL이나 페이지 타이틀에 있지 않은지, 다른 사이트에서 더 쉽게 얻을 수 없는지 확인하기