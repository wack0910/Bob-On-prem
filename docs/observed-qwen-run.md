# 사용자 제공 Qwen 실행 구성과 Bob 작업 관찰

## 1. 자료와 확인 범위

2026-10-16 사용자가 제공한 전체 vLLM YAML, 달력 프로그램 생성 요청의 서버 로그, Bob IDE 화면을 정리한 기록입니다. 장비/버전은 사용자 확인 기준 L40S 1장 / vLLM 0.31.0입니다. 작성자가 원격 서버에서 명령을 실행하거나 프로그램 코드를 검사한 결과는 아닙니다.

민감한 클라이언트 IP·접속 포트·프로세스 ID가 들어 있는 로그 원문과 화면 이미지는 커밋하지 않습니다. 아래에는 필요한 설정과 집계 수치만 기록합니다.

## 2. 실행 방식과 이전 예제 차이

사용자가 제공한 실행 명령:

```bash
vllm serve --config config.yaml
```

주소를 마스킹한 [전체 구성](../examples/vllm-qwen-l40s.example.yaml)과 [파라미터 설명](vllm-configuration.md)을 참고하세요.

| 항목 | 이전 문서의 후보 | 사용자 제공 현재값 |
|---|---|---|
| `model` | 로컬 checkpoint placeholder | `Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8` Hub ID |
| `gpu-memory-utilization` | `0.90` | `0.92` |
| `max-num-seqs` | `1` | `2` |
| `max-num-batched-tokens` | `2048` | `4096` |
| `pipeline-parallel-size` | 미기재 | `1` |
| `language-model-only` | 미기재 | `true` |
| `enforce-eager` | 미기재 | `true` |
| `max-model-len` | `131072` 목표 | `131072` 설정 확인 |
| `kv-cache-dtype` | `fp8` 후보 | `fp8` 설정 확인 |

Bob의 현재 `model_info.provider/family/version` 전문은 이번에도 제공되지 않았습니다. Qwen 연결 동작은 관찰되지만 저장소의 Laguna metadata 호환 예제가 실제로 적용된 tuple이라고 단정하지 않습니다.

## 3. 로그 수치 해석

| 관찰 | 해석 | 이 수치만으로 판단할 수 없는 것 |
|---|---|---|
| 생성 중 약 `16.6~17.0 tokens/s`, Running `1` | 해당 엔진의 로그 구간 평균 생성 처리량. 단일 활성 요청 구간이라 그 요청의 decode 속도를 대략 보여줌 | 모델 전반 벤치마크, 동시 2요청 속도, eager off 대비 손익 |
| 생성 중 prompt throughput `0.0` | 새 prefill 없이 기존 요청의 decode가 진행되는 구간일 수 있음 | 입력 미수신, 서버 멈춤 |
| prompt 처리량이 일부 구간에만 나타남 | 새 요청/후속 대화 처리와 구간 평균 집계의 영향 | cold prompt의 최대 prefill 처리량 |
| Running `0~1`, Waiting `0` | 샘플 시점에 활성 요청이 최대 한 개이고 대기열이 보이지 않음 | 전체 작업 동안 순간 대기가 전혀 없었음, 두 요청 동시 처리 검증 |
| KV cache usage `5.7~6.8%` | 할당한 KV cache block pool에서 사용 중으로 집계된 비율 | 전체 GPU VRAM 점유율, GPU 연산 사용률, 128K 중 실제 사용한 비율 |
| Prefix cache hit rate `89.3~89.8%` | prefix token 조회에서 높은 재사용 관찰 | 전체 응답의 cache 적중률, 총 지연 89.8% 단축, 모든 대화가 같은 적중률 |
| 요청 후 Running `0`, KV usage `0.0%` | 해당 시점에 활성 KV block 점유가 해제/재사용 가능 상태로 집계될 수 있음 | 서버 모델 unload, VRAM 반환, prefix cache 전체 삭제 |
| 여러 chat completion의 HTTP `200` | API 요청의 HTTP 응답 성공 관찰 | 모든 streaming body 정상 종료, 생성 코드 정상 동작, 도구 전체 작업 완료 |

vLLM 0.31.0의 로그용 prefix hit rate는 기본적으로 **최근 최대 1000개 요청을 모으는 창의 token hits / queried tokens**입니다. 매 10초의 성공 요청 비율이나 서버 기동 이후의 영구 누적 비율로 해석하지 않습니다. 요청 종료 후 해당 비율이 남아 있는 것은 통계 창이 유지되기 때문일 수 있으며 현재 cache 내용이 모두 남았다는 증거도 아닙니다.

`gpu-memory-utilization: 0.92`와 KV usage `6.8%`는 분모가 다릅니다. 모델 가중치 등 GPU 메모리 전체는 `nvidia-smi`와 메모리 지표로 보고, KV pool capacity와 점유는 엔진의 시작/실행 로그로 따로 확인합니다. `6.8% × 131072`로 현재 prompt tokens를 계산하지 않습니다.

