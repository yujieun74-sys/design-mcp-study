# Google Design MCP로 컬러 시안 테스트하기

## 0. 쓰게 된 배경
시안 잡을 때 같은 화면을 컬러별로 비교해봐야 하는 경우가 종종 있는데, 매번 팔레트 뽑고 칩 만들고 화면에 적용하는 게 손이 많이 감. 그래서 AI가 컬러 스킴 생성부터 Figma 적용까지 해줄 수 있는지 테스트해봄.

## 1. Google Design MCP란?
구글이 공식으로 낸 디자인용 MCP 서버. AI(Claude 등)가 구글의 디자인 기능을 도구처럼 쓸 수 있게 해줌.

**주요 기능**
- **generate_color_scheme**: 키 컬러(hex) 넣으면 Material 3 컬러 스킴 생성 (primary, secondary, tertiary, surface 등 롤별 hex, 라이트/다크 포함)
- **search_icons**: 키워드로 Material Symbols 아이콘 검색
- **search_fonts / describe_font**: 언어·카테고리로 Google Fonts 검색 및 설명

**특징**
- API 키, 로그인 필요 없음
- 결과가 Material 3 체계 기준이라 롤 이름도 M3 이름(primary, on_primary_container 등)으로 나옴

**Figma MCP랑 같이 쓰면**: Google Design으로 컬러 생성 → Figma에 칩/변수/화면 적용까지 한 번에 가능

참고: [Google Design MCP 공식 문서](https://developers.google.com/design-mcp/overview)

## 2. 연결 방법 및 주의점

### 먼저 용어 정리
- **터미널(PowerShell / Mac 터미널)**: 명령어 치는 창
- **Claude Code**: 터미널 안에서 돌아가는 Claude 프로그램. 터미널에 `claude` 치면 실행됨
- 아래 `claude mcp add ...` 같은 명령어도 Claude Code 명령어라, 터미널에서 치면 Claude Code에 MCP가 등록됨

### 연결 방법

**1) Claude Code 설치** (처음 쓰는 사람만, 이미 있으면 생략)
- 설치 여부 확인: 터미널에서 `claude --version` → 버전 나오면 설치된 것
- Mac: `curl -fsSL https://claude.ai/install.sh | bash`
- Windows(PowerShell): `irm https://claude.ai/install.ps1 | iex` (Git for Windows 필요)

**2) 터미널에서 MCP 서버 추가**
```
claude mcp add --transport http -s user google-design https://design.googleapis.com/mcp
claude mcp add --transport http -s user figma https://mcp.figma.com/mcp
```
`-s user`를 붙이면 어느 폴더에서 실행해도 두 MCP를 쓸 수 있음

**3) 작업 폴더 만들고 Claude Code 실행**
```
mkdir color-test
cd color-test
claude
```

**4) Figma 인증**: Claude Code 입력창에 `/mcp` → figma → Authenticate → 브라우저에서 허용

**5) 확인**: `/mcp`에서 둘 다 connected면 완료

### 주의점
- **claude.ai 웹 커넥터로는 연결 안 됨**: OAuth 등록 에러 발생. Claude Code나 Cursor 사용 권장
- **로그인 계정 확인**: 상단에 `API Usage Billing`이 뜨면 사용량 과금 방식. `/login`에서 구독 계정으로 로그인하면 Pro/Max로 바뀜
- **실행 위치 확인**: `claude` 실행한 폴더에 파일이 생성됨. 상단 경로 확인 필수
- **auto mode**: 허락 없이 진행하는 모드. Figma 수정 전엔 `Shift+Tab`으로 끄는 게 안전
- **공용 라이브러리 주의**: 테스트는 반드시 개인 드래프트 파일에서
- **이름 체계 다름**: M3 롤 이름 ≠ 우리 토큰 이름. 실제 반영 시 매핑 작업 필요

## 3. 실험 내용

**목표**: 파란 계열 후보 2개를 컬러 스킴으로 만들어 비교
- A: Cobalt Blue `#0047AB`
- B: Royal Blue `#2563EB`

### Step 1. 컬러 스킴 생성 프롬프트
```
google-design으로 A타입 Cobalt Blue(#0047AB), B타입 Royal Blue(#2563EB) 각각 컬러 스킴 만들어줘.
1. primary / secondary / tertiary 그룹별로 정리 (기본색, on, container, on-container / 라이트·다크)
2. A/B 나란히 비교 표
3. 생성된 primary가 입력 hex와 얼마나 달라졌는지 표시
4. secondary/tertiary 색 계열 설명
5. 주요 차이점 3줄 요약
6. color-compare.html 파일로 컬러 칩 생성
```

### 결과 (기본 방식 TONAL_SPOT)

| | 입력 → primary | 채도 변화 | 원래 색과 차이(ΔE) |
|---|---|---|---|
| A | #0047AB → #485D92 | -48% | 31.5 |
| B | #2563EB → #4B5C92 | -59% | 47.8 |

- 기본 방식은 채도를 일정 수준으로 낮춰서 둘 다 탁한 슬레이트 블루가 됨
- 원래 두 색 차이(ΔE 22)가 스킴에선 1.4로 줄어 **사실상 같은 팔레트**
- secondary는 채도 뺀 블루그레이, tertiary는 색상각 약 60° 돌린 더스티 모브 계열로 자동 생성

> **인사이트**: 브랜드 컬러 비교가 목적이면 기본 방식은 부적합. 입력 채도를 유지하는 **FIDELITY** 방식으로 뽑아야 후보 간 차이가 살아남

### Step 2. Figma에 칩으로 그리기
```
A/B를 FIDELITY 방식으로 다시 컬러 스킴 만들고,
TONAL_SPOT/FIDELITY 두 버전 모두 이 Figma 파일에 컬러 칩으로 그려줘: [링크]
- 버전별 섹션, A/B 나란히, 라이트/다크 구분
- 그룹별 칩 4개, 칩 아래 롤 이름과 hex
- 변수 컬렉션 이름은 m3-test, 공용 라이브러리는 건드리지 않기
```

### 결과
<!-- Figma 캡처 이미지 첨부 -->

## 다음 단계로 해볼 것
- 실제 화면(온보딩 등) 복제본에 변수 바인딩해서 후보별 비교
- WCAG 대비율 자동 체크
- 우리 토큰 체계와 M3 롤 매핑표 만들기
