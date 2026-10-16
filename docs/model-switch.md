# 외부 모델 변경: Laguna에서 Qwen3-Coder로

## 1. 적용 대상과 확인 수준

| 항목 | 기준 |
|---|---|
| 변경할 실제 모델 | `Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8` |
| 모델 서버 | 사용자 확인: vLLM `0.31.0`, NVIDIA L40S 1장 |
| 컨텍스트 목표 | 사용자 보고: 128K로 상향. 이 문서의 목표값은 총 `131072` tokens이며 실측 용량은 별도 검증 |
| Bob | 설치 문서: 2.0.0. 장애 로그의 inference 컴포넌트 버전: 1.26.7 |
| 관찰된 문제 | `qwen / qwen3-coder / 30b-a3b` 조합이 해당 Bob alias table에 없어 시작 실패 |
| 임시 호환 예제 | 기존 성공 구성의 `poolside / laguna / s2.1` metadata를 유지하고 실제 backend만 Qwen으로 변경 |

최종 vLLM 실행 명령 전문과 최종 적용된 호환 metadata 조합은 현재 확보한 자료에 없습니다. 따라서 아래 YAML은 설명용 후보이며, 현재 실서버의 설정값 또는 Qwen 우회 성공이 확인된 구성이라고 표시하지 않습니다. 사실과 예제의 범위는 [자료와 확인 상태](sources-and-status.md)를 참고하세요.

## 2. 변경해야 하는 세 계층

```text
Bob 논리 이름: premium-ide / security / background
    -> Bob이 인정하는 model_info metadata
    -> openai_compatible.model = vLLM의 served-model-name
    -> vLLM이 실제 로드한 Qwen checkpoint
```

| 계층 | 바꾸는 값 | 이유 |
|---|---|---|
| vLLM 실제 모델 | `model`, 필요 시 tokenizer 및 모델 revision | Qwen 가중치와 Qwen chat template를 로드하기 위해 변경 |
| vLLM API 모델 이름 | `served-model-name` | `/v1/models`에 노출하고 API 요청이 선택할 이름 |
| Bob backend | `openai_compatible.model`, `base_url`, 인증 설정 | 실제 추론 요청을 새 서버/모델에 보내기 위해 변경 |
| Bob metadata | `model_info`와 토큰 한도 | Bob 시작 시 alias 검증과 클라이언트에 노출되는 모델 정보에 사용 |
| Bob 논리 alias | `premium-ide`, `security`, `background` | 기존 IDE/라우터가 사용하는 이름이므로 이번 변경 예제에서는 유지 |

`model_info`의 모델 이름을 바꾸는 것만으로 vLLM 가중치가 바뀌지 않습니다. 반대로 vLLM만 교체하면 Bob이 예전 API 모델 ID를 요청해 `model not found`가 발생할 수 있습니다.

## 3. 변경 전 준비

1. 현재 정상 설정과 버전을 보호된 경로에 백업합니다. 상세 명령은 [Deployment 반영·복구](deployment-rollout.md)를 따릅니다.
2. 새 Qwen checkpoint, tokenizer, chat template가 같은 revision인지 확인합니다.
3. [vLLM 설정 가이드](vllm-configuration.md)에 따라 모델 서버를 먼저 준비합니다.
4. 새 vLLM의 `/v1/models`, 짧은 chat completion, tool call을 직접 확인합니다.
5. 그 후 Bob 설정을 변경합니다. 단일 GPU에서 같은 endpoint를 교체하면 이전 모델이 중단되는 시간이 생깁니다.

## 4. Qwen alias 오류와 임시 호환 처리

### 4.1 오류의 의미

장애 시 `openai_compatible.model`은 Qwen으로 지정되어 있었지만 다음 metadata가 거부됐습니다.

```yaml
model_info:
  provider: qwen
  family: qwen3-coder
  version: "30b-a3b"
```

오류는 `model_info triple ... does not match any alias table entry`였습니다. 이 기록은 **해당 backend 버전과 alias table에서 이 조합을 인정하지 않았음**을 뜻합니다. 모든 Bob 버전이 Qwen을 지원하지 않는다고 일반화하지 않습니다.

### 4.2 기존 alias metadata를 유지하는 호환 경로

기존 설치에서 인정된 metadata 조합을 `model_info`에 유지하고 실제 요청 대상만 Qwen으로 바꾸는 방식입니다. 이 저장소의 후보는 최초 성공 설치에서 사용했던 `poolside / laguna / s2.1`입니다.

