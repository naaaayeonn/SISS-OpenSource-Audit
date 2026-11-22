# [juice-shop] 커밋 분석 - [f232022]
**주제:** 챌린지 비활성(safe mode) 환경에서 사용자명 렌더링 시 발생하는 SSTI 차단

---

## 🧩 개요
- **커밋 해시:** f232022 (f232022779bf384cad1c9a62aece2216941a6504)
- **파일 변경:** routes/userProfile.ts (+19/-16), test/api/user-profile.test.ts (+50)
- **관련 PR/릴리즈:** 연결된 PR 없음 (GitHub UI 및 API 확인, 2026-11-22) / 이 커밋을 포함하는 가장 오래된 태그: v20.2.0 (이후 v20.3.0, v20.4.0에도 포함, 원본 저장소 태그 기준 2026-11-22 확인)

---

## 🔍 변경 요약
- safe mode(`usernameXssChallenge` 비활성)에서 `'\\' + username`을 pug 템플릿 소스에 삽입하던 방식을 고정 문자열 `#{username}` 삽입으로 변경
- 챌린지 활성 모드의 `eval` 동작은 그대로 유지 (의도된 취약점)
- `themeKey`/`theme` 계산을 위로 이동 (동작 변화 없음)
- 테스트 2개 추가: 일반 모드 회귀 테스트, safe mode 차단 테스트

---

## 🔐 보안 관점 분석
| 항목 | 이전 버전 | 변경 버전 | 보안적 의미 |
|------|------------|------------|--------------|
| 사용자 입력 위치 | 템플릿 소스에 직접 삽입 | 템플릿 변수(`fn(user)`)로 전달 | 코드와 데이터 분리 |
| 방어 방식 | `\` 접두 이스케이프 | 고정 표현식 `#{username}` | 입력 형태에 의존하지 않음 |
| 우회 payload | `A#{7*7}` → `A49` (평가됨) | 출력: `A#{7*7}` (글자 그대로) | SSTI 차단 |
| 챌린지 모드 | eval 유지 | eval 유지 | 의도된 취약점은 보존 |

**취약 원인:** `\`는 바로 뒤의 `#{`만 무력화하므로, 앞에 문자가 있으면 pug 보간이 평가된다. pug의 `#{}`는 서버 측 JavaScript 표현식을 평가하므로 서버 코드 실행으로 이어질 수 있는 유형이다.

---

## 🧪 재현 (pug 템플릿 수준)
- 환경: Node.js v26.13.1, pug 3.0.4 (Juice Shop package.json: ^3.0.3), 저장소 외부 임시 폴더에서 실행
- 실행 코드:
```js
const pug = require('pug')
console.log(pug.render('p \\A#{7*7}'))
console.log(pug.render('p \\#{7*7}'))
console.log(pug.render('p #{username}', { username: 'A#{7*7}' }))
```
- 결과:

| 입력 | 출력 | 해석 |
|------|------|------|
| `\A#{7*7}` (수정 전 방식) | `<p>\A49</p>` | 보간이 평가됨 → 우회 성공 |
| `\#{7*7}` | `<p>#{7*7}</p>` | `#{`가 맨 앞이면 `\`로 방어됨 |
| `#{username}` + 데이터 `A#{7*7}` (수정 후 방식) | `<p>A#{7*7}</p>` | 입력이 데이터로 처리됨 |

- **재현 범위:** pug 템플릿 수준의 동작만 확인함. 앱 전체(end-to-end) 동작은 커밋에 포함된 테스트(`GET user profile does NOT evaluate SSTI payload in safe mode`)로 확인하며 직접 실행하지는 않음.

---

## 🧾 검증 및 확인
- [x] 관련 PR/릴리즈 로그 확인 (연결된 PR 없음, v20.2.0부터 릴리즈에 포함)
- [x] 보안 패치(CVE) 여부 확인 (juice-shop 저장소에 공개된 Security Advisory 없음. CVE/GHSA 확인하지 못함)
- [x] 테스트/빌드 정상 동작 확인 (pug 템플릿 수준 직접 재현, 커밋 테스트 코드 분석)

---

## 💬 결론
- [x] 업데이트 성격: 보안 강화 (safe mode 한정, 챌린지 모드는 의도된 취약점으로 유지)
- [x] CI/CD 신뢰성 영향: 없음 (테스트 추가)
- [ ] 추가 확인 필요사항: 앱 전체 end-to-end 재현 여부, 공식 심각도 평가는 확인되지 않음

---

## 📚 참고
- PR 링크: 없음
- 릴리즈 노트: v20.2.0 (이 커밋을 처음 포함한 태그)
- 관련 문헌/참고자료: https://github.com/juice-shop/juice-shop/commit/f232022779bf384cad1c9a62aece2216941a6504
- Security Advisories: https://github.com/juice-shop/juice-shop/security/advisories (공개 항목 없음, 2026-11-22 확인)
