# 모델 설정 반영, Deployment rollout과 복구

## 1. 어떤 변경이 어떤 재시작을 요구하는가

| 변경 | 반영 대상 | 확인 사항 |
|---|---|---|
| Bob `model-gateway.yaml` | `bobctl update-model-config` → `bob-inference` | 설정 적용 뒤 프로세스가 새 구성을 읽었는지 확인 |
| vLLM YAML만 변경 | ConfigMap/설정 원본 → vLLM 프로세스 | ConfigMap 갱신 자체는 Deployment rollout을 만들지 않음 |
| vLLM 이미지·command·args·mount 변경 | Deployment의 Pod template | template 변경은 보통 자동으로 새 ReplicaSet과 rollout 생성 |
| IDE `gatewayUrl` | 사용자 IDE 재실행 | Gateway 주소가 같다면 모델 변경만으로 바꿀 필요 없음 |

이 문서의 vLLM `oc` 명령은 vLLM이 Kubernetes/OpenShift의 **Deployment**로 운영될 때 적용합니다. 외부 VM의 systemd/Docker 프로세스라면 해당 관리자의 설정 변경·재시작 절차를 사용하고, Bob inference 부분만 이 문서대로 진행합니다.

## 2. 작업 변수와 변경 전 백업

아래 placeholder를 실제 환경의 값으로 바꿉니다. 실제 값이 있는 파일은 저장소 밖에서 보관합니다. 명령은 Bash 기준이며 다른 터미널에서는 변수를 다시 설정해야 합니다.

```bash
set -euo pipefail
umask 077

BOB_NS='ibm-bob'
VLLM_NS='<VLLM_NAMESPACE>'
VLLM_DEPLOY='<VLLM_DEPLOYMENT_NAME>'
VLLM_CONFIGMAP='<VLLM_CONFIGMAP_NAME>'
VLLM_SELECTOR='<VLLM_POD_LABEL_SELECTOR>'
BOB_RELEASE_DIR='<BOB_RELEASE_DIRECTORY>'
BOB_MODEL_CONFIG='<PROTECTED_MODEL_GATEWAY_PATH>'
VLLM_CONFIG_FILE='<PROTECTED_VLLM_CONFIG_PATH>'

BACKUP_DIR="$HOME/ibm-bob/private-backups/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

# 변경 전 로컬 설정 원본을 저장합니다.
cp "$BOB_MODEL_CONFIG" "$BACKUP_DIR/model-gateway.previous.yaml"
oc get deployment bob-inference -n "$BOB_NS" -o yaml \
  > "$BACKUP_DIR/bob-inference.snapshot.yaml"

# vLLM도 Deployment로 운영할 때만 아래 백업을 실행합니다.
oc get deployment "$VLLM_DEPLOY" -n "$VLLM_NS" -o yaml \
  > "$BACKUP_DIR/vllm-deployment.snapshot.yaml"
oc get configmap "$VLLM_CONFIGMAP" -n "$VLLM_NS" \
  -o jsonpath='{.data.config\.yaml}' \
  > "$BACKUP_DIR/vllm-config.previous.yaml"
test -s "$BACKUP_DIR/vllm-config.previous.yaml"
```

ConfigMap의 data key를 `config.yaml`로 사용하는 예제입니다. 실제 key가 다르면 백업/생성/volume item의 key를 모두 맞추세요. 설정이 ConfigMap이 아니라 다른 관리 원본에 있으면 해당 원본을 백업합니다. `test -s`가 실패하면 백업이 성립하지 않은 상태이므로 key와 리소스를 먼저 확인합니다.

snapshot에는 실행 인자, 내부 주소, 환경변수 등 민감정보가 들어갈 수 있습니다. 보호된 백업용이며 Git 파일이 아닙니다. 서버가 생성한 `status`, resourceVersion 등을 포함하므로 snapshot 전체를 무조건 `oc apply`하지 않습니다. 되돌릴 수 있도록 Helm values/Operator CR/GitOps 원본, 이전 이미지 digest와 설정 파일도 같은 보호 경로에 확보합니다.

