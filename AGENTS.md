# Confluence CLI — 프로젝트 가이드

## 아키텍처

```
bin/
  index.js          # 엔트리포인트 (Node 버전 체크 → confluence.js 로딩)
  confluence.js     # Commander CLI 정의 (모든 커맨드 등록)

lib/
  confluence-client.js  # 핵심 클라이언트 (API 호출, 포맷 변환, 페이지 CRUD)
  config.js             # 설정 로드 (~/.confluence-cli/config.json)
  analytics.js          # 익명 사용 통계

tests/
  confluence-client.test.js  # Jest 단위 테스트

.github/workflows/
  ci.yml        # push/PR → lint + test (Node 18.x, 20.x)
  publish.yml   # tag push (v*) → npm publish with OIDC provenance
```

## 코드 스타일

- ESLint flat config (`eslint.config.js`)
- 들여쓰기: 2칸 스페이스
- 따옴표: 작은따옴표
- 세미콜론: 필수
- `no-unused-vars`: `_` 접두사 무시

## 핵심 변환 파이프라인

```
Markdown → MarkdownIt.render() → HTML → htmlToConfluenceStorage() → Storage XML
```

`htmlToConfluenceStorage()`는 MarkdownIt이 생성한 HTML 중 Confluence 전용 처리가 필요한 것만 변환:
- `<li>` → `<li><p>...</p></li>` (Confluence 필수)
- `<pre><code>` → `ac:structured-macro` code block with CDATA
- `<blockquote>` → info/warning/note 매크로
- `<th>`, `<td>` → `<p>` 래핑
- `<hr>` → `<hr />` (self-closing)

나머지 HTML 태그(h1-h6, p, strong, em, ul, ol, table 등)는 이미 Confluence storage와 호환되므로 변환하지 않음.

## 버전 관리

- `package.json`의 version은 직접 올리지 않아도 됨
- publish.yml이 git tag에서 버전을 추출해서 package.json을 자동 맞춤
- 단, 로컬 개발 시 혼동 방지를 위해 릴리스 전 맞춰두는 것을 권장

## 릴리스 절차

```bash
# 1. 변경 커밋 (Conventional Commits)
git add -A
git commit -m "feat: add --parent option to create command"

# 2. 태그 생성 (SemVer)
git tag v1.16.0

# 3. 푸시 (커밋 + 태그)
git push origin main --follow-tags
```

태그 push 시 자동으로:
1. `ci.yml` → lint + test (push to main 트리거)
2. `publish.yml` → npm publish with OIDC provenance (tag 트리거)

### 버전 결정 기준

| 변경 유형 | 예시 | 버전 |
|-----------|------|------|
| 새 명령어/옵션 추가 | `--parent` 옵션 | minor (x.Y.0) |
| 버그 수정, 내부 정리 | no-op regex 제거 | patch (x.y.Z) |
| 하위 호환 깨짐 | 명령어 인터페이스 변경 | major (X.0.0) |

### npm OIDC Trusted Publishing

publish.yml은 `id-token: write` 권한으로 npm OIDC provenance를 사용.
npm 토큰이 아닌 GitHub Actions OIDC로 인증하므로 시크릿 관리 불필요.
Node.js 24.x (npm 11.5.1+) 필요.

## 테스트

```bash
npm test          # Jest 단위 테스트 (40개)
npm run lint      # ESLint
```

테스트는 `axios-mock-adapter`로 HTTP 모킹. 실제 Confluence 인스턴스 불필요.

## agent-rules 연동

이 CLI는 [mona/agent-rules](https://oss.navercorp.com/mona/agent-rules)의 `confluence-cli` 스킬로 사용됨:
- 스킬 문서: `sources/skills/confluence-cli/SKILL.md`
- 설치: `npm install -g @bestend/confluence-cli@latest`

CLI 버전 릴리스 후 agent-rules의 SKILL.md `metadata.version`도 맞춰 업데이트할 것.

## 금지 사항

- `htmlToConfluenceStorage()`에 no-op regex 추가 금지 (X→X 변환은 의미 없음)
- 전역 entity unescape (`&lt;`→`<` 등) 금지 — code block 내부는 이미 별도 디코딩됨
- OIDC publish 설정 변경 시 Node.js 24.x 이상 유지 필수
