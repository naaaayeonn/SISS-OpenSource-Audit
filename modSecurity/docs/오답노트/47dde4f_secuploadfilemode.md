# [ModSecurity] 커밋 분석 - 47dde4f
**주제:** SecUploadFileMode의 진법 해석 오류로 인한 파일 권한 문제

---

## 🧩 개요
- **커밋 해시:** 47dde4f
- **관련 후속 커밋**: 5159b773
- **파일 변경:** headers/modsecurity/rules_set_properties.h, src/request_body_processor/multipart.cc, test/test-cases/regression/config-upload_file_mode.json
- **관련 Issue:** https://github.com/owasp-modsecurity/ModSecurity/issues/3580
- **관련 PR/릴리즈:** https://github.com/owasp-modsecurity/ModSecurity/pull/3688

---

## 🔍 변경 요약
- 커밋 9a8ce8b(3.0.15)의 strtol(..., 8)을 10진수 기반 범용 ConfigUnsignedInt::parse()로 대체 --> SecUploadFileMode 0600을 10진수 정수 600으로 잘못 파싱하여, 실제로는 8진수 01130에 해당하는 권한이 적용됨. 그 결과 임시 파일의 읽기 권한이 사라지는 회귀 오류가 발생함.
"0600" (설정 문자열) --> 10진수로 잘못 파싱 --> 600 (10진수 정수) --> fchmod()에 전달 --> 01130 (8진수 권한 표현)
- m_uploadFileMode에 8진수 숫자만 허용
- 부모 클래스의 범위 검사를 재사용하는 8진수 파서(ConfigOctalInt) 부여 --> 기존 범용 정수 파서의 범위 검사 기능은 유지 + SecUploadFileMode에 필요한 8진수 해석을 별도로 구현하기 위함
- 10진수 값 앞에 "o" 접두사만 붙여 출력해서 실제 권한 오류를 로그만으로 알아보기 어렵게 했던 멀티파트 디버그 로그(정수 600을 o600으로 출력했음)에서도 권한을 실제 8진수로 출력하도록 수정
- 회귀 테스트 추가: 정상적인 8진수 설정값 0640을 허용하고 예상 로그를 출력하는지 확인 / 잘못된 8진수 문자열 0689를 거부하는지 확인

### 후속 커밋(5159b773) 변경 요약
- 자릿수 오버플로우 방지 로직이 기존 파서에서는 허용되던 000000000640과 같은 값을 거부함 --> 유효 숫자(significant digits)의 자릿수만 계산하도록 수정
- 회귀 테스트 추가: 000000000640처럼 선행 0이 많은 입력도 허용하는지 확인

---

## 🔐 보안 관점 분석
| 항목 | 이전 버전 | 변경 버전 | 보안적 의미 |
|------|------------|------------|--------------|
| 설정값 파싱 | 10진수 파싱 | 8진수 파싱 | 의도한 파일 권한값을 전달하도록 복구 |
| 실제 파일 권한 | 0600을 10진수로 잘못 해석해서 권한 잘못 부여함(읽기 권한 상실) | 8진수로 해석한 권한값을 fchmod()에 전달하도록 수정 | 읽기 권한 상실 및 의도하지 않은 권한 부여 문제 |
| 업로드 파일 검사 | 권한 설정 오류로 @inspectFile이 파일을 읽지 못하는 경우가 생김 | 의도한 파일 권한값이 설정되도록 파싱 로직 수정. 실제 검사 프로그램 동작은 미검증 | @inspectFile 등 파일 소비자의 정상 동작에 미치는 영향 |

---

## 🧾 검증 및 확인
- [x] 관련 PR/릴리즈 로그 확인: PR #3688은 v3/master 브랜치에 병합됨. git tag --contains 47dde4f 실행 결과, 해당 커밋을 포함하는 태그가 확인되지 않음. 공식 Releases 페이지에서도 해당 패치가 반영된 릴리즈를 확인하지 못함. 따라서 현재 개발 브랜치에는 수정이 반영되었으나 공식 릴리즈 배포는 확인되지 않음.
- [x] 보안 패치(CVE) 여부 확인: Issue #3580, PR #3688 및 GitHub Security Advisories를 조사했으나 해당 수정과 직접 연결된 CVE는 확인되지 않음.
- [x] C 최소 재현 및 실제 파일 권한 확인

### 직접 재현 결과
- strtol("0600", ..., 8) → 384 확인
- strtol("0600", ..., 10) → 600 확인
- fchmod()로 권한 적용 후 fstat() 조회
- 정상 파싱: 0600
- 잘못된 파싱: 01130
- 함수의 반환값 검사 및 실행 결과 확인
재현 범위: 별도 C 프로그램을 이용한 파싱/파일 권한 오류의 최소 재현. ModSecurity 자체의 빌드 및 통합 테스트는 미실시.

---

## 💬 결론
- [ ] 업데이트 성격: 단순 리팩토링이나 기능 추가가 아니라, 리팩토링 과정에서 발생한 파일 권한 설정 회귀 오류를 수정한 보안 관련 버그 픽스.
- [ ] CI/CD 신뢰성 영향: 핵심 수정은 CI/CD 자체를 변경하지 않지만, PR에서 회귀 테스트를 테스트 스위트에 등록한 부분은 향후 동일 오류를 탐지하는 데 기여함.
- [ ] 추가 확인 필요사항: 실제 ModSecurity 빌드 및 통합 테스트, 업로드 임시 파일의 실제 권한 확인, 검사 실패 시 요청 허용/차단 동작 검증.
- 분류: 파일 권한 설정 오류를 해결하는 보안 관련 버그 수정. 공식 CVE 패치로 확인되지는 않음.
- fchmod() 호출에 성공하더라도 전달된 권한값이 잘못되면 의도하지 않은 파일 권한이 적용될 수 있음. 따라서 함수 성공과 그로 인한 결과는 분리해서 분석해야 한다.

---

## 📚 참고
- 관련 PR: https://github.com/owasp-modsecurity/ModSecurity/pull/3688
- 관련 Issue: https://github.com/owasp-modsecurity/ModSecurity/issues/3580
- 핵심 수정 커밋: https://github.com/owasp-modsecurity/ModSecurity/commit/47dde4f
- 후속 수정 커밋: https://github.com/owasp-modsecurity/ModSecurity/commit/5159b773