## 3. 현재 Deployment와 실행 경로 확인

```bash
oc whoami
oc get deployment "$VLLM_DEPLOY" -n "$VLLM_NS"
oc get deployment "$VLLM_DEPLOY" -n "$VLLM_NS" \
  -o jsonpath='{.spec.strategy}{"\n"}'
oc get pods -n "$VLLM_NS" -l "$VLLM_SELECTOR" -o wide
oc rollout history deployment/"$VLLM_DEPLOY" -n "$VLLM_NS"
```

기존 Pod의 이미지, GPU request/limit, 모델 PVC, command/args와 설정 mount 경로를 보호된 관리 화면에서 확인합니다. Helm/Operator/GitOps가 관리하면 해당 원본에서 수정합니다. 하위 Deployment를 직접 patch하면 controller가 원복할 수 있습니다.

실행 경로는 다음과 같은 구조여야 합니다. 아래는 **기존 Deployment의 일부를 설명하는 조각**이며 전체 매니페스트가 아닙니다. 나머지 보안 설정·volume·Service·probe를 유지합니다.

```yaml
spec:
  template:
    spec:
      containers:
        - name: vllm
          command: ["vllm"]
          args: ["serve", "--config", "/etc/vllm/config.yaml"]
          resources:
            limits:
              nvidia.com/gpu: 1
          volumeMounts:
            - name: vllm-config
              mountPath: /etc/vllm
              readOnly: true
      volumes:
        - name: vllm-config
          configMap:
            name: "<VLLM_CONFIGMAP_NAME>"
```

컨테이너 이미지의 기존 entrypoint가 다르면 이 조각을 그대로 덮어쓰지 말고 같은 목적의 실행 경로로 조정합니다. CLI에 모델·메모리 옵션이 추가로 남아 있으면 YAML보다 우선하므로 함께 정리합니다.

## 4. L40S 1장: rollout 전략 먼저 결정

기존 vLLM Pod가 GPU 1개를 사용하고, 전체 사용 가능한 GPU도 1개이면 두 번째 GPU Pod를 동시에 실행할 수 없습니다. `replicas: 1`에서 RollingUpdate의 surge가 새 Pod를 먼저 만들고 기존 Pod를 계속 살려 두도록 설정되어 있으면 새 Pod는 `Insufficient nvidia.com/gpu`로 Pending에 머물 수 있습니다. 이것은 설정 로딩 오류와 다릅니다.

해당 배포의 재시작에 중단 시간이 허용된다면 관리 원본에서 **Recreate**를 사용합니다. 기존 Pod 종료, GPU 반환, 새 Pod 스케줄, 모델 로딩과 준비 완료까지 추론이 중단될 수 있습니다.

독립 관리 Deployment에서 같은 변경을 명령으로 표현하면 다음과 같습니다. Operator/Helm/GitOps 관리 리소스에는 그 관리 원본에서 같은 전략을 설정합니다.

```bash
oc patch deployment "$VLLM_DEPLOY" -n "$VLLM_NS" --type=merge \
  -p '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'
```

다른 선택은 RollingUpdate에 `maxSurge: 0`, `maxUnavailable: 1`을 설정하는 것입니다. 역시 단일 replica의 무중단 전환은 보장되지 않습니다. 무중단 모델 교체가 필요하면 별도 GPU capacity와 트래픽 전환 구조가 필요합니다. GPU 공유/time-slicing은 두 FP8 모델과 KV cache의 메모리가 동시에 충분하다는 보장이 아닙니다.

## 5. vLLM 설정 적용

`VLLM_CONFIG_FILE`에는 자리표시자를 대체하고 검토한 **새** 설정 파일을 지정합니다. 예제 파일을 읽기 전에 [파라미터 설명](vllm-configuration.md)을 확인합니다.

독립 관리 ConfigMap일 때 다음 명령으로 data key `config.yaml`을 생성/갱신할 수 있습니다.

