# Qwen 모델 연결, 도구 호출과 긴 컨텍스트 검증

## 1. 검증 범위

대상은 `Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8`, 사용자 확인 환경은 L40S 1장과 vLLM 0.31.0입니다. 아래는 실제 서버에서 수행할 절차이며 이 문서 작성 중 클러스터/GPU 테스트를 실행한 결과가 아닙니다. 설정 변경은 [모델 변경](model-switch.md)과 [롤아웃](deployment-rollout.md) 순서를 따릅니다.

## 2. vLLM과 Bob 정상 기동

vLLM 시작 로그에서 실제 checkpoint, dtype/quantization, `max_model_len`, KV capacity를 확인합니다. Bob은 다음 명령으로 상태를 확인합니다.

```bash
oc get pods -n ibm-bob -l app=bob-inference
oc logs -n ibm-bob deployment/bob-inference --since=10m
```

label과 리소스 이름은 실제 배포에 맞춥니다. `Running`이어도 Ready 조건과 재시작을 확인합니다. `model_info triple` panic이 있으면 추론 전에 alias를 수정해야 합니다. router 초기화 로그의 `0 model(s)` 문구 하나만으로 최종 모델 등록 상태를 단정하지 말고 시작 완료 여부와 아래 model-info 응답을 함께 확인합니다.

## 3. 실제 vLLM API 모델 확인

보호된 curl 옵션 파일에 필요한 인증/TLS 옵션을 준비합니다. `<PROTECTED_CURL_OPTIONS>`는 curl이 읽는 로컬 파일이며 Git에 추가하지 않습니다. 인증 없는 격리 테스트 endpoint라면 승인된 환경에 한해 해당 옵션을 생략할 수 있습니다. TLS 검증을 비활성화하지 않습니다.

```bash
set -euo pipefail
VLLM_BASE_URL='<VLLM_BASE_URL_WITHOUT_V1>'
CURL_OPTIONS='<PROTECTED_CURL_OPTIONS>'
RESULT_DIR='<PROTECTED_VALIDATION_OUTPUT_DIRECTORY>'
umask 077
mkdir -p "$RESULT_DIR"

curl --fail --silent --show-error --config "$CURL_OPTIONS" \
  "$VLLM_BASE_URL/v1/models" |
  jq '.data[] | {id, max_model_len: (.max_model_len // "not_exposed")}'
```

`id`는 Bob의 `openai_compatible.model`과 같아야 합니다. `max_model_len`이 API에 없으면 누락을 실패로 오인하지 말고 시작 로그와 실제 경계 요청으로 확인합니다. 단지 checkpoint가 지원하는 길이를 서버의 현재 허용 길이로 대체하지 않습니다.

## 4. 직접 짧은 추론

이 절의 Bash 명령은 앞 절의 변수를 이어서 사용합니다. 응답은 보호된 결과 경로에 저장합니다.

```bash
curl --fail --silent --show-error --config "$CURL_OPTIONS" \
  -H 'Content-Type: application/json' \
  --data-binary @- "$VLLM_BASE_URL/v1/chat/completions" \
  > "$RESULT_DIR/chat.json" <<'JSON'
{
  "model": "Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8",
  "messages": [{"role": "user", "content": "Reply with the word ready."}],
  "max_tokens": 32,
  "temperature": 0,
  "stream": false
}
JSON

jq -e '.choices[0].message.content | type == "string" and length > 0' \
  "$RESULT_DIR/chat.json"
jq '{model, usage, finish_reason: .choices[0].finish_reason}' \
  "$RESULT_DIR/chat.json"
```

HTTP 성공, API error 부재, 비어 있지 않은 응답과 실제 모델 선택을 확인합니다. 짧은 출력이라도 `finish_reason: length`면 요청 출력 한도에 도달한 것인지 점검합니다. 이 결과는 tool calling 또는 긴 입력 성공을 증명하지 않습니다.

## 5. 직접 tool calling 확인

vLLM 0.31.0에서 Qwen3-Coder는 `enable-auto-tool-choice: true`, `tool-call-parser: qwen3_xml`을 기준으로 합니다. 외부 작업을 수행하지 않는 합성 도구로 JSON 형식부터 검사합니다.