약 16.8 tokens/s가 유지된다면 출력 1000 tokens의 decode만 대략 60초입니다. prefill, 네트워크, 파일/터미널 도구, 여러 차례의 모델 요청 시간은 추가되며 실제 요청 전체 지연과는 다릅니다. 한국어 글자 수와 token 수도 같지 않습니다.

## 4. Bob 화면의 context 표시

제공 화면에는 `16.4k / 114.7k`가 보입니다. 분모 `114.7k`는 이전 Bob 설정의 `max_input_tokens: 114688`을 십진 k로 반올림한 값과 일치합니다. vLLM 총 context `131072`와 Bob 입력 한도 `114688`은 다른 설정이므로 화면에 131k가 보이지 않는 것 자체는 오류 근거가 아닙니다.

단순 표시값 비율은 약 14.3%입니다. 이는 Bob이 표시한 context 사용량이며 vLLM의 실제 `usage.prompt_tokens`와 완전히 같다고 보장하지 않습니다. system prompt, 도구 정의, 이력과 파일 등도 입력 예산을 사용하므로 사용자가 짧게 요청했어도 전체 context가 커질 수 있습니다. 제공 화면만으로 16.4k의 정확한 구성 비율은 알 수 없습니다.

이번 기록의 약 16.4k 표시만으로 128K 경계 입력 성공을 판단할 수 없습니다. 입력 `114688` + 출력 상한 `8192` = `122880`으로, 총 `131072` 대비 `8192` 여유라는 이전 예산 구분도 유지합니다.

## 5. IDE가 실행 중인데 vLLM이 idle인 경우

화면에서는 파일 조회, `calendar.py` 작성, Python 실행, 코드 수정 뒤 다시 `python calendar.py` 실행이 진행 중입니다. 파일·터미널 도구가 실행된 경로는 관찰되지만 최종 정상 종료는 보이지 않습니다.

로그의 idle 구간과 IDE 화면이 같은 시점이라면 현재는 모델 응답보다 실행 도구의 완료를 기다리는 상황을 우선 확인합니다. 가능성은 다음과 같습니다.

- `input()`이나 메뉴 반복문이 사용자 입력을 기다림.
- GUI 이벤트 루프나 서버처럼 계속 살아 있어야 하는 프로세스를 동기 실행 도구가 기다림.
- 프로그램 오류 뒤 재시도 또는 하위 프로세스 종료 대기.

실행 중인 터미널 출력과 코드를 확인해야 원인을 확정할 수 있습니다. 단지 모델 추론 한도를 늘리거나 서버를 재시작해서 해결할 문제라고 판단하지 않습니다.

추가로 파일명이 `calendar.py`인 점은 표준 라이브러리 `calendar`와 이름이 같습니다. 코드가 `import calendar`를 사용한다면 자기 파일이 표준 모듈을 가릴 수 있습니다. 코드가 제공되지 않았으므로 실제 발생한 오류라고 단정하지 않지만, 이 경우 `calendar_app.py`처럼 이름을 바꾸고 import 경로를 확인합니다.

IDE 후속 요청 예시:

> 실행 중인 달력 테스트를 중지하고 터미널 출력을 확인해 줘. CLI 프로그램은 테스트 시 input()이나 무한 메뉴 대기 없이 종료되도록 해 줘. 파일명은 calendar_app.py로 하고, --year 2026 --month 10 같은 인자로 달력을 출력하는 비대화형 테스트를 실행해 종료 코드와 출력까지 확인해 줘. GUI 버전이면 앱을 띄우는 실행과 자동 종료되는 테스트를 구분해 줘.

## 6. 현재 확인된 것과 남은 검증

- **사용자 자료로 관찰**: 실제 설정 전문, chat completion HTTP 200, 활성 요청의 생성 진행, 높은 prefix 재사용, Bob의 파일 조회·작성·터미널 실행 경로.
- **미확인**: 달력 코드의 정확성/최종 종료, 최대 128K 경계, 두 개의 긴 요청 동시 처리, cold prefix 성능, FP8 KV 품질 비교, eager off와의 성능·메모리 비교, 현재 Bob metadata tuple.

다음 검증은 [모델 검증 가이드](model-validation.md)의 경계 입력·동시성 절차를 따릅니다. 현재 단일 작업의 낮은 KV 점유율만으로 context/concurrency 값을 더 높이지 않습니다.

## 7. 지표와 실행 옵션의 근거

- [0.31.0 ModelConfig: eager는 compile과 CUDA graph를 모두 비활성화](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/config/model.py)
- [0.31.0 MultiModalConfig: language-model-only](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/config/multimodal.py)
- [0.31.0 로그 구간 처리량과 scheduler 지표](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/v1/metrics/loggers.py)
- [0.31.0 CachingMetrics: 최근 요청 창의 token hit rate](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/v1/metrics/stats.py)
- [0.31.0 block pool의 KV cache 점유 계산](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/v1/core/block_pool.py)
