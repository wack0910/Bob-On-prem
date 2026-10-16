# IBM Bob 2.0.0 온라인 설치

## 1. 목적과 적용 범위

이 문서는 온라인 OpenShift Container Platform 클러스터에 IBM Bob 2.0.0을 배포하고, 조직의 IDP와 외부 OpenAI 호환 vLLM 서비스를 연결하는 절차를 설명합니다. 참조 설치는 OCP 4.20.36에서 수행됐습니다. 다른 제품·클러스터 버전에서는 배포 번들에 포함된 템플릿과 IBM 문서를 우선 확인하세요.

최초 설치 문서의 Laguna는 과거 성공 이력입니다. 현재 변경 기준은 Qwen3-Coder-30B-A3B-Instruct-FP8 / L40S 1장 / vLLM 0.31.0이며, 새 구성과 검증 절차는 [모델 변경 가이드](model-switch.md)를 함께 사용합니다.

에어갭 설치, 레지스트리 미러링, 프라이빗 인증서 발급 절차는 다루지 않습니다.

## 2. 구성 흐름

```text
Windows Bob IDE
    -> HTTPS Bob Gateway Route
        -> 조직 IDP (사용자 인증)
        -> Bob inference gateway
            -> 조직이 운영하는 외부 vLLM endpoint
```

Bob Operator와 Bob 인스턴스는 별도 namespace에 설치합니다. 참조 설치에서 사용한 namespace 이름은 `ibm-bob-operators`와 `ibm-bob`입니다. 조직 표준이 다르면 설치 전에 이름과 모든 후속 명령을 함께 조정하세요.

## 3. 사전 조건

| 항목 | 확인 사항 |
|---|---|
| OpenShift | 대상 클러스터와 현재 `oc` context가 맞는지 확인합니다. cluster-scoped 리소스를 적용할 권한이 필요합니다. |
| Bastion 도구 | `oc`, Helm 3.14 이상, Bash, `openssl`, `curl`, `jq`를 준비합니다. |
| IBM 이미지 레지스트리 | 온라인 경로에서 IBM Container Registry에 접근할 수 있고 유효한 IBM Entitlement Key가 준비되어야 합니다. |
| 스토리지 | 공유 파일용 RWX StorageClass와 데이터베이스용 RWO StorageClass의 이름 및 PVC 생성 상태를 확인합니다. 참조 환경은 CephFS와 Ceph RBD를 사용했습니다. |
| 인증서 | `cert-manager`와 사용할 Issuer/ClusterIssuer가 준비되어야 합니다. 클라이언트와 서비스가 인증서 체인을 신뢰하도록 합니다. |
| IDP 및 DNS | 조직 IDP 도메인, 사용자 프로비저닝 절차, IDE 사용자의 Gateway DNS 및 방화벽 경로를 확인합니다. 실제 도메인과 사용자는 문서나 커밋에 기록하지 않습니다. |
| 모델 endpoint | Bastion과 OCP workload에서 endpoint에 접근 가능해야 합니다. endpoint가 OpenAI 호환 `/v1/models`를 제공하고 사용할 모델 ID를 노출하는지 확인합니다. |

기본 도구 확인:

```bash
oc whoami
oc version
helm version
bash --version
openssl version
curl --version
jq --version
```

모델 endpoint 확인 (내부 주소와 모델 ID는 승인된 값으로 대체):

```bash
set -o pipefail
curl --fail --silent --show-error \
  "<VLLM_BASE_URL>/v1/models" |
  jq '.data[] | {id, max_model_len: (.max_model_len // "not_exposed")}'
```

여기에 실제 내부 URL, IP 또는 토큰이 포함된 명령 출력은 저장소에 추가하지 마세요.

## 4. 설치 번들 준비

IBM Bob 2.0.0의 공식 GitHub release에서 배포 번들을 Bastion의 보호된 작업 디렉터리에 내려받습니다. 제공된 체크섬이 있으면 다운로드한 파일과 대조합니다.

