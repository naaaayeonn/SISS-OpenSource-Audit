# [Cosign] 커밋 분석 - 8ac891ff
주제: 보안 패치 - 서명/증명 개수 제한 추가 (CVE-2023-46737)

---

## 개요
- 커밋 해시: `8ac891ff0e29ddc67965423bee8f826219c6eb0f` (2023-11-07)
- 커밋 메시지: `Merge pull request from GHSA-vfp6-jrw2-99g9`
- 파일 변경: `pkg/cosign/fetch.go` (+9), `test/e2e_test.go` (+114)
- 관련 PR: 없음. 보안 권고 [GHSA-vfp6-jrw2-99g9](https://github.com/sigstore/cosign/security/advisories/GHSA-vfp6-jrw2-99g9)로 수정됨
- 릴리즈: [v2.2.1](https://github.com/sigstore/cosign/releases/tag/v2.2.1)
- 배경: cosign은 이미지를 검증할 때 레지스트리에서 서명/증명 목록을 받아온다. 패치 전에는 개수 제한이 없어서, 공격자의 레지스트리가 서명을 아주 많이 보내면 cosign이 다 처리하느라 멈췄다. (DoS)

---

## 변경 요약
```diff
+const maxAllowedSigsOrAtts = 100

 func FetchSignaturesForReference(...) {
 	if len(l) == 0 {
 		return nil, fmt.Errorf("no signatures associated with %s", ref)
 	}
+	if len(l) > maxAllowedSigsOrAtts {
+		return nil, fmt.Errorf("maximum number of signatures on an image is %d, found %d", maxAllowedSigsOrAtts, len(l))
+	}

 func FetchAttestations(...) {
 	if len(l) == 0 {
 		return nil, errors.New("found no attestations")
 	}
+	if len(l) > maxAllowedSigsOrAtts {
+		errMsg := fmt.Sprintf("maximum number of attestations on an image is %d, found %d", maxAllowedSigsOrAtts, len(l))
+		return nil, errors.New(errMsg)
+	}
```

1. 최대 개수 상수 100 추가
2. 서명 조회 함수에 100개 초과 시 에러 반환 추가
3. 증명 조회 함수에도 같은 검사 추가
4. 이를 확인하는 e2e 테스트 추가

삭제된 코드는 없다.

---

## 보안 관점 분석
| 항목 | 이전 버전 (≤ 2.2.0) | 변경 버전 (2.2.1) | 보안적 의미 |
|------|------|------|------|
| 서명/증명 개수 | 제한 없음 | 100개 초과 시 에러 | 자원 과다 사용 방지 (CWE-400) |
| 공격 시 결과 | cosign이 멈춤 | 에러로 종료 | 가용성 문제만 있음, 데이터 유출·변조 없음 |
| 정상 사용 | 영향 없음 | 100개 넘는 이미지는 검증 실패 | 실제로 100개 넘는 경우는 드묾 |

심각도: NVD 5.3 Medium / GitHub 3.1 Low. 공격 난이도 평가가 달라서 점수가 다르고, 둘 다 DoS라는 점은 같다.

---

## 검증 및 확인
- [x] 관련 PR/릴리즈 로그 확인 (PR 번호 없음, 권고문과 v2.2.1 릴리즈 노트에서 확인)
- [x] 보안 패치(CVE) 여부 확인 (CVE-2023-46737, 영향 버전 ≤ 2.2.0, 패치 버전 2.2.1)
- [ ] 테스트/빌드 정상 동작 확인 (직접 실행 안 함, 커밋에 포함된 테스트로 갈음)

---

## 결론
- 업데이트 성격: [x] 보안 강화 / [ ] 기능 개선 / [ ] 단순 리팩토링
- CI/CD 신뢰성 영향: [x] 있음 / [ ] 없음
- 추가 확인 필요사항: 서명이 100개 넘게 붙는 이미지는 2.2.1부터 검증 실패

외부에서 받은 데이터의 개수를 제한하지 않은 것이 취약점이 된 사례다.

---

## 참고
- 권고문: https://github.com/sigstore/cosign/security/advisories/GHSA-vfp6-jrw2-99g9
- 커밋: https://github.com/sigstore/cosign/commit/8ac891ff0e29ddc67965423bee8f826219c6eb0f
- 릴리즈 노트: https://github.com/sigstore/cosign/releases/tag/v2.2.1
- NVD: https://nvd.nist.gov/vuln/detail/CVE-2023-46737