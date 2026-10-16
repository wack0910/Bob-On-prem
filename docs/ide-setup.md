# Windows용 IBM Bob IDE 연결

## 1. 준비 사항

- 조직의 운영자가 배포한 Bob Gateway HTTPS Route와 DNS에 접근할 수 있어야 합니다.
- 사용자 계정은 조직 IDP에서 발급받아야 합니다. 계정명·초기 암호를 문서나 Git에 저장하지 않습니다.
- 사설 CA를 사용하는 경우 인증서 파일을 승인된 IT/PKI 채널에서 받습니다. 이메일/메신저 첨부 파일만으로 신뢰 여부를 판단하지 말고 발급자와 지문을 별도 경로로 검증합니다.

## 2. IDE 설치

공식 다운로드 페이지에서 Windows용 IBM Bob IDE를 내려받아 설치합니다.

<https://bob.ibm.com/download>

다운로드 주소나 설치 파일이 조직 정책에 따라 다르면 승인된 배포 방법을 사용합니다.

## 3. Gateway 인증서 신뢰

조직에서 인증서 설치가 필요하다고 안내한 경우에만, 승인된 CA 인증서를 사용합니다. 아래 명령은 Windows 관리자 PowerShell에서 실행합니다.

```powershell
Set-Location "<CA_CERTIFICATE_DIRECTORY>"
certutil -addstore Root .\bob-ca.crt
```

CA 지문을 IT/PKI 담당자가 제공한 값과 먼저 대조하세요. 확인되지 않은 인증서를 Root store에 추가하지 마세요. 인증서 파일은 저장소에 커밋하지 않습니다.

## 4. Gateway URL 지정

Bob IDE에서 **Settings → General → Open settings file**을 열고 `settings.json`에 조직의 Gateway URL을 추가합니다. 기존 JSON 설정은 유지하고 JSON 문법이 유효한지 확인합니다.

```json
{
  "gatewayUrl": "https://<BOB-GATEWAY-HOST>"
}
```

설정을 저장하고 IDE를 완전히 종료한 뒤 다시 실행합니다.

## 5. 로그인 및 기능 확인

1. 조직 IDP에서 발급된 테스트 계정으로 로그인합니다.
2. 채팅에서 민감하지 않은 짧은 프롬프트를 보냅니다.
3. 응답이 반환되는지 확인합니다.
4. 문제가 있으면 시간을 기록하고 운영자에게 inference 및 Gateway 로그의 해당 구간을 확인하도록 요청합니다. 로그를 공유할 때 토큰, 이메일, URL의 사내 호스트명, 사용자 코드/프롬프트는 마스킹합니다.

정상 로그인은 IDE와 IDP 연결만 확인합니다. 모델 추론 성공은 별도로 검증해야 합니다.
