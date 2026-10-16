# 자료, 확인 상태와 문서 유지 기준

## 1. 문서의 사실을 읽는 기준

| 구분 | 내용 | 확인 근거/한계 |
|---|---|---|
| 최초 설치 성공 이력 | IBM Bob 2.0.0, OCP 4.20.36, Laguna 외부 모델과 Windows IDE 추론 | 제공된 설치 절차서의 성공 기록. 이 저장소 작업 중 실제 클러스터에서 재실행한 결과가 아님 |
| 후속 모델 변경 | `Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8` | 참조 대화의 적용 설정과 사용자의 후속 설명 |
| 현재 장비/버전 | L40S 1장, vLLM 0.31.0 | 사용자 직접 확인 |
| vLLM 실행 전문 | `vllm serve --config config.yaml`과 전체 YAML 확보 | 사용자 후속 메시지. 주소를 마스킹하고 공개용 YAML에 반영 |
| 128K | 제공된 YAML의 `max-model-len: 131072` | 짧은 작업 로그는 확인. 실제 최대 길이 요청과 전체 KV capacity는 미확인 |
| alias 장애 | backend 1.26.7에서 `qwen/qwen3-coder/30b-a3b` tuple 거부 | 참조 대화의 panic 로그. 모든 버전에 대한 지원 여부를 의미하지 않음 |
| 호환 metadata 후보 | `poolside/laguna/s2.1`을 유지하고 실제 backend만 Qwen으로 변경 | 최초 설치에서 인정된 tuple을 이용한 설명용 경로. Qwen 연결 관찰은 있으나 현재 tuple와 이 예제의 실적용 여부 미확인 |
| vLLM 현재값 | 0.92 memory, 2 sequences, 4096 batch tokens, FP8 KV, eager true 등 | 사용자가 전체 YAML 제공. 이전 후보 0.90/1/2048을 대체 |
| 사용자 작업 관찰 | 생성 약 16.6~17.0 tokens/s, KV 5.7~6.8%, prefix hits 89.3~89.8%, Bob 도구 화면 | 원문 로그와 화면을 요약. 전체 benchmark/최대 길이/코드 정상 종료의 증거는 아님 |
| 작성자의 직접 검증 | 문서, YAML 구조/값, 링크와 명령 문법 | 온프레미스 원격 실행은 하지 않았음. 사용자 제공 관찰과 직접 수행한 검증을 구별 |

사용자 제공 실적용값과 제안값을 분리해 기록합니다. 전체 실행 설정이 확보된 vLLM과, 아직 최종 metadata가 미확인인 Bob 호환 예제를 구별합니다. 민감값 없는 작업 해석은 [관찰 결과](observed-qwen-run.md)를 참고하세요.

## 2. 원본 자료

- 제공된 IBM Bob 2.0.0 On-Premises 설치 절차서.
- 제공된 IBM Bob 2.0.0 IDE 연결 가이드.
- 사용자가 참조한 설치 대화의 적용 설정과 오류 로그.
- 현재 대화에서 사용자가 확인한 모델, GPU와 vLLM 버전.
- 후속으로 제공한 vLLM 전체 YAML, 달력 생성 요청의 로그 및 Bob context/도구 실행 화면.

초기에는 참조 대화의 일부 답변이 본문 대신 참조 표기로 제공되어 실행 구성을 복원할 수 없었습니다. 이후 사용자가 vLLM 실행 명령과 설정 전문을 직접 제공해 해당 미확보 상태를 해소했습니다. Bob 최종 우회 metadata는 여전히 미확인입니다. 민감정보가 들어 있는 원본 문서·로그·화면은 업로드하지 않고 필요한 사실만 마스킹·요약합니다.

## 3. 버전별 공식 근거

아래 vLLM 링크는 모두 **v0.31.0**으로 고정했습니다. 버전을 변경하면 해당 tag의 옵션/기본값과 실제 `--help`를 다시 대조합니다.

- [vLLM YAML 구성 및 CLI 우선순위](https://github.com/vllm-project/vllm/blob/v0.31.0/docs/configuration/serve_args.md)
- [ModelConfig: dtype, quantization, context, served name, generation config](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/config/model.py)
- [CacheConfig: GPU memory fraction, KV dtype와 cache budget](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/config/cache.py)
- [SchedulerConfig: batch token 예산, 동시 sequence, chunked prefill](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/config/scheduler.py)
- [Qwen3-Coder tool calling: qwen3_xml](https://github.com/vllm-project/vllm/blob/v0.31.0/docs/features/tool_calling.md)
- [FP8 KV cache와 calibration](https://github.com/vllm-project/vllm/blob/v0.31.0/docs/features/quantization/quantized_kvcache.md)
- [MultiModalConfig: language-model-only](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/config/multimodal.py)
- [실행 로그 지표와 cache 통계](observed-qwen-run.md)
- [Qwen 공식 모델 목록](https://github.com/QwenLM/Qwen3-Coder)

Qwen 공식 목록은 해당 30B/FP8 모델에 256K context를 기재합니다. 이 저장소의 128K는 단일 GPU 운영에서 검토하는 **서버 제한 목표**이며 모델 자체의 최대 길이라고 표시하지 않습니다. 실제 내려받은 checkpoint config, tokenizer, backend 지원과 GPU 용량은 별개로 확인합니다.

## 4. Notion 및 AI 재사용

- 버전과 확인 상태를 절차와 함께 옮깁니다. 호환 후보를 최종 성공 구성으로 요약하지 않습니다.
- 모델 변경 시 `model-switch`, `vllm-configuration`, `deployment-rollout`, `model-validation`과 예제를 함께 갱신합니다.
- 실제 모델 ID와 Bob 표시용 metadata를 구별합니다.
- 내부 IP/DNS, 사용자 이메일, 자격 증명, 전체 운영 로그는 공개 문서로 복사하지 않습니다.
- 검증 결과에는 날짜, GPU/버전, 적용값, 요청 토큰 수와 통과/실패를 기록하되 민감값은 대체합니다.
