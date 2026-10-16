# vLLM 0.31.0: Qwen3-Coder-30B FP8와 L40S 1장

## 1. 사용자 제공 실적용 구성과 확인 범위

사용자가 확인한 장비/버전은 **NVIDIA L40S 1장, vLLM 0.31.0**이며 대상 모델은 `Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8`입니다. 후속으로 `vllm serve --config config.yaml`과 전체 YAML을 제공받아 이 문서의 값을 실적용 구성 기준으로 갱신했습니다. 이전 예제의 0.90 memory / 1 sequence / 2048 batch tokens는 현재값이 아닙니다. 현재값은 **0.92 / 2 / 4096**, `enforce-eager: true`입니다.

사용자가 제공한 로그와 Bob 화면에서는 짧은 에이전트 작업의 추론·도구 동작이 관찰됩니다. 128K 경계 입력, 두 개의 긴 요청 동시 처리, 생성된 달력 프로그램의 최종 정상 종료는 아직 확인하지 않았습니다. [관찰 결과](observed-qwen-run.md)를 참고하세요.

공식 vLLM **v0.31.0 tag**의 [서버 구성](https://github.com/vllm-project/vllm/blob/v0.31.0/docs/configuration/serve_args.md), [모델 설정](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/config/model.py), [cache 설정](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/config/cache.py), [scheduler 설정](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/config/scheduler.py), [tool calling](https://github.com/vllm-project/vllm/blob/v0.31.0/docs/features/tool_calling.md)을 기준으로 옵션 의미를 확인했습니다. 컨테이너가 같은 버전의 사용자 빌드라면 실제 이미지의 `--help`와 시작 로그도 대조합니다.

## 2. 단일 L40S에서 먼저 알아야 할 제약

L40S의 명목 메모리는 48GB입니다. 실제 가용 메모리는 드라이버, 다른 프로세스, 런타임 예약량 등에 따라 달라지므로 `nvidia-smi`로 확인합니다.

`30B-A3B`는 MoE의 전체/활성 파라미터 규모를 구분한 이름입니다. 요청당 활성 파라미터가 약 3B라고 해서 GPU에 3B 분량의 가중치만 올리면 되는 것이 아닙니다. 전체 전문가 가중치, 양자화 scale, activation, CUDA graph, workspace 및 KV cache가 함께 메모리를 사용합니다.

FP8 checkpoint는 가중치 메모리를 줄입니다. **가중치 FP8과 KV cache FP8은 별도 설정**입니다. `kv-cache-dtype: auto`가 FP8 가중치 이름을 보고 자동으로 FP8 cache를 선택한다고 가정하지 마세요. 이 설정은 모델 dtype에 따라 BF16/FP16 cache를 사용할 수 있습니다.

일반적인 full-attention 모델의 KV cache 원시 크기는 다음 값에 대략 비례합니다.

```text
2(K,V) × layer 수 × KV head 수 × head dimension
× 살아 있는 전체 토큰 수 × cache 원소 byte 수
```

예를 들어 **실제 config.json이** 48 layers, 4 KV heads, head dimension 128인 경우 단일 131072-token sequence는 BF16/FP16에서 약 12GiB, FP8에서 약 6GiB의 원시 KV 공간이 필요합니다. 이는 조건부 계산이며 현재 checkpoint의 구조를 직접 확인해야 합니다. 메모리 allocator와 block/scale overhead, graph 등은 추가됩니다. 두 긴 요청의 KV 공간은 대체로 더해지므로 128K 한 요청 성공이 128K 동시 여러 요청 성공을 의미하지 않습니다.

```bash
nvidia-smi
python -c 'import vllm; print(vllm.__version__)'
vllm serve --help
```

## 3. YAML과 CLI 중 실제 적용 경로 확인

vLLM YAML에는 CLI의 **긴 옵션 이름에서 `--`를 뺀 이름**을 사용합니다. Bob의 `model_info.max_input_tokens`처럼 underscore를 쓰는 구성과 혼동하지 마세요.

```text
vLLM CLI:  --max-model-len 131072
vLLM YAML: max-model-len: 131072
Bob YAML:  max_input_tokens: 114688
```

실행은 다음 형식입니다.

```bash
vllm serve --config "<PROTECTED_VLLM_CONFIG_PATH>"
```

vLLM 0.31.0의 우선순위는 **CLI 인자 > YAML > 기본값**입니다. ConfigMap의 `max-model-len`을 수정해도 Deployment `args`에 예전 값이 남아 있으면 예전 CLI 값이 우선합니다. 실제 Pod의 `command`, `args`, mount 경로, entrypoint wrapper를 확인합니다. 인증키가 인자에 있을 수 있으므로 출력 전체를 공개하지 않습니다.

## 4. 사용자 제공 YAML: 공개용 주소 마스킹

[공개용 구성 파일](../examples/vllm-qwen-l40s.example.yaml)은 사용자가 제공한 수치를 유지하고 bind address만 자리표시자로 대체했습니다. 실제 원본은 모든 네트워크 인터페이스에 bind하도록 지정되어 있었습니다. 실사용 복사본의 주소와 접근 통제를 확인합니다. `model`은 제공된 Hub ID 그대로이며, 네트워크 다운로드 또는 이미 준비된 cache가 필요합니다. 오프라인 운용 시 같은 revision의 로컬 checkpoint 경로로 대체할 수 있습니다.

```yaml
model: Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8
served-model-name: Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8

host: "<BIND_ADDRESS>"
port: 8000

dtype: auto
kv-cache-dtype: fp8
tensor-parallel-size: 1
pipeline-parallel-size: 1
gpu-memory-utilization: 0.92

max-model-len: 131072
language-model-only: true

max-num-seqs: 2
max-num-batched-tokens: 4096
enable-chunked-prefill: true

enable-auto-tool-choice: true
tool-call-parser: qwen3_xml
enable-prefix-caching: true

generation-config: vllm

enforce-eager: true
```

인증 설정은 이 공개 예제에서 생략했습니다. 이 상태의 예제가 인증을 제공한다고 생각하지 마세요. 서버 노출 범위를 제한하고, 실제 API key/프록시 인증은 조직이 운영하는 기존 인증 경로와 지원되는 secret 주입 방식으로 유지합니다. GPU workload는 기존 이미지, 모델 PVC, 보안 컨텍스트, GPU 리소스 요청과 함께 실행해야 하며 YAML만으로 GPU가 할당되지는 않습니다.

## 5. 각 파라미터를 이렇게 정한 이유

### 모델 선택과 병렬 처리

| 옵션 | 제공된 값 | 의미와 선택 이유 |
|---|---|---|
| `model` | Qwen Hub ID | 실제 로드할 FP8 가중치. 현재는 Hub ID를 사용하므로 모델 cache 또는 다운로드 경로를 확인. 로컬 경로를 쓰면 같은 revision의 tokenizer와 template를 함께 준비 |
| `served-model-name` | Qwen API ID | 로컬 경로와 외부 요청 이름을 분리. Bob의 `openai_compatible.model` 및 `/v1/models` ID와 일치시킴 |
| `host`, `port` | 승인 bind 주소, `8000` | 컨테이너 listener. Kubernetes Service의 `targetPort`와 맞추고 외부 노출 범위는 별도로 통제 |
| `dtype` | `auto` | checkpoint/model config에 따라 dtype 선택. 가중치 quantization 설정과 구분되며 실제 선택은 startup log로 확인 |
| `tensor-parallel-size` | `1` | GPU가 한 장이므로 tensor parallel rank도 하나. 2로 바꾼다고 한 장의 메모리가 두 배가 되지 않음 |
| `pipeline-parallel-size` | `1` | 모델 layer를 여러 pipeline stage로 나누지 않음. 단일 GPU의 현재 구성과 일치 |
| `language-model-only` | `true` | 멀티모달 모델에서는 모든 modality 입력 한도를 0으로 설정하는 옵션. 이 Qwen 코드 모델은 원래 텍스트 모델이므로 큰 성능 향상을 주는 튜닝값으로 보지 않음 |

예제는 `quantization`을 강제로 지정하지 않습니다. vLLM은 checkpoint의 `quantization_config`로 양자화 방식을 판별할 수 있습니다. Qwen FP8이라고 해서 AWQ/GPTQ로 바꾸거나 BF16 checkpoint에 이름만 FP8로 붙이지 않습니다. 현재 GPU/driver/CUDA/커널이 그 FP8 checkpoint를 지원하는지 실제 로딩 결과로 확인합니다.

### 컨텍스트와 GPU 메모리

| 옵션 | 제공된 값 | 의미와 선택 이유 |
|---|---|---|
| `max-model-len` | `131072` | 입력+출력 총 context 목표. 모델의 지원 길이와 별도로 현재 서버에 허용할 길이를 제한. 가중치를 바꾸거나 길이 지원을 새로 만드는 옵션이 아님 |
| `gpu-memory-utilization` | `0.92` | 모델 executor의 메모리 예산을 가용 GPU 메모리의 92% 비율로 설정. 확인한 0.31.0 소스 기본값과 같으며, 실제 사용량이 항상 92%라는 뜻은 아님. GPU 계산 사용률이나 KV cache만의 비율도 아님 |
| `kv-cache-dtype` | `fp8` | 긴 입력의 cache 공간을 줄이는 설정. FP16/BF16 대비 원시 cache byte가 대략 절반이지만 품질·scale·커널 호환성을 따로 검증 |

현재 `0.92`에서도 더 긴 부하가 실패하면 무조건 `0.99`로 올리지 않습니다. vLLM의 모델 로딩/프로파일링 로그에서 남은 KV cache capacity와 가능한 최대 길이를 확인합니다. 우선 context를 낮춰 기능을 검증하고, GPU에서 다른 프로세스를 제거하거나 batch/concurrency를 줄인 뒤 다시 측정합니다. 비율을 올릴 때는 실제 여유 메모리와 peak OOM을 확인합니다. `kv-cache-memory-bytes`를 별도로 지정한 경우 0.31.0에서는 이 설정이 cache 예산을 직접 정하며 `gpu-memory-utilization`에 따른 추정을 대체하므로 두 값을 함께 바꾸지 말고 우선 적용값을 확인합니다.

FP8 KV cache에는 적절한 scale이 중요합니다. 0.31.0 공식 가이드는 calibration이 없는 FP8 cache의 기본 scale과 dataset calibration 경로를 구분합니다. 가중치 FP8 checkpoint에 알맞은 KV calibration 정보까지 들어 있다고 가정하지 마세요. 긴 입력 회수 정확도, 코드 생성, tool arguments가 BF16/FP16 기준과 비교해 유지되는지 확인합니다. 오래된 가이드의 `--calculate-kv-scales`를 그대로 붙이지 않습니다. 이 저장소가 확인한 0.31.0 cache/engine 옵션에는 해당 인자가 없으며, 사용할 build의 `--help`와 공식 calibration 절차를 따릅니다.

### 처리량과 긴 입력 분할

| 옵션 | 제공된 값 | 의미와 선택 이유 |
|---|---|---|
| `max-num-seqs` | `2` | scheduler/model runner가 동시에 처리할 sequence 수를 제한. 단일 GPU에서 여러 요청의 메모리 경쟁을 제한하면서 두 sequence까지 허용. HTTP 접속 수 제한 또는 128K 두 요청의 메모리 보장은 아님 |
| `max-num-batched-tokens` | `4096` | 한 scheduler iteration의 토큰 처리 예산. 너무 큰 prefill batch의 activation/workspace 부담을 제한하면서 처리량을 확보하는 절충값. 긴 prompt 전체 길이 제한이 아님 |
| `enable-chunked-prefill` | `true` | 긴 prompt의 prefill을 여러 iteration으로 나눔. 따라서 4096 batch budget으로도 128K prompt 처리가 가능할 수 있음 |
| `enable-prefix-caching` | `true` | 동일한 prefix의 KV 재사용. 반복되는 IDE system prompt/대화 prefix에서 prefill 비용을 줄일 수 있으나 무관한 입력 또는 cold request의 cache 필요량은 사라지지 않음 |

Chunked prefill을 끈 상태에서 batch budget을 context보다 작게 두면 엔진 제약에 걸릴 수 있습니다. 반대로 batch token 수를 131072로 키우면 한 번에 긴 입력을 처리할 activation/workspace peak가 커집니다. 현재 `4096`을 기준으로, 메모리 압력이 있으면 `2048`, 여유가 있고 처리량 개선을 확인하려면 `8192` 등을 비교할 수 있습니다. 무조건 큰 값이 IDE 응답을 빠르게 하지는 않습니다. 첫 토큰 지연(TTFT), token 간 지연, 처리량, peak memory를 함께 기록합니다.

`max-num-seqs: 2`를 사용하더라도 제공된 로그에는 동시 Running 2가 관찰되지 않았습니다. 한 요청이 대부분의 KV 공간을 사용하면 다른 요청이 대기하거나 preemption이 발생할 수 있습니다. 최대 길이 요청을 함께 넣어 KV capacity, preemption, 대기시간, timeout을 측정해야 두 요청 동시 처리를 검증할 수 있습니다. Bob의 세 alias가 같은 서버를 사용하면 background/security 요청도 이 예산을 공유합니다.

### Bob IDE의 도구 호출과 생성 기본값

| 옵션 | 제공된 값 | 의미와 선택 이유 |
|---|---|---|
| `enable-auto-tool-choice` | `true` | `tool_choice: auto` 요청에서 모델이 함수 호출을 선택할 수 있도록 서버 기능 활성화 |
| `tool-call-parser` | `qwen3_xml` | **vLLM 0.31.0 공식 문서의 Qwen3-Coder parser 이름**. Qwen3 일반 모델, Hermes, 임의의 `qwen3_coder` 이름과 혼동하지 않음 |
| `generation-config` | `vllm` | 모델 디렉터리의 generation_config.json에서 숨은 기본값/`max_new_tokens` 상한이 적용되는 것을 피하고 요청별 제한을 명확히 하기 위한 설정 |

일반 채팅 성공만으로 IDE 도구 호출이 검증되지는 않습니다. `tools`, `tool_choice`, streaming 응답을 실제로 확인해야 합니다. Qwen의 tokenizer/chat template를 사용하고, Bob 호환 metadata가 Laguna여도 vLLM template를 Laguna용으로 바꾸지 않습니다. 단순 텍스트 답변을 function call로 오해하지 말고 응답에 구조화된 `tool_calls`와 파싱 가능한 JSON arguments가 있는지 확인합니다.

`generation-config: vllm`은 Qwen 제작자의 sampling 권장값을 자동으로 적용하지 않게 합니다. 품질 목표에 맞는 temperature/top_p 등은 클라이언트 요청 또는 검증된 서버 정책에 명시합니다. 현재 Bob이 어떤 sampling 인자를 보내는지는 별도 확인 사항입니다. `max_output_tokens`를 8192로 광고했다고 모든 요청이 정확히 8192 tokens를 출력하거나 그 값이 항상 서버에 전달되는 것은 아닙니다.

이 예제는 `trust-remote-code`, reasoning parser, RoPE override를 추가하지 않습니다. 모델의 원래 구현과 context 범위, Qwen3-Coder의 template를 사용하고 실제로 필요한 옵션만 추가합니다. RoPE 설정을 늘리는 것으로 KV 메모리 부족을 해결할 수 없습니다.

### Eager 실행과 메모리·속도 절충

`enforce-eager: true`는 vLLM 0.31.0에서 **torch.compile과 CUDA graph를 모두 비활성화**하고 eager 실행을 사용합니다. graph capture/compile 관련 초기화와 graph 메모리 부담을 피하는 방향이므로, 단일 GPU에 큰 모델과 긴 KV cache를 함께 운용할 때 고려할 수 있습니다. CUDA graph의 실행 overhead 절감 효과도 포기하므로 decode 처리량이 낮아질 수 있습니다.

이 설정이 반드시 필요했던 원래 장애 원인은 제공되지 않았습니다. 현재 값의 효과를 설명할 수는 있지만 과거에 graph OOM이 발생했었다고 단정하지 않습니다. 로그의 약 16.6~17.0 tokens/s가 이 옵션 하나 때문에 결정됐다고 볼 수도 없습니다. 모델·커널·GPU·동시성·문맥 길이가 함께 영향을 줍니다.

현재 실행 구성을 기준으로 유지한 뒤, 속도 튜닝이 필요할 때만 같은 조건에서 `false`와 비교합니다. 재기동 peak, 실제 KV capacity, cold/warm 요청의 지연·처리량을 함께 기록하고 메모리 오류가 나면 원래값으로 복원합니다. graph 설정 변경은 작은 무위험 수정으로 간주하지 않습니다.

## 6. 단계별 프로파일

| 단계 | 총 context | KV dtype | 동시 sequence | 목적 |
|---|---:|---|---:|---|
| 초기 기능 확인 후보 | 32768 | `auto` | 1 | FP8 KV 영향 없이 chat/tool 동작 확인. 이 값도 GPU 로딩/프로파일링으로 검증 |
| 사용자 제공 현재 구성 | 131072 | `fp8` | 2 | 0.92 memory / 4096 batch / eager true. 짧은 Bob 작업 관찰, 최대 길이·동시성 검증은 미완료 |
| 운영 튜닝 후보 | 검증한 한도 | 검증한 dtype | 현재 2를 기준으로 측정 | 두 개의 긴 요청과 background 부하를 포함해 지연/처리량 측정 |

서로 다른 단계의 Bob 한도를 혼용하지 않습니다. 예를 들어 vLLM을 32768로 낮췄다면 Bob 입력+출력 예산도 그 이하로 낮추고 system/tool overhead를 확인합니다. KV dtype와 context를 함께 변경하면 실패 원인 구분이 어려우므로 가능한 한 하나씩 비교합니다.

## 7. 반영과 실제 적용값 확인

Kubernetes/OpenShift에서 ConfigMap을 수정했다면 [Deployment 반영·롤백](deployment-rollout.md)의 경로로 실행합니다. VM/systemd 또는 Docker 직접 실행이면 기존 관리자의 설정 원본을 수정하고 같은 관리자로 재시작합니다. 같은 GPU에 두 번째 `vllm serve`를 추가로 실행하지 않습니다.

다음 항목을 모두 확인합니다.

- 이미지/패키지 버전이 실제 0.31.0인지.
- 실행 명령이 새 `--config`를 읽는지, 오래된 CLI override가 없는지.
- 실제 로드 checkpoint, dtype, quantization, max model length.
- 사용 가능한 KV cache tokens와 최대 context에서의 capacity/concurrency 추정 로그.
- `/v1/models`의 ID와, 제공되는 경우 `max_model_len`. 이 필드가 없는 버전/프록시에서는 시작 로그와 실제 경계 요청으로 확인.
- 짧은 추론, tool call, streaming, 단계별 긴 입력 결과.

전체 운영 로그와 실제 URL/API key가 포함된 설정은 공개 저장소에 추가하지 않습니다.