```yaml
openai_compatible:
  model: "Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8"
  base_url: "<VLLM_BASE_URL_WITHOUT_V1>"
  api_key: "<MODEL_API_KEY_OR_APPROVED_SENTINEL>"
model_info:
  provider: poolside
  family: laguna
  version: "s2.1"
```

이 예제는 alias metadata 호환 방식이며 인증을 우회하는 설정이 아닙니다. 실제 모델은 Qwen이지만 Bob에서 표시·계량되는 정체성은 Laguna가 될 수 있습니다. 가격/사용량 집계, 토큰 계산, 기능 표시, context budget 및 지원 범위가 실제 Qwen과 다를 수 있습니다. 세 metadata 필드가 통과해도 tool calling과 긴 컨텍스트 동작까지 보장되지 않습니다.

이 경로는 현재 Bob 버전에서 tuple이 인정되는지, 새로운 토큰 한도가 유지되는지, 실제 vLLM으로 Qwen ID가 전달되는지를 확인한 후에만 사용합니다. 제품 업데이트로 Qwen 정식 alias가 제공되면 실제 Qwen metadata로 전환하고 다시 검증합니다. 지원되는 custom alias table 확장 방법이 있다면 IBM이 안내한 해당 버전의 방법을 우선합니다. 확인되지 않은 alias 파일 경로, 환경변수 또는 검증 비활성화 옵션을 만들지 않습니다.

## 5. model-gateway.yaml 전체 예제

버전 관리용 템플릿: [model-gateway.qwen.compat.example.yaml](../examples/model-gateway.qwen.compat.example.yaml)

```yaml
x-providers:
  inference_service: &qwen_backend
    openai_compatible:
      model: "Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8"
      base_url: "<VLLM_BASE_URL_WITHOUT_V1>"
      api_key: "<MODEL_API_KEY_OR_APPROVED_SENTINEL>"
    model_info:
      exposed: true
      provider: poolside
      family: laguna
      version: "s2.1"
      max_input_tokens: 114688
      max_output_tokens: 8192
      supports_vision: false
      mode: chat
models:
  - <<: *qwen_backend
    model_name: premium-ide
  - <<: *qwen_backend
    model_name: security
  - <<: *qwen_backend
    model_name: background
routers:
  - name: router
    strategy: static
    static:
      default_model: premium-ide
```

### 각 필드를 이렇게 설정하는 이유

| 필드 | 값/선택 기준 | 이유와 변경 시 영향 |
|---|---|---|
| `openai_compatible.model` | vLLM `/v1/models`의 `id`와 동일 | Bob이 실제로 보내는 API 모델 선택값. 로컬 디렉터리 이름과 다를 수 있음 |
| `base_url` | 내부 모델 API의 승인된 origin | 최초 설치의 Bob adapter에서는 `/v1` 없는 origin을 사용. `/v1/v1/...` 중복이 생기지 않게 실제 요청 경로 확인 |
| `api_key` | 인증 서버의 실제 요구 사항에 맞춤 | 자리표시자를 그대로 보내면 인증 실패. 실제 값/secret 참조는 해당 Bob 번들의 지원 문법 사용 |
| `exposed` | `true` | Bob에서 사용할 모델 정보로 노출하기 위한 예제값 |
| `provider/family/version` | 인정되는 tuple 또는 검증한 호환 tuple | `openai_compatible`의 API adapter 선택과 구별. Qwen 실제 ID를 이 세 필드로 대체하지 않음 |
| `max_input_tokens` | `114688` | 자료에 나온 Qwen 변경 시점 값. 입력과 시스템/도구/파일 등 실제 요청 전체의 예산을 확인해야 함 |
| `max_output_tokens` | `8192` | 출력 상한의 예제. 요청별 출력 설정과 backend 한도가 별도로 적용될 수 있음 |
| `supports_vision` | `false` | 이 텍스트 코드 모델에 검증되지 않은 이미지 기능을 광고하지 않기 위해 설정 |
| `mode` | `chat` | chat completion 경로 사용 |
| `model_name` | 기존 세 논리 alias 유지 | IDE·보안·background 작업의 기존 참조를 유지. 세 alias가 같은 vLLM 자원과 큐를 공유 |
| `routers.static.default_model` | `premium-ide` | 이 예제의 기본 논리 모델 선택. 실제 존재하는 alias와 일치해야 함 |