```bash
mkdir -p "$HOME/ibm-bob/release"
cd "$HOME/ibm-bob/release"

wget https://github.com/IBM/ibm-bob/releases/download/v2.0.0/ibm-bob-install-20260924-163534.tgz
# Release에서 체크섬을 제공하면 공식 값과 대조한 다음 압축을 풉니다.
tar -xzf ibm-bob-install-20260924-163534.tgz

./bobctl --help
```

실제 작업 경로는 조직의 관리 정책에 맞춰 정합니다. release 파일의 이름이나 버전이 변경되었으면 현재 공식 release 자산을 확인하고 명령을 조정하세요.

## 5. 설치 설정

배포 번들의 템플릿을 복사하고, 번들 버전에 맞는 실제 key 이름을 확인하여 필요한 값만 설정합니다.

```bash
cp config-template.yaml config.yaml
```

최소한 다음 설정을 환경에 맞게 확인합니다.

| 설정 | 의미 |
|---|---|
| `global.operatorNamespace` | Operator namespace |
| `global.instanceNamespace` | Bob 인스턴스 namespace |
| `global.fileStorageClass` | RWX 파일 StorageClass |
| `global.blockStorageClass` | RWO 블록 StorageClass |
| `global.imagePullSecretName` | IBM 이미지 pull secret 이름. 생성 방법과 기본값은 배포 번들을 확인합니다. |
| `bob.rag.enabled` | RAG를 사용하지 않으면 `false`; 활성화 시 관련 workload와 저장소 요구 사항을 확인합니다. |
| `bob.zProxy.enabled` | 선택 기능입니다. 활성화하면 RAG도 활성화되는 종속성을 확인합니다. |
| `bob.understand.enabled` | 선택 workload 사용 여부 |

`bob.modelGateway`의 실제 키와 secret 연결 방식은 같은 release의 `config-template.yaml`을 기준으로 합니다. API key, 토큰, 패스워드, 인증서 내용이 들어갈 수 있는 `config.yaml`은 임시 입력으로만 취급하고 Git에 올리지 않습니다.

```bash
# 안전한 저장소 전용 예시가 아니라, Bastion의 로컬 작업 파일입니다.
# 생성된 파일은 저장소 밖에서 보관하고 접근 권한을 제한합니다.
chmod 600 config.yaml
```

## 6. Cluster-scoped 리소스와 설치

생성될 매니페스트를 먼저 검토합니다. 대상 클러스터와 namespace가 맞는지 확인한 뒤 적용합니다.

```bash
./bobctl generate-cluster-resources
oc apply -f work/cluster-resources.yaml --server-side --force-conflicts
```

유효한 IBM Entitlement Key로 번들에서 안내하는 설치 명령을 실행하고 라이선스에 동의합니다. 키를 명령 기록, 로그, 티켓, Git 파일에 복사하지 마세요.

대화형 터미널에서 자격 증명을 숨겨 입력하면 명령 기록에 실제 값을 직접 적지 않아도 됩니다.

```bash
read -r -s -p 'Registry credentials (cp:<IBM_ENTITLEMENT_KEY>): ' REGISTRY_CREDS
printf '\n'
./bobctl install --registry-creds "$REGISTRY_CREDS" --accept-license
unset REGISTRY_CREDS
```

`bobctl`이 자격 증명을 명령 인자로 받으므로 실행 중 프로세스 메타데이터에 값이 노출될 수 있습니다. 승인된 Bastion에서 실행하고, 조직의 보안 기준에 맞는 입력 방법을 확인하세요. 실제 키를 명령 텍스트, 셸 기록, 로그, 설정 파일에 저장하지 않습니다.

## 7. 배포 상태 확인

```bash
oc get bob -n ibm-bob
oc get pods -n ibm-bob
oc get pods -n ibm-bob-operators
oc get route -n ibm-bob
```

주요 Pod가 Ready이고 Bob 리소스가 Ready 상태인지 확인합니다. `Running`만으로 초기화 성공을 판단하지 말고 재시작 횟수와 최근 로그도 확인합니다.

