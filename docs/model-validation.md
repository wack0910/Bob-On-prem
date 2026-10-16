# 모델 연결 및 긴 컨텍스트 검증

## 1. 먼저 정상 기동 확인

모델 테스트 전에 Bob inference가 반복 재시작 없이 Ready인지 확인합니다.

```bash
oc get pods -n ibm-bob -l app=bob-inference
oc logs -n ibm-bob deployment/bob-inference --since=10m
```

배포판이나 label이 다른 경우 해당 버전의 리소스 이름을 확인합니다. `CrashLoopBackOff`, panic, 또는 모델 0개 등록 상태에서는 IDE로 긴 입력 테스트를 하지 않습니다.

## 2. 모델 endpoint와 alias 확인

OCP에서 모델 서버가 보이는지 먼저 검사합니다.

```bash
curl --fail --silent --show-error \
  "<VLLM_BASE_URL>/v1/models" |
  jq '.data[] | {id, max_model_len}'
```

그 다음 Bob inference 내부 모델 정보 endpoint에 기대한 alias가 나타나는지 확인합니다. 참조 설치에서 사용한 내부 URL 형식은 `https://bob-inference.<NAMESPACE>.svc.cluster.local:7330/v1/model/info`였습니다. 조직이 승인한 DNS/TLS 구성과 신뢰 CA를 사용합니다. TLS 검증을 끄는 옵션(`-k`, `--insecure`)을 정상 점검 절차로 사용하지 마세요.

```bash
oc exec -n ibm-bob deployment/bob-gateway -- \
  curl --fail --silent --show-error \
    --cacert "<APPROVED_CA_FILE>" \
    "https://bob-inference.<NAMESPACE>.svc.cluster.local:7330/v1/model/info"
```

인증서가 OS trust store에서 이미 신뢰되면 `--cacert`는 생략할 수 있습니다. endpoint 또는 컨테이너 내 인증서 경로가 버전마다 다를 수 있으므로 배포 환경에서 확인합니다.

응답에서 설정한 alias (예: `premium-ide`, `security`, `background`)가 실제로 등록되었는지 확인합니다. YAML에 모델을 적었다는 사실만으로 등록 성공이 증명되지는 않습니다.

## 3. IDE 기능 테스트

1. IDE 로그인 후 비민감한 짧은 프롬프트로 기본 추론을 확인합니다.
2. 테스트 프롬프트가 사용할 모델 alias로 라우팅되는지 확인합니다.
3. 응답 내용이 비어 있지 않고, Gateway와 inference 로그에 같은 테스트 시간대의 요청이 있는지 운영자가 대조합니다.
4. 모델 서버에서 오류 없이 응답했는지 확인합니다.

## 4. 128K급 입력 테스트 계획

“128K”가 입력 한도인지, 입력과 출력 합계인지는 먼저 정의해야 합니다. 토큰은 문자 수와 같지 않습니다. vLLM의 `max_model_len`, Bob의 `max_input_tokens`, `max_output_tokens`, 모델 tokenizer, Bob이 붙이는 시스템 프롬프트/도구 정의를 함께 고려하세요.

참조 문서의 값 `max_input_tokens: 114688`과 `max_output_tokens: 8192`는 합계 122880 tokens입니다. 이는 128 Ki tokens (131072)와 같지 않으며, 설정만으로 backend가 실제 해당 길이를 지원한다고 단정할 수 없습니다. `max_model_len`은 모델 서버에서 직접 조회하고, 다음 조건을 확인합니다.

```text
실제 요청 입력 토큰 + 요청 출력 토큰 <= vLLM max_model_len
Bob 입력 제한과 출력 제한은 실제 Bob 설정에서 별도로 확인
```

권장 절차:

1. 승인된 테스트 모델의 tokenizer로 입력 토큰 수를 계산할 방법을 준비합니다. 근사 문자 수를 토큰 수로 간주하지 않습니다.
2. 짧은 요청이 성공하는지 확인합니다.
3. 비민감한 합성 텍스트를 사용해 입력 크기를 단계적으로 늘립니다. 예: 약 1K → 8K → 16K → 32K → 64K → 목표 길이.
4. 각 단계에서 목표 입력 토큰 수와 작은 고정 출력 상한을 기록하고, backend 총 context 한도를 넘지 않도록 합니다.
5. 성공 여부, HTTP/애플리케이션 오류, 응답 시간, 출력 완결성, Pod 재시작/OOM, vLLM 로그를 기록합니다.
6. 목표 크기에서 성공한 뒤 반복 실행해 안정성을 확인합니다. 운영 데이터, 실제 고객 코드, 비밀 prompt를 부하 테스트에 사용하지 않습니다.

테스트 결과에는 모델 ID와 Bob alias, 실제 적용된 제한값, 토큰 계산 방법, 입력/출력 토큰 수, 결과와 오류를 기록합니다. 공유 가능한 문서에는 내부 URL, 계정, request body 원문, API key, access token을 넣지 않습니다.

## 5. 실패 판별

| 증상 | 확인할 내용 |
|---|---|
| Bob inference가 시작 직후 panic | alias metadata 검증 오류, 잘못된 YAML, 모델 alias 매핑을 먼저 확인합니다. |
| Bob에 모델이 0개 등록됨 | startup log, 적용된 ConfigMap/secret 참조, 모델 gateway 설정 경로를 확인합니다. |
| vLLM에서 모델을 찾을 수 없음 | `/v1/models`의 실제 ID와 Gateway `openai_compatible.model` 값이 일치하는지 확인합니다. |
| 짧은 요청은 되지만 긴 요청은 실패 | Bob 제한, server `max_model_len`, 입력+출력 합계, tokenizer, KV cache 및 GPU 메모리 로그를 확인합니다. |
| 응답이 잘리거나 너무 짧음 | `max_output_tokens`, 요청 output limit, 종료 조건과 실제 응답 토큰 수를 확인합니다. |
| IDE에서 로그인은 되지만 추론 실패 | IDP가 아니라 Gateway→inference→vLLM 경로와 model alias를 진단합니다. |
