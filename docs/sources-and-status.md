# 자료, 확인 상태와 문서 유지 기준

## 1. 문서의 사실을 읽는 기준

| 구분 | 내용 | 확인 근거/한계 |
|---|---|---|
| 최초 설치 성공 이력 | IBM Bob 2.0.0, OCP 4.20.36, Laguna 외부 모델과 Windows IDE 추론 | 제공된 설치 절차서의 성공 기록. 이 저장소 작업 중 실제 클러스터에서 재실행한 결과가 아님 |
| 후속 모델 변경 | `Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8` | 참조 대화의 적용 설정과 사용자의 후속 설명 |
| 현재 장비/버전 | L40S 1장, vLLM 0.31.0 | 사용자 직접 확인 |
| 128K | 사용자가 한도를 128K로 상향했다고 보고 | 예제는 `131072` 총 context로 표현. 실제 KV capacity와 최종 실행 인자 전문 미확보 |
| alias 장애 | backend 1.26.7에서 `qwen/qwen3-coder/30b-a3b` tuple 거부 | 참조 대화의 panic 로그. 모든 버전에 대한 지원 여부를 의미하지 않음 |
| 호환 metadata 후보 | `poolside/laguna/s2.1`을 유지하고 실제 backend만 Qwen으로 변경 | 최초 설치에서 인정된 tuple을 이용한 설명용 경로. 최종 실적용 tuple와 Qwen 우회 성공 결과 미확인 |
| vLLM 수치 예제 | 0.90 memory, 1 sequence, 2048 batch tokens, FP8 KV 등 | 단일 GPU에서 평가할 후보. 기존 운영값이라고 단정하지 않음 |
| 현재 검증 범위 | 문서, YAML 구조/값, 링크와 명령 문법 | 실 GPU/클러스터의 부하, 모델 품질, rollout 성공을 증명하지 않음 |

예제와 사용자 확인값을 분리해 기록합니다. 실제 정상 실행 명령과 runtime 출력이 확보되면 민감값을 제거한 후 확인 상태와 관련 문서를 함께 갱신합니다.

## 2. 원본 자료

- 제공된 IBM Bob 2.0.0 On-Premises 설치 절차서.
- 제공된 IBM Bob 2.0.0 IDE 연결 가이드.
- 사용자가 참조한 설치 대화의 적용 설정과 오류 로그.
- 현재 대화에서 사용자가 확인한 모델, GPU와 vLLM 버전.

참조 대화에서 일부 이전 답변은 본문이 아닌 참조 표기로만 제공되어 마지막 vLLM 실행 명령과 최종 우회 설정 전체를 복원할 수 없었습니다. 원본 문서 파일과 화면 이미지는 계정·내부 주소·초기 암호가 포함되어 저장소에 업로드하지 않습니다.

## 3. 버전별 공식 근거

아래 vLLM 링크는 모두 **v0.31.0**으로 고정했습니다. 버전을 변경하면 해당 tag의 옵션/기본값과 실제 `--help`를 다시 대조합니다.

- [vLLM YAML 구성 및 CLI 우선순위](https://github.com/vllm-project/vllm/blob/v0.31.0/docs/configuration/serve_args.md)
- [ModelConfig: dtype, quantization, context, served name, generation config](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/config/model.py)
- [CacheConfig: GPU memory fraction, KV dtype와 cache budget](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/config/cache.py)
- [SchedulerConfig: batch token 예산, 동시 sequence, chunked prefill](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/config/scheduler.py)
- [Qwen3-Coder tool calling: qwen3_xml](https://github.com/vllm-project/vllm/blob/v0.31.0/docs/features/tool_calling.md)
- [FP8 KV cache와 calibration](https://github.com/vllm-project/vllm/blob/v0.31.0/docs/features/quantization/quantized_kvcache.md)
- [Qwen 공식 모델 목록](https://github.com/QwenLM/Qwen3-Coder)

Qwen 공식 목록은 해당 30B/FP8 모델에 256K context를 기재합니다. 이 저장소의 128K는 단일 GPU 운영에서 검토하는 **서버 제한 목표**이며 모델 자체의 최대 길이라고 표시하지 않습니다. 실제 내려받은 checkpoint config, tokenizer, backend 지원과 GPU 용량은 별개로 확인합니다.

## 4. Notion 및 AI 재사용

- 버전과 확인 상태를 절차와 함께 옮깁니다. 호환 후보를 최종 성공 구성으로 요약하지 않습니다.
- 모델 변경 시 `model-switch`, `vllm-configuration`, `deployment-rollout`, `model-validation`과 예제를 함께 갱신합니다.
- 실제 모델 ID와 Bob 표시용 metadata를 구별합니다.
- 내부 IP/DNS, 사용자 이메일, 자격 증명, 전체 운영 로그는 공개 문서로 복사하지 않습니다.
- 검증 결과에는 날짜, GPU/버전, 적용값, 요청 토큰 수와 통과/실패를 기록하되 민감값은 대체합니다.