```bash
set -o pipefail
oc create configmap "$VLLM_CONFIGMAP" -n "$VLLM_NS" \
  --from-file="config.yaml=$VLLM_CONFIG_FILE" \
  --dry-run=client -o yaml |
  oc apply -f -

oc rollout restart deployment/"$VLLM_DEPLOY" -n "$VLLM_NS"
oc rollout status deployment/"$VLLM_DEPLOY" -n "$VLLM_NS" --timeout=20m
```

원래 ConfigMap이 여러 data key를 갖는다면 모두 보존하는 관리 원본에서 수정합니다. 위 예제의 단일 파일 ConfigMap 방식으로 다른 key까지 갱신/삭제하지 않도록 현재 소유권과 apply 관리 이력을 확인하세요. 인증 값은 별도 Secret 등 기존 비밀 관리 경로를 유지합니다.

Deployment 매니페스트의 Pod template를 변경하여 자동 rollout이 이미 진행 중이면 먼저 `rollout status`로 기다립니다. ConfigMap만 바뀌면 volume 갱신 여부와 관계없이 vLLM이 실행 시 읽는 설정을 다시 읽도록 프로세스 재시작이 필요합니다. ConfigMap 이름을 새 버전으로 바꾸고 Pod template에서 참조를 바꾸는 관리 방식도 가능합니다.

20분은 예시 대기 한도입니다. 모델 load/compile/profiling 시간이 더 길면 실제 측정값에 맞춰 조정합니다. `rollout status` timeout은 rollout을 취소하지 않으며, Pod가 이후에도 계속 실행/시작될 수 있습니다. startup probe가 정상 로딩 도중 컨테이너를 재시작시키지 않는지도 확인합니다.

## 6. 새 vLLM을 먼저 검증

```bash
oc get pods -n "$VLLM_NS" -l "$VLLM_SELECTOR"
oc logs -n "$VLLM_NS" "<NEW_VLLM_POD>" --since=20m
```

`oc logs deployment/...`는 rollout 중 이전 Pod를 선택할 수 있으므로 새 Pod 이름을 확인합니다. startup log의 checkpoint, 총 context, KV capacity, FP8/kernel 오류와 `/v1/models`를 확인합니다. `Running` 또는 포트 열림만으로 완료하지 않고 직접 chat 및 tool call을 검사합니다. [검증 절차](model-validation.md)를 따릅니다.

## 7. Bob 모델 구성 반영

새 vLLM이 정상일 때 검토한 Bob 설정을 반영합니다. 실제 API model ID와 endpoint, 세 alias 모두가 맞아야 합니다.

```bash
cd "$BOB_RELEASE_DIR"
./bobctl update-model-config --model-config "$BOB_MODEL_CONFIG"

# 자동 rollout이 없거나 새 프로세스가 아직 설정을 읽지 않은 경우 재시작합니다.
oc rollout restart deployment/bob-inference -n "$BOB_NS"
oc rollout status deployment/bob-inference -n "$BOB_NS" --timeout=10m
oc get pods -n "$BOB_NS"
oc logs -n "$BOB_NS" deployment/bob-inference --since=10m
```

이미 자동 rollout이 발생하면 완료를 기다리고 재차 restart하지 않습니다. Bob model-info 등록, 짧은 IDE 응답, 도구 호출, 긴 입력 순으로 확인합니다. 호환 metadata 사용 시 UI 모델명보다 실제 vLLM 요청의 model ID를 대조합니다.

## 8. 실패 진단

| 상태 | 먼저 확인할 것 |
|---|---|
| 새 GPU Pod가 Pending | GPU 자원 부족, rollout strategy, 기존 Pod 종료, node selector/toleration, PVC attach |
| vLLM CrashLoopBackOff | 이전 로그, checkpoint mount, CLI 덮어쓰기, parser 이름, GPU 메모리, dtype/kernel 호환성 |
| Pod가 로딩 중 반복 종료 | startup/liveness probe, OOMKilled, 모델 load 시간과 노드 자원 |
| Bob 시작 시 alias panic | `model_info` tuple와 현재 Bob alias table; 토큰 길이 오류와 구별 |
| Ready인데 추론 오류 | 요청 model ID, Service 경로, 인증/TLS, context 합계, tool parser |