## 8. IDP 설정

배포 번들의 IDP 템플릿을 복사하고, 승인된 IDP 도메인 및 필요한 사용자/그룹 매핑을 설정합니다. 설치 자료의 예시에는 필드명이 잘못 입력된 것으로 보이는 항목이 있으므로, 필드명과 YAML 구조를 추정하지 말고 해당 버전의 `config-idp-template.yaml`을 기준으로 확인하세요.

```bash
cp config-idp-template.yaml "<LOCAL_IDP_CONFIG>.yaml"
# "<LOCAL_IDP_CONFIG>.yaml"을 조직의 승인된 IDP 값으로 편집합니다.
./bobctl configure-idp --config "<LOCAL_IDP_CONFIG>.yaml"
```

이 설정 파일에는 이메일 도메인이나 관리 사용자 정보가 들어갈 수 있습니다. 실사용 파일을 커밋하지 않습니다.

```bash
oc get bobidp -n ibm-bob
oc get bobidp "<BOB_IDP_RESOURCE>" -n ibm-bob -o yaml
```

## 9. 외부 모델 Gateway 설정

외부 vLLM이 OpenAI 호환 API를 제공해야 합니다. 아래 두 시점의 설정을 섞지 않습니다.

| 항목 | 최초 설치 이력 | 현재 Qwen 변경 기준 |
|---|---|---|
| 실제 모델 | `poolside/Laguna-XS-2.1-INT4` | `Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8` |
| vLLM 총 context | `16384` | `131072` 목표, GPU profiling/요청 검증 필요 |
| Bob 입력/출력 | `12288` / `2048` | 변경 시점 자료: `114688` / `8192`, 최종 적용값 재확인 |
| Bob metadata | `poolside / laguna / s2.1` | native Qwen tuple 거부 관찰. 호환 metadata 경로는 별도 검증 |

Bob 논리 이름 `premium-ide`, `security`, `background`는 유지하면서 `openai_compatible.model`을 새 vLLM ID와 일치시킵니다. `model_info`는 Bob alias table에서 인정되는 값이어야 하며, 현재 관찰된 Qwen 등록 실패를 무시하고 native tuple을 복사하지 않습니다.

전체 YAML과 각 필드의 이유는 [모델 변경](model-switch.md), vLLM 파라미터는 [vLLM 설정](vllm-configuration.md)을 따릅니다. 기존 성공 이력의 metadata를 Qwen 연결에 재사용하는 예제는 임시 호환 후보이며 공식 Qwen 등록이나 실제 성공 결과로 간주하지 않습니다.

새 vLLM을 먼저 검증한 뒤 Bob의 지원되는 구성 경로로 반영합니다. 실제 endpoint·API key가 들어간 파일은 Bastion의 보호된 경로에만 둡니다.

```bash
./bobctl update-model-config --model-config "<PROTECTED_MODEL_CONFIG_PATH>"
oc rollout restart deployment/bob-inference -n ibm-bob
oc rollout status deployment/bob-inference -n ibm-bob --timeout=10m
```

자동 rollout 여부와 이전 설정 복원까지 포함한 절차는 [Deployment 반영·복구](deployment-rollout.md)를 참고합니다.

## 10. 완료 기준

- Bob Custom Resource가 Ready입니다.
- Operator, Gateway, inference와 필수 의존 Pod가 Ready이고 반복 재시작하지 않습니다.
- IDP 리소스가 적용되고 승인된 테스트 사용자가 로그인할 수 있습니다.
- Bob inference의 모델 정보 endpoint에 기대한 모델 alias가 모두 등록됩니다.
- Bob IDE에서 실제 요청을 보내면 응답이 반환되고 해당 요청이 inference 측 로그에서 확인됩니다.

모델 시작 오류가 해결되기 전에는 IDE 로그인 성공만으로 inference 연결을 검증했다고 판단하지 마세요.