`114688 + 8192 = 122880`이며, vLLM 총 한도 `131072` 대비 `8192`가 남습니다. 이 여유를 사용자 입력에 자동으로 더할 수 있는 것은 아닙니다. 실제 입력에는 system prompt, 대화 이력, tool schema, 파일 내용, chat template가 포함되고 Bob과 vLLM의 tokenizer/계산 방식이 다를 수 있습니다. [토큰 예산과 검증](model-validation.md)을 함께 확인하세요.

`&qwen_backend`와 `<<: *qwen_backend`는 YAML anchor와 merge입니다. 세 모델에 같은 설정을 복사하는 작성 편의 기능이며 GPU 모델 세 개를 로드한다는 뜻이 아닙니다. ConfigMap에 펼쳐진 YAML을 수동으로 고칠 경우 세 alias와 `x-providers` 값이 서로 달라질 수 있으므로 원본 파일을 수정하고 `bobctl`로 다시 반영합니다.

## 6. 보호된 실사용 파일 만들기

저장소의 템플릿을 Bastion의 보호된 작업 경로로 복사합니다. `<...>`는 자동 치환되지 않습니다. 실제 파일에서 승인된 URL과 인증 설정으로 교체해야 합니다.

```bash
umask 077
MODEL_CONFIG_DIR="<PROTECTED_CONFIG_DIRECTORY>"
REPO_DIR="<REPOSITORY_DIRECTORY>"
mkdir -p "$MODEL_CONFIG_DIR"
cp "$REPO_DIR/examples/model-gateway.qwen.compat.example.yaml" \
  "$MODEL_CONFIG_DIR/model-gateway.yaml"
chmod 600 "$MODEL_CONFIG_DIR/model-gateway.yaml"
# 이 로컬 파일을 편집해 자리표시자와 metadata를 검증한 값으로 바꿉니다.
```

인증이 없는 격리 테스트 endpoint는 조직에서 허용한 경우에만 해당 adapter가 요구하는 sentinel을 사용합니다. 키를 비워 두거나 sentinel을 입력했다고 endpoint에 인증 기능이 생기지 않습니다. 보호된 endpoint에서 실제 secret 값을 템플릿이나 Git에 넣지 않습니다.

## 7. Bob에 적용하고 rollout

```bash
cd "<BOB_RELEASE_DIRECTORY>"
./bobctl update-model-config \
  --model-config "<PROTECTED_CONFIG_DIRECTORY>/model-gateway.yaml"

oc rollout restart deployment/bob-inference -n ibm-bob
oc rollout status deployment/bob-inference -n ibm-bob --timeout=10m
oc get pods -n ibm-bob
oc logs -n ibm-bob deployment/bob-inference --since=10m
```

`update-model-config`가 자동 재시작했는지 실제 동작을 확인합니다. 자동 rollout이 시작됐다면 먼저 그 완료를 기다리고 불필요한 연속 재시작을 피합니다. ConfigMap만 바뀌고 Pod template이 그대로이면 새 프로세스가 읽도록 명시적 재시작이 필요할 수 있습니다.

Bob이 Operator로 관리되는 경우 생성된 ConfigMap/Deployment를 직접 수정하는 방식은 reconciliation으로 되돌아갈 수 있습니다. 지원되는 `bobctl`/CR 설정 경로를 사용합니다. 실패 시 반복 재시작보다 이전 로그와 실제 적용 설정을 먼저 확인합니다. 상세 상태 확인과 복구는 [롤아웃 가이드](deployment-rollout.md)를 따릅니다.

## 8. 모델 변경 완료 판정

1. vLLM이 Qwen checkpoint와 기대한 `served-model-name`으로 Ready.
2. 직접 chat completion과 tool call 성공.
3. Bob inference가 alias 검증 panic 없이 기동.
4. Bob 모델 정보에 세 alias와 기대한 토큰 제한이 표시됨.
5. IDE 요청과 vLLM 요청 로그를 대조하여 실제 Qwen으로 전달됨을 확인.
6. 짧은 채팅, 코드 작업, 도구 호출, 점진적 긴 입력 테스트 성공.

IDE Gateway 주소는 Bob 접속점이 바뀌지 않았다면 그대로 사용합니다. 호환 metadata를 사용 중이면 UI에 표시된 모델명만으로 실제 backend를 판단하지 마세요.