```bash
curl --fail --silent --show-error --config "$CURL_OPTIONS" \
  -H 'Content-Type: application/json' \
  --data-binary @- "$VLLM_BASE_URL/v1/chat/completions" \
  > "$RESULT_DIR/tool.json" <<'JSON'
{
  "model": "Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8",
  "messages": [{"role": "user", "content": "Call lookup_test_value with key BOB_TEST. Do not guess the value."}],
  "tools": [{
    "type": "function",
    "function": {
      "name": "lookup_test_value",
      "description": "Look up a synthetic test fixture value.",
      "parameters": {
        "type": "object",
        "properties": {"key": {"type": "string", "enum": ["BOB_TEST"]}},
        "required": ["key"]
      }
    }
  }],
  "tool_choice": "auto",
  "max_tokens": 256,
  "temperature": 0,
  "stream": false
}
JSON

jq -e '.choices[0].message.tool_calls | length > 0' "$RESULT_DIR/tool.json"
jq -e '.choices[0].message.tool_calls[0].function |
  (.name == "lookup_test_value") and (.arguments | fromjson | .key == "BOB_TEST")' \
  "$RESULT_DIR/tool.json"
```

`auto`에서는 모델이 호출하지 않는 경우도 있습니다. 호출이 없으면 이 사례는 미통과로 기록하고 template/parser, prompt와 모델 출력을 확인합니다. 본문에 XML이나 함수 이름이 나온 것만으로 구조화 tool call 성공이라고 판단하지 않습니다.

그 다음에는 받은 `tool_call_id`를 유지한 assistant 메시지와, 합성 결과 `ready`를 반환하는 `role: tool` 메시지를 다음 요청에 포함해 최종 답변까지 확인합니다. 실제 기밀정보나 운영 도구를 사용할 필요는 없습니다.

## 6. Bob model-info와 IDE

참조 설치에서 사용한 내부 URL 형식은 `https://bob-inference.<NAMESPACE>.svc.cluster.local:7330/v1/model/info`입니다. 승인된 TLS 신뢰 체인을 사용합니다.

```bash
oc exec -n ibm-bob deployment/bob-gateway -- \
  curl --fail --silent --show-error \
    --cacert "<APPROVED_CA_FILE_IN_GATEWAY_CONTAINER>" \
    "https://bob-inference.<NAMESPACE>.svc.cluster.local:7330/v1/model/info"
```

CA 경로는 **Bastion이 아닌 실행 중인 Gateway 컨테이너 안의 경로**입니다. OS trust store에서 이미 신뢰하면 `--cacert`를 생략할 수 있습니다. endpoint/path/auth가 버전에 따라 다르면 해당 배포 설정을 확인하고 추가로 필요한 인증을 적용합니다.

1. `premium-ide`, `security`, `background` 등 필요한 alias와 토큰 제한을 확인합니다.
2. IDE에서 짧은 채팅과 작은 합성 코드 설명을 요청합니다.
3. 승인된 테스트 작업 폴더에서 파일 읽기 등의 도구가 실제 호출되는지 확인합니다.
4. streaming 답변과 tool call 인자가 끊김 없이 조립되는지 확인합니다. 직접 API에서도 `stream: true`와 curl `--no-buffer`로 델타·최종 종료 이벤트를 검사할 수 있습니다.
5. 같은 시간의 Bob/vLLM 로그와 응답 model ID를 대조해 실제 Qwen으로 전달됐는지 확인합니다. UI의 호환 모델명만으로 판정하지 않습니다.

## 7. 128K 토큰 예산

128 Ki tokens는 `131072`입니다. 이 값은 vLLM의 **입력+출력 총 한도 목표**이며 Qwen 자체의 최대 길이라는 뜻이 아닙니다. 모델 tokenizer, chat template와 실제 서버 용량을 확인합니다.

