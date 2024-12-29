---
layout: post
title: "Encoding Differentials Lead to Filter Bypass"
date: 2024-12-29
---

최근 X에서 Gareth Heyes가 올린 포스트(https://x.com/garethheyes/status/1871540782328352965)를 보게 되었습니다. 공백 문자를 삽입해 blacklist 기반의 문자열 필터링을 우회하고 스크립트를 실행하는 방식은 잘 알려져있는 반면, 인코딩 간의 차이를 이용해 이를 우회하는 방법은 생소해서 background를 찾아보다가 Sonar의 연구원인 Stefan Schiller가 작성한 글을 읽고 정리한 내용을 공유해보려고 합니다.

자세한 내용이 궁금하신 분들은 https://www.sonarsource.com/blog/encoding-differentials-why-charset-matters/ 를 참고해주세요.

## Encoding
Encoding은 데이터를 컴퓨터가 처리할 수 있는 형태로 변환하는 과정을 의미합니다. 웹은 전 세계에 걸쳐 사용되는 서비스인만큼 각국에서 사용하고 있는 언어와 기호를 적절한 방식으로 변환하기 위해 인코딩 방법 또한 여러가지가 존재하고 있습니다.

만약 잘못된 방법을 통해 페이지를 디코딩하게 된다면 사용자는 아래와 같이 전혀 예상하지 못한 화면을 보게 될 수 있습니다.

![](/assets/images/241229_wrong_encoding_google.png)

## Encoding auto detection
브라우저는 이러한 상황을 예방하기 위해 `Content-Type` 헤더의 `charset` 속성이나 `meta` 태그의 `charset` 속성 등에 명시된 정보를 통해 html 문서가 어떤 방법으로 디코딩 되어야 하는지 결정합니다. 그런데 이러한 정보들이 누락되어있다면 브라우저는 어떻게 문서를 디코딩해야 할까요?

브라우저는 `auto-detection`이라는 방법을 통해 문서가 어떤 방법으로 인코딩되어있는지 추측하게 됩니다.

Chrome, Edge 등의 브라우저에서 사용하고 있는 렌더링 엔진인 blink에서는 CED라는 라이브러리(https://github.com/google/compact_enc_det)를 통해 인코딩 방식을 자동으로 탐지하고 있습니다. 

## ISO-2022-JP
ISO-2022-JP는 일본어 문자를 표현하기 위해 설계된 인코딩 방식 중 하나로 기본적으로 ASCII와 호환되도록 설계되었으며, 필요에 따라 escape sequence를 통해 다른 character set으로 전환해가면서 문자를 표현할 수 있다는 특징을 가지고 있습니다.

이 인코딩 방식에 포함된 character set 중 하나가 JIS X 0201 1976입니다. 이 character set은 대부분의 문자 매핑이 ASCII와 동일하지만, 몇 가지 예외가 존재합니다. 그 중 대표적인 예외가 바로 `0x5c`에 해당하는 `￥`인데요.

ASCII의 코드페이지에서 `0x5c`는 `\` 에 매핑이 되고 JIS X 0201 1976의 코드페이지에서는 `￥` 에 매핑이 됩니다. 만약 공격자가 문서에 적용될 인코딩 방법을 임의로 지정할 수 있다면, 공격자는 이 점을 이용해서 클라이언트 사이드에서 `\` 기호를 `￥` 기호로 치환할 수 있습니다.

## 공격 시나리오
Sonar에서 제시한 예시를 기반으로 간단하게 서비스를 만들어봤는데요. 코드는 다음과 같습니다. (조금 더 명확한 이해를 위해 auto detection에 사용되는 google의 compact_enc_det 라이브러리를 node.js에 바인딩한 (https://github.com/sonicdoe/ced) 라이브러리를 사용했습니다.)

```javascript
const ced = require('ced');
const express = require('express');
const app = express();
const port = 3000;

function escapeHTML(str) {
  return String(str)
    .replaceAll(/</g, '&lt;')
    .replaceAll(/>/g, '&gt;')
}

function escapeString(str) {
  return String(str)
    .replaceAll(/\\/g, '\\\\')
    .replaceAll(/"/g, '\\"');
}

app.get('/', (req, res) => {
  const search = escapeHTML(req.query.search || '');
  const lang = escapeString(req.query.lang || '');

  const htmlText = `
    <html>
    <body>
        <h1>You searched for: ${search}</h1>
        <script>
          var language = "${lang}";
        </script>
    </body>
    </html>
  `;

  const htmlBuffer = Buffer.from(htmlText);
  const encoding = ced(htmlBuffer);

  res.end(`
    <html>
      <body>
        <h1>Current Page Encoding: ${encoding}</h1>
        <pre>${htmlText}</pre>
      </body>
    </html>
  `);
});

app.listen(port, () => {
  console.log(`Server is running at http://localhost:${port}`);
});
```

search 파라미터에서는 태그 삽입에 필요한 꺽쇠 기호를 필터링하고 있고 lang 파라미터에서는 백슬래시 추가를 통해 string context 탈출에 필요한 특수문자들을 이스케이프 처리하고 있어 일반적인 방법으로는 페이지 내에 원하는 자바스크립트 코드를 삽입할 수 없습니다.

![](/assets/images/241229_filtered.png)

하지만 여기서 공격자가 character set을 JIS X 0201 1976으로 전환하는 escape sequence를 삽입하게 된다면 CED의 auto detection에 의해 해당 sequence 이후의 모든 바이트가 JIS X 0201 1976으로 디코딩되면서 escapeString 함수가 추가한 `\`가 브라우저에서는 `￥`으로 해석되게 됩니다.

![](/assets/images/241229_encoding_differential.png)

그리고 이를 통해 “가 정상적으로 삽입되면서 string context를 탈출할 수 있게 되고 그 결과 공격자가 원하는 자바스크립트 코드를 삽입할 수 있게 됩니다.

![](/assets/images/241229_bypass.png)

## Mitigation
Content-Type 헤더의 charset 속성이나 meta 태그의 charset 속성 등을 통해 character set을 지정하면 됩니다.

## Summary
exploit에 몇 가지 전제 조건이 있긴 하지만 많은 서비스에서 충분히 접할법한 환경이라고 생각되어 공유해봤습니다. 