```bash
oc describe pod -n "$VLLM_NS" "<NEW_VLLM_POD>"
oc logs -n "$VLLM_NS" "<NEW_VLLM_POD>" --previous --tail=200
oc get events -n "$VLLM_NS" --sort-by=.metadata.creationTimestamp
```

`--previous`는 이전 컨테이너가 존재할 때만 유효합니다. 관찰 명령이 실패하면 Pod 이름/namespace/컨테이너 이름부터 확인합니다. 로그에 있는 내부 정보는 마스킹하여 공유합니다.

## 9. 롤백: 설정 원본도 복원

`oc rollout undo`는 Deployment의 Pod template revision을 되돌리는 기능입니다. **동일 이름의 ConfigMap 내용을 이전 버전으로 복원하지 않습니다.** ConfigMap만 수정한 뒤 undo/restart하면 새 프로세스가 계속 잘못된 설정을 읽을 수 있습니다.

### 9.1 vLLM 복원

먼저 관리 원본에서 이전 설정·이미지·args·volume 참조를 한 쌍으로 복구합니다. 동일한 단일 key ConfigMap을 사용했다면 **새 Pod를 시작하기 전에** 이전 파일 내용을 복원합니다.

```bash
set -o pipefail
test -s "$BACKUP_DIR/vllm-config.previous.yaml"
oc create configmap "$VLLM_CONFIGMAP" -n "$VLLM_NS" \
  --from-file="config.yaml=$BACKUP_DIR/vllm-config.previous.yaml" \
  --dry-run=client -o yaml |
  oc apply -f -
```

그 다음 실제 변경 범위에 맞는 한 가지 rollout 경로를 선택합니다.

- **Pod template도 변경한 독립 Deployment**: 이전 이미지/args/volume 참조가 포함된 known-good revision으로 undo하고 기다립니다. 그 revision이 참조하는 ConfigMap/Secret도 올바른 내용인지 먼저 확인합니다.

  ```bash
  oc rollout history deployment/"$VLLM_DEPLOY" -n "$VLLM_NS"
  oc rollout undo deployment/"$VLLM_DEPLOY" -n "$VLLM_NS" \
    --to-revision="<KNOWN_GOOD_REVISION>"
  oc rollout status deployment/"$VLLM_DEPLOY" -n "$VLLM_NS" --timeout=20m
  ```

- **ConfigMap 내용만 변경**: 이전 내용을 복원한 후 재시작합니다.

  ```bash
  oc rollout restart deployment/"$VLLM_DEPLOY" -n "$VLLM_NS"
  oc rollout status deployment/"$VLLM_DEPLOY" -n "$VLLM_NS" --timeout=20m
  ```

Helm/Operator/GitOps 배포는 같은 복구 내용을 관리 원본에 반영합니다. 잘못된 설정으로 중간 Pod가 다시 생기지 않도록 설정·template를 함께 복원하고 rollout을 한 번으로 묶습니다. 알려진 정상 모델을 다시 load한 뒤 `/v1/models`와 직접 추론까지 확인합니다.

### 9.2 Bob 복원

```bash
cd "$BOB_RELEASE_DIR"
./bobctl update-model-config \
  --model-config "$BACKUP_DIR/model-gateway.previous.yaml"
oc rollout restart deployment/bob-inference -n "$BOB_NS"
oc rollout status deployment/bob-inference -n "$BOB_NS" --timeout=10m
```

Bob만 이전 Laguna ID로 되돌리고 서버에는 Qwen만 남겨 두면 여전히 model ID 불일치가 납니다. **Bob이 요청하는 ID와 실제 서버가 제공하는 ID를 한 쌍으로 복원**합니다. 마지막으로 IDE 추론까지 확인하고 실패했던 설정, 원인, 복구 시각을 민감값 없이 기록합니다.
