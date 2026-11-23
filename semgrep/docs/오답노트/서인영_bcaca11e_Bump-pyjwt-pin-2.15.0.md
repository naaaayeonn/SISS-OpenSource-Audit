# Semgrep 커밋 분석 - bcaca11e
**주제:** PyJWT 의존성 핀 상향(`~=2.13.0` → `>=2.15.0,<3`)으로 알려진 취약점(CVE-2026-102267 등) 해소

---

## 🧩 개요
- **커밋 해시:** `bcaca11e919e6d4dc94992865f77adcfb1216131` (2026-10-01, Tean Lai)
- **커밋 메시지:** `fix(deps): bump pyjwt pin from ~=2.13.0 to >=2.15.0,<3 (semgrep/semgrep-proprietary#7164)`
- **파일 변경:** `cli/pyproject.toml`, `cli/uv.lock`, `changelog.d/gh-11953.fixed` (3개 파일)
- **관련 Issue:** [semgrep/semgrep#11925](https://github.com/semgrep/semgrep/issues/11925), [semgrep/semgrep#11953](https://github.com/semgrep/semgrep/issues/11953)
- **관련 PR:** `semgrep/semgrep-proprietary#7164` → 비공개 저장소라 열람 불가 (404 확인)
- **반영 릴리즈:** **v1.179.0** (CHANGELOG 기준 2026-10-01)
- **관련 CVE:** CVE-2026-102267 (커밋 메시지 명시), 그 외 같은 핀에 걸려 있던 PyJWT 취약점 다수

---

## 🔍 변경 요약

```diff
# cli/pyproject.toml
-  "pyjwt[crypto]~=2.13.0",
+  "pyjwt[crypto]>=2.15.0,<3",
```

- `cli/uv.lock`: pyjwt 2.13.0 → 2.15.0 (uv 0.10.9로 `uv lock --upgrade-package pyjwt`), 그 외에는 `exclude-newer` 타임스탬프만 변경 (커밋 메시지 기준)
- `changelog.d/gh-11953.fixed`: 변경 이력 추가 → 1.179.0 CHANGELOG에 `(gh-11953)`으로 반영됨
- `~=2.13.0`은 `>=2.13.0, <2.14.0`과 같아서 **2.14.0 이상(수정 버전)은 설치 자체가 불가능**했음. 새 제약은 수정 버전을 하한으로 삼음.
- 2.15.1(2026-09-28 릴리즈)은 락 파일의 1주 쿨다운(`exclude-newer-span = "P1W"`) 안이라 락은 2.15.0으로 해석됨 (커밋 메시지).
- `mcp==1.29.0`이 `pyjwt>=2.10.1`을 허용하므로 다른 의존성과 충돌 없음 (커밋 메시지).

**수정 배경 (커밋 메시지 + 공개 Issue)**
- Semgrep 자체 CI(`semgrep-ci / job`)가 2026-09-30부터 `develop`에서 `pyjwt - CVE-2026-102267` reachable 차단 항목 2건으로 실패 중이었음. 도달 경로는 `cli/src/semgrep/mcp/utilities/token_verifier.py` (`jwt.PyJWKClient` 사용 코드로 직접 확인).
- **#11925** (jkugler, 2026-09-24): "Semgrep dependencies are too tight". semgrep ≥1.177.0이 `pyjwt>=2.13.0,<2.14.dev0`을 요구해 PyJWT 2.14가 필요한 프로젝트에 설치할 수 없다는 **호환성 문제** 제기 (우선순위 P1). 10월 6일 작성자가 "v1.179.0에서 수정됨"이라고 코멘트하고 completed로 닫음.
- **#11953** (bjpaolucci, 2026-10-01): "pyjwt pinned to ~=2.13.0 blocks fixes for CVE-2026-101917, CVE-2026-101918 and others". pip-audit 기준 PyJWT 2.13.0의 알려진 취약점 13건이 모두 2.14.0 또는 2.15.0에서 수정된다는 **보안 문제** 제기 (우선순위 P1). `pyjwt>=2.15.1`을 따로 지정하면 pip가 semgrep 1.156.0까지 되돌아가고, 그 버전이 끌어오는 mcp 1.23.3, click 8.1.8에도 취약점이 있어 어느 쪽을 택해도 감사가 실패한다고 설명. 제안은 `pyjwt[crypto]>=2.15.1,<3`. 상태는 Closed(completed).
- #11925 댓글(dbrunet73, 10/1): OSV 기준 2.13.0에 11건(CRITICAL 1, HIGH 5, MODERATE 5)이 있다고 보안 관점을 보탬. 임시 해결로 uv `override-dependencies`를, 정식 수정으로 `~=2.14`를 제안.
- 두 Issue는 같은 핀이 원인이지만 제기 관점이 다름 (설치 불가 vs 취약점 미해소). 이슈 타임라인에는 외부 프로젝트(데이터 파이프라인, 린트 도구, NixOS 패키징 등)가 같은 핀 때문에 우회 PR을 올린 흔적이 다수 있음.

---

## 🔐 보안 관점 분석

| 항목 | 이전 버전 | 변경 버전 | 보안적 의미 |
|------|------------|------------|--------------|
| `pyproject.toml` 제약 | `pyjwt[crypto]~=2.13.0` (2.13.x만 허용) | `pyjwt[crypto]>=2.15.0,<3` | 수정 버전을 하한으로 지정해 취약 버전 설치 경로를 제약 단계에서 차단. 메이저 상한(`<3`)은 유지 |
| `uv.lock` (Semgrep 개발용 락) | pyjwt 2.13.0 | pyjwt 2.15.0 | 해시 고정된 락 파일 자체도 수정 버전으로 이동 |
| 사용자의 `pip install` 결과 | semgrep 1.178.0 → PyJWT **2.13.0** | semgrep 1.179.0 → PyJWT **2.15.1** | 사용자는 락이 아니라 제약 범위 안의 최신 버전을 받음 (직접 확인) |
| `pip-audit` (pyjwt) | **15건** | **0건** | 수정 전후 감사 결과 차이 (2026-11-23 실행, 감사 DB 기준 시점에 따라 변동) |
| CVE-2026-101918 재현 | `RecursionError` | `DecodeError` | 처리되지 않는 예외가 PyJWT 정의 예외로 바뀜 |
| 버전 범위 | 좁음 (패치 못 받음) | 넓음 (2.x 후속 업데이트 자동 수용) | 수정은 쉬워졌지만, 호환성과 공급망 노출 면은 넓어지는 트레이드오프 |

**관련 CVE (직접 확인한 출처 기준)**

| CVE | 내용 | 영향 버전 | 수정 버전 | 심각도 |
|---|---|---|---|---|
| CVE-2026-102267 | `PyJWKClient`가 리디렉션 대상을 JWKS 신뢰 경계에 대해 재검증하지 않음 | 2.14.0 미만 | 2.14.0 | CVSS 3.1 7.4 High (Ubuntu) |
| CVE-2026-101918 | `PyJWKClient.get_signing_key_from_jwt`와 `verify_signature=False` 경로에서 깊게 중첩된 페이로드가 `RecursionError`를 일으켜 인증 없이 핸들러가 중단될 수 있음 (CWE-248) | 2.0.0a1 이상 2.15.0 미만 | **2.15.0** (2.14.0은 여전히 영향) | CVSS 3.1 5.3 Medium |
| CVE-2026-102268 | PEM 형식 판별 우회로 공개키가 HMAC 키로 쓰이는 알고리즘 혼동 (CWE-347) | 2.14.0 미만 | 2.14.0 | CVSS 3.1 9.1 Critical (GitLab Advisory) |

- 감사 도구(pip-audit)로 2.13.0의 15건을 조회한 결과, 12건은 2.14.0에서, 2건(CVE-2026-101918, CVE-2026-102275)은 **2.15.0에서** 수정됨. 이것이 하한을 `>=2.14`가 아니라 `>=2.15.0`으로 잡은 이유로 해석됨. CVE-2026-103001은 감사 DB에 수정 버전이 표시되지 않음.
- 커밋 메시지가 명시한 것은 CVE-2026-102267 하나이고, Issue는 13건(#11953)과 11건(#11925 댓글, OSV 9/30 기준)을 각각 제기함. 개수 차이는 조회 시점과 도구(pip-audit vs OSV)가 다르기 때문으로 보이며 이는 추정임.
- **(분석자 추정)** `token_verifier.py`가 `PyJWKClient`로 JWKS 키를 가져오고 `algorithms=[key.algorithm_name]`처럼 단일 알고리즘만 허용하므로, 102268(알고리즘 혼용 허용 조건)과 빈 HMAC 키(102266)는 조건이 맞기 어렵고, `PyJWKClient`를 쓰는 102267, 101918이 이 코드와 직접 닿는 것으로 이해됨. 실제 악용 가능성은 별도 검증이 필요함.

---

## 🧾 검증 및 확인
- [x] 관련 PR/릴리즈 로그 확인
  - 로컬 클론에서 `git merge-base --is-ancestor bcaca11e 4802833a5` → 포함됨 (`4802833a5` = Release Version 1.179.0, OSS 미러 커밋 날짜 2026-10-07)
  - `CHANGELOG.md` 257번째 줄이 `## [1.179.0] - 2026-10-01` 섹션 아래에 있음 (194~261줄). 문구는 "Semgrep now allows PyJWT 2.15 and later ... (gh-11953)"이며 CVE 번호는 적혀 있지 않음
  - 공개 Issue #11925에서 작성자가 "This was fixed by v1.179.0"이라고 확인
  - PR #7164는 비공개 저장소(404)라 열람 불가
- [x] 보안 패치(CVE) 여부 확인
  - Ubuntu Security, Debian Security Tracker, GitLab Advisory Database에서 위 CVE 3건 확인 (NVD 페이지는 이 분석 환경에서 내용을 읽지 못해 대조하지 못함)
- [x] 테스트/빌드 정상 동작 확인 (직접 재현)
  - 환경: macOS, Python 3.11, pip-audit 2.10.1, 2026-11-23 실행
  - 수정 전 `semgrep==1.178.0` → PyJWT 2.13.0, `pip-audit` pyjwt 15건
  - 수정 후 `semgrep==1.179.0` → PyJWT 2.15.1, `pip-audit` pyjwt 취약점 없음
  - CVE-2026-101918 유형 재현: 깊이 5000으로 중첩된 JSON 페이로드로 `jwt.decode(token, options={"verify_signature": False})` 실행 → 2.13.0 `RecursionError`, 2.15.1 `DecodeError`
  - 한계: `PyJWKClient` 경로는 직접 실행하지 않음, 2.15.0 단독 실행은 하지 않음, Semgrep 자체 CI 빌드 로그는 확인하지 않음

---

## 💬 결론
- [x] 업데이트 성격: **보안 강화** (의존성 핀 상향, 알려진 취약점 해소)
- [x] CI/CD 신뢰성 영향: **있음** (Semgrep 자체 공급망 스캔 CI의 차단 항목 해소, 그리고 이 핀 때문에 의존성 감사가 실패하던 다운스트림 사용자의 CI에도 영향)
- [x] 추가 확인 필요사항:
  - CVE-2026-103001의 수정 버전 (감사 DB에 미표시)
  - `PyJWKClient` 경로에서의 실제 재현 (이번에는 `jwt.decode` 경로로 대체)
  - 연결된 다른 커밋 `a8fc986`(semgrep-ci, 10/2)의 정체
  - Issue #11953의 정확한 종료일

**시사점**
1. 보안 패치가 나와도 `~=` 같은 좁은 핀이 **패치 설치를 막을 수 있음**. 락 파일뿐 아니라 제약 표기를 함께 점검해야 함.
2. 보안 도구도 자기 의존성에서 자유롭지 않음. Semgrep 자체 공급망 스캔이 자신의 PyJWT 핀을 잡아냄.
3. 도달 가능성(reachability) 분석이 우선순위를 정함. CI는 `PyJWKClient`를 쓰는 CVE-2026-102267을 차단 항목으로 판정.
4. 한 프로젝트의 과도한 핀은 다운스트림에 파급됨. 이슈 타임라인에서 같은 핀 때문에 우회 PR을 올린 외부 프로젝트가 여러 곳 확인됨.
5. 같은 시기에 urllib3(CVE-2026-97689), anyio 취약점 대응 커밋(`57cfaa2f`, v1.180.0)이 이어짐. 의존성 취약점은 연쇄적으로 나옴.

---

## 📚 참고
- **PR 링크:** `semgrep/semgrep-proprietary#7164` (비공개, 열람 불가)
- **Issue:** [#11925](https://github.com/semgrep/semgrep/issues/11925), [#11953](https://github.com/semgrep/semgrep/issues/11953)
- **커밋:** [bcaca11e](https://github.com/semgrep/semgrep/commit/bcaca11e919e6d4dc94992865f77adcfb1216131)
- **릴리즈 노트:** [Semgrep v1.179.0](https://github.com/semgrep/semgrep/releases/tag/v1.179.0)
- **CVE 출처:**
  - [Ubuntu Security – CVE-2026-102267](https://ubuntu.com/security/CVE-2026-102267)
  - [Debian Security Tracker – CVE-2026-102267](https://security-tracker.debian.org/tracker/CVE-2026-102267)
  - [GitLab Advisory – CVE-2026-101918](https://advisories.gitlab.com/pypi/pyjwt/CVE-2026-101918/)
  - [Ubuntu Security – CVE-2026-101918](https://ubuntu.com/security/CVE-2026-101918)
  - [GitLab Advisory – CVE-2026-102268](https://advisories.gitlab.com/pypi/pyjwt/CVE-2026-102268/)
- **관련 개념:** PEP 440 호환 릴리즈(`~=`), 소프트웨어 공급망 보안, reachability 분석
