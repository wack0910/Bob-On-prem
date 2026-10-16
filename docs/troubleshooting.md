# 트러블슈팅 기록

현재 변경 대상은 Qwen3-Coder-30B-A3B-Instruct-FP8이며 사용자 확인 환경은 L40S 1장 / vLLM 0.31.0입니다. 전체 변경 절차는 [모델 변경](model-switch.md), GPU와 엔진 옵션은 [vLLM 설정](vllm-configuration.md), 반영/복원은 [Deployment 가이드](deployment-rollout.md)를 따릅니다.

## 1. 알려진 사례: 모델 alias 검증 오류로 inference가 시작하지 않음

### 관찰된 상황

한 후속 설정에서 Bob inference backend `1.26.7`이 반복 재시작했습니다. 로그에서 확인된 핵심 오류는 다음 의미였습니다.

```text
invalid bifrost model_info: model "premium-ide" has
(provider="qwen", family="qwen3-coder", version="30b-a3b")
but no matching alias table entry exists
```

같은 시작 로그에는 router 하나, 모델 0개가 기록된 뒤 embedded gateway 초기화가 panic으로 종료되었습니다. Kubernetes 이벤트에는 container BackOff와 반복 재시작이 나타났습니다.

router 초기화 단계의 모델 개수만으로 최종 등록 상태를 판단하지 않습니다. 이 사례의 실패 근거는 alias mismatch panic과 프로세스 종료이며, 정상 등록 여부는 시작 완료와 model-info 응답으로 확인합니다.

이 오류는 128K 입력의 한계 테스트 결과가 아닙니다. 모델 alias metadata 검증이 실패해 inference HTTP server가 시작되기 전 발생한 초기화 오류입니다. 따라서 이 상태에서는 긴 컨텍스트 추론이 검증되지 않았습니다.

### 진단 및 조치 순서

1. Pod의 이전 로그와 재시작 상태를 확인합니다.

   ```bash
   oc logs -n ibm-bob "<BOB_INFERENCE_POD>" -c runtime --previous --tail=200
   oc get pod -n ibm-bob "<BOB_INFERENCE_POD>"
   ```

2. 배포에 실제 적용된 model gateway 설정에서 문제 alias의 `provider`, `family`, `version`을 확인합니다. 실제 ConfigMap 전체를 공개 로그나 저장소에 복사하지 않습니다.
3. 같은 Bob/backend 버전이 제공하는 alias table 또는 공식 모델 메타데이터 정의에서 정확히 지원되는 triple을 확인합니다.
4. 정식 Qwen alias가 제공되면 그 정의를 사용합니다. 현재 table에 없다면 기존 성공 metadata와 실제 Qwen endpoint를 구분하는 [임시 호환 경로](model-switch.md)를 검토할 수 있습니다. 이 저장소의 `poolside/laguna/s2.1`은 설명용 후보이며 최종 우회 성공 구성이 확인된 것은 아닙니다. 지원되는 custom alias 확장 방식은 해당 Bob 버전의 IBM 안내를 확인합니다.
5. 구성 업데이트 후 inference deployment를 재시작하고 rollout을 기다립니다.

   ```bash
   ./bobctl update-model-config --model-config "<PROTECTED_MODEL_CONFIG_PATH>"
   oc rollout restart deployment/bob-inference -n ibm-bob
   oc rollout status deployment/bob-inference -n ibm-bob --timeout=10m
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

## 3. Qwen 변경 후 자주 만나는 문제

| 증상 | 원인 후보 | 조치 |
|---|---|---|
| vLLM API의 `model not found` | Bob model ID와 vLLM served name 불일치 | `/v1/models`의 `id`와 `openai_compatible.model`을 일치시킴 |
| ConfigMap을 바꿔도 예전 context | CLI가 YAML을 덮어씀, 다른 mount/key, 재시작 누락 | 실제 Pod의 command/args와 `--config` 경로 확인 후 올바른 원본 갱신·rollout |
| Qwen 일반 채팅 성공, IDE 도구 실패 | parser/template 불일치, auto tool 기능 미활성화 | 0.31.0의 `qwen3_xml`과 Qwen template를 확인하고 구조화 tool call 테스트 |
| GPU Pod Pending | 단일 GPU를 기존 Pod가 점유, RollingUpdate surge | 이벤트 확인 후 중단 시간을 고려한 Recreate 또는 maxSurge 0 경로 사용 |
| vLLM 시작 시 KV 용량 부족 | 128K cache와 전체 FP8 가중치·runtime 메모리의 합이 가용량 초과 | 짧은 context부터 검증, KV dtype/동시성/batch와 profiling 로그 대조 |
| 요청 중 OOM 또는 preemption 증가 | 복수 긴 요청, 큰 prefill batch, 메모리 여유 부족 | sequence/batch budget 감소 후 실제 peak와 지연 측정 |
| FP8 가중치인데 cache가 큼 | 가중치 dtype와 KV dtype 혼동 | actual KV dtype 확인. FP8 KV 사용 시 scale/품질을 별도 검증 |
| 긴 입력은 수락되지만 문서 끝 내용 누락 | IDE/Gateway truncation, context budget/회수 품질 | 합성 표식과 실제 prompt tokens를 확인하고 길이별 결과 비교 |
| UI가 Laguna로 표시됨 | 임시 호환 metadata가 노출됨 | 실제 요청 Qwen ID를 확인하고 표시/계량 차이를 기록 |
| rollout undo 뒤에도 같은 오류 | 같은 ConfigMap 내용이 복구되지 않음 | 이전 설정 원본과 template/image를 함께 복원하고 재시작 |

실제 달력 작업에서 관찰된 생성 속도·prefix cache·IDE 실행 대기 해석은 [실행 관찰](observed-qwen-run.md)을 참고합니다. `python calendar.py`의 입력 대기나 표준 라이브러리 이름 충돌은 코드를 확인해 진단하며 서버 추론 실패로 단정하지 않습니다.

128K는 이 구성의 제공된 총 context 설정값입니다. 모델 자체 최대 길이나 한 장의 GPU가 보장하는 길이를 의미하지 않습니다. [검증 가이드](model-validation.md)의 직접 API→Bob→IDE→긴 입력 순서로 실패 구간을 좁힙니다.
