# 트러블슈팅 기록

## 1. 알려진 사례: 모델 alias 검증 오류로 inference가 시작하지 않음

### 관찰된 상황

한 후속 설정에서 Bob inference backend `1.26.7`이 반복 재시작했습니다. 로그에서 확인된 핵심 오류는 다음 의미였습니다.

```text
invalid bifrost model_info: model "premium-ide" has
(provider="qwen", family="qwen3-coder", version="30b-a3b")
but no matching alias table entry exists
```

같은 시작 로그에는 router 하나, 모델 0개가 기록된 뒤 embedded gateway 초기화가 panic으로 종료되었습니다. Kubernetes 이벤트에는 container BackOff와 반복 재시작이 나타났습니다.

이 오류는 128K 입력의 한계 테스트 결과가 아닙니다. 모델 alias metadata 검증이 실패해 inference HTTP server가 시작되기 전 발생한 초기화 오류입니다. 따라서 이 상태에서는 긴 컨텍스트 추론이 검증되지 않았습니다.

### 진단 및 조치 순서

1. Pod의 이전 로그와 재시작 상태를 확인합니다.

   ```bash
   oc logs -n ibm-bob "<BOB_INFERENCE_POD>" -c runtime --previous --tail=200
   oc get pod -n ibm-bob "<BOB_INFERENCE_POD>"
   ```

2. 배포에 실제 적용된 model gateway 설정에서 문제 alias의 `provider`, `family`, `version`을 확인합니다. 실제 ConfigMap 전체를 공개 로그나 저장소에 복사하지 않습니다.
3. 같은 Bob/backend 버전이 제공하는 alias table 또는 공식 모델 메타데이터 정의에서 정확히 지원되는 triple을 확인합니다.
4. alias table이 인식하는 metadata를 사용하거나, 해당 버전에서 지원되는 방식으로 alias를 추가합니다. 단지 model ID나 버전 문자열을 바꾸어 추정하지 않습니다.
5. 구성 업데이트 후 inference deployment를 재시작하고 rollout을 기다립니다.

   ```bash
   ./bobctl update-model-config --model-config "<PROTECTED_MODEL_CONFIG_PATH>"
   oc rollout restart deployment/bob-inference -n ibm-bob
   oc rollout status deployment/bob-inference -n ibm-bob
   ```

6. 로그에 초기화 panic이 없고 모델 정보 endpoint에 `premium-ide` 등 필요한 alias가 등록된 뒤에만 IDE 추론과 긴 컨텍스트 테스트로 진행합니다.

### 보안 및 데이터 정리

원본 대화/설정에는 내부 IP, 내부 DNS, 개인/조직 이메일, 초기 암호, API 설정과 인프라 식별자가 포함되어 있었습니다. 이 저장소에서는 해당 값을 재현하지 않고 자리표시자로 대체합니다. 로그를 외부에 공유할 때 Pod 이름, namespace, endpoint, 이메일, 토큰, 사용자 prompt를 검토하고 마스킹하세요.

## 2. 일반적인 진단 명령

```bash
oc get bob -n ibm-bob
oc get pods -n ibm-bob
oc describe pod -n ibm-bob "<POD_NAME>"
oc logs -n ibm-bob deployment/bob-inference --since=10m
oc get route -n ibm-bob
```

실패 시 순서: Pod 상태와 이전 로그 → 적용된 설정의 정확한 값 → model API 응답 → Bob model-info endpoint → IDE 요청 경로. 자격 증명이나 전체 ConfigMap을 이슈에 붙이지 않습니다.
