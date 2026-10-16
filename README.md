# IBM Bob on-premises deployment and IDE guide

IBM Bob 2.0.0의 온라인 OpenShift 설치, 외부 OpenAI 호환 vLLM 연결, Windows용 Bob IDE 연결 절차를 정리한 운영 가이드입니다. 이 문서는 제공된 설치 절차서, IDE 연결 가이드, 설치 후 트러블슈팅 대화에서 확인된 내용을 재구성했습니다.

## 문서

- [설치 가이드](docs/installation.md): 사전 조건, 설치 파일, OpenShift 리소스, IDP와 외부 모델 설정, 배포 확인
- [IDE 연결 가이드](docs/ide-setup.md): Windows IDE 설치, 신뢰할 인증서 등록, Gateway URL, 로그인과 연결 확인
- [모델 및 긴 컨텍스트 검증](docs/model-validation.md): 모델 등록, 실제 추론 확인, 긴 입력 테스트 방법
- [트러블슈팅](docs/troubleshooting.md): 알려진 모델 alias 시작 오류와 일반 진단 순서

## 범위와 기준

- 기준 설치: IBM Bob 2.0.0, OpenShift Container Platform 4.20.36, 온라인 설치, 외부 vLLM 연결
- IDE 연결 대상: Windows용 IBM Bob IDE와 조직의 Bob Gateway
- 저장소에는 재사용 가능한 절차와 예시만 둡니다. 클러스터 주소, 계정, 토큰, 비밀번호, 실제 설정 파일, 인증서와 운영 로그는 커밋하지 않습니다.
- 에어갭 설치, 프라이빗 레지스트리 미러 구성, 조직별 인증서 발급은 이 가이드 범위에 포함되지 않습니다.

## 사이트별 값

`<...>` 형태는 각 조직의 승인된 환경 값으로 바꾸어야 합니다. 예시를 실제 환경의 검증 없이 적용하지 마세요. `config.yaml`, IDP 설정, 모델 Gateway 설정은 배포 번들의 해당 버전 템플릿을 기준으로 작성하고, 민감한 값이 들어갈 수 있는 실파일은 저장소 밖의 보호된 위치에 둡니다.