```text
vLLM 입력: system + tools + 이력 + 사용자/파일 + template 등
vLLM 실제 입력 토큰 + 요청 출력 상한 <= vLLM max_model_len

Bob 변경 시점 기록: 입력 114688 + 출력 8192 = 122880
vLLM 목표 131072 - 위 합계 = 8192 여유
```

여유분을 입력 상한에 자동으로 추가하지 않습니다. Bob이 광고한 metadata, 요청별 `max_tokens`, 엔진의 실제 제한, alias에 따른 예산 처리 중 더 엄격한 제한이 적용될 수 있습니다. 호환 metadata에서는 token 계산과 기능 표시가 실제 모델과 달라질 가능성을 검사합니다.

로컬 Qwen tokenizer로 합성 사용자 prompt의 입력 길이를 계산할 수 있습니다. `transformers`가 설치된 모델 서버/관리 환경에서 실행하는 예제입니다.

```python
from pathlib import Path
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained(
    "<LOCAL_QWEN_CHECKPOINT_DIRECTORY>", local_files_only=True
)
messages = [{"role": "user", "content": Path("<SYNTHETIC_PROMPT_FILE>").read_text()}]
tokens = tokenizer.apply_chat_template(
    messages, tokenize=True, add_generation_prompt=True
)
print(len(tokens))
```

실제 Bob 요청은 system prompt, tools, 이전 이력 등이 더해지므로 위 결과만으로 실제 입력 크기를 확정하지 않습니다. 직접 API의 `usage.prompt_tokens`와 요청 직렬화 방식을 대조합니다. 원문이나 토큰을 외부 로그로 유출할 필요는 없습니다.

## 8. 점진적 긴 입력 검증

1. 1K → 8K → 16K → 32K → 64K → 허용 입력 한도 순으로 늘립니다. 전체 prompt의 토큰 수를 기준으로 하며 문자 수로 대신하지 않습니다.
2. 초기 출력 상한은 256/512처럼 작게 고정하고 총 context 한도를 지킵니다. 요청에 따라 요구되는 최소 출력량은 별도로 고려합니다.
3. 합성 문서의 시작/중간/끝에 서로 다른 표식을 넣고 이를 정확히 찾아 답하게 합니다. 요청이 수락되는 것과 입력이 끝까지 전달되고 활용되는 것을 구별합니다.
4. `usage`, 응답 완결성, TTFT/총 지연, KV cache 사용량, preemption, GPU peak memory, OOM/Pod 재시작을 기록합니다.
5. cold 요청과 동일 prefix 재사용 요청을 구분합니다. prefix caching이 있는 반복 성공만으로 cold 요청 capacity를 판단하지 않습니다.
6. 단일 요청을 통과한 뒤에만 동시 요청 수를 늘리고 IDE background/security 부하를 포함합니다.

직접 vLLM의 128K 경계 요청이 성공해도 Bob 입력 제한이 114688이면 Bob 경로에서 같은 128K 입력을 허용한다고 볼 수 없습니다. 각각의 경로와 한도를 별도로 기록합니다. 서버가 거부해야 하는 경계 초과 요청도 통제된 테스트에서 명확한 오류로 거부되는지 확인하고, 자동 truncation으로 짧아진 입력을 긴 입력 성공으로 기록하지 않습니다.

## 9. 결과 기록 양식

| 항목 | 기록할 내용 |
|---|---|
| 환경 | 날짜, Bob/backend/vLLM 버전, 모델 revision, GPU 종류·개수 |
| 적용값 | 실제 context, input/output, KV dtype, memory fraction, sequences와 batch token budget |
| 경로 | 직접 vLLM / Bob model-info / IDE / tool / streaming / 긴 입력 |
| 요청 | 합성 입력 식별자, 실제 prompt tokens, 요청 output limit, 동시성 |
| 결과 | 통과·실패·미실행, HTTP/API error, finish reason, 사용량, 지연과 재시작 |
| 호환 처리 | 사용한 metadata tuple, 실제 Qwen model ID, 표시/계량 차이 확인 여부 |

이 저장소에는 민감값 없는 결과 요약만 기록합니다. 실제 URL, 이메일, token, 전체 request body와 운영 로그는 보호된 위치에 둡니다.
