---
layout: post
title: "Monitoring Clipboard for Secrets with Trufflehog"
date: 2025-02-03
---

클립보드는 편리하고 종종 사용되지만 주의하지 않으면 민감한 정보의 노출로 이어질 수도 있는 기능이기도 합니다. 디버깅을 위해 Copy & Paste 한 에러메세지에 API key가 포함되어있거나, 번역이나 문서 요약을 위해 복사한 텍스트에 Credential이나 webhook URL이 포함되어 있을 수도 있구요.

특히 일부 서비스에서는 사용자가 명시적으로 버튼을 눌러 데이터를 전송하지 않더라도 클라이언트 측의 로직(e.g., keyup keydown event...)에 의해 입력된 데이터가 서버로 전송(e.g., 번역기)될 수 있기 때문에 중요한 정보가 외부로 유출되기 전에 이를 미리 감지하는 것이 중요합니다.

이번 포스트에서는 `trufflehog`를 활용해 클립보드에서 민감한 정보를 실시간으로 감지하는 방법에 대해 소개하려고 합니다.

## Trufflehog

`Trufflehog` (https://github.com/trufflesecurity/trufflehog)는 다양한 플랫폼에서 secret(e.g., API keys, database passwords, private encryption keys)을 탐지하고 분석할 수 있는 도구 중 하나입니다. 오랜기간 꾸준하게 유지보수되고 있는 프로젝트로 다른 도구들과 비교했을 때도 False Positive나 False Negative가 확실히 적습니다. (버그바운티에서도 자주 사용되는 모습이 보입니다) 

## Concept

### Pre-requisite

* trufflehog
* jq
* pbpaste (macOS)
* osascript (macOS)

### Script

```bash
LAST_CLIPBOARD=""

while true; do
    CLIPBOARD_DATA=$(pbpaste)

    if [[ "$CLIPBOARD_DATA" == "$LAST_CLIPBOARD" || -z "$CLIPBOARD_DATA" ]]; then
        sleep 0.2
        continue
    fi

    TMP_FILE=$(mktemp)
    echo "$CLIPBOARD_DATA" > "$TMP_FILE"

    RESULT=$(trufflehog filesystem --concurrency=1 -j --log-level=-1 "$TMP_FILE")

    DETECTOR_NAME=$(echo "$RESULT" | jq -r '.DetectorName')
    RAW=$(echo "$RESULT" | jq -r '.Raw')

    if [[ -n "$RESULT" ]]; then
        osascript -e "display notification \"$RAW\" with title \"🚨 $DETECTOR_NAME secret detected in clipboard!\""
    fi

    rm "$TMP_FILE"
    LAST_CLIPBOARD="$CLIPBOARD_DATA"
    sleep 0.2
done
```

### Description

스크립트의 길이만큼 내용도 간단합니다. 주요 로직은 다음과 같습니다.

1. 먼저 클립보드에 저장된 데이터를 주기적으로 가져와서(`pbpaste`) 임시 파일로 저장합니다.
2. trufflehog의 sub-command(filesystem)를 통해 임시 파일에 저장된 클립보드 데이터를 분석합니다.
3. secret이 탐지되었다면 사용자에게 notification을 전달합니다.(`osascript`)

> 본문에서는 macOS를 기반으로 스크립트를 작성했지만 다른 플랫폼에서도 비슷한 방식으로 구현할 수 있습니다. 😊

### Demo

![](/assets/images/250203_demo.gif)