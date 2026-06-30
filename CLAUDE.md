# CLAUDE.md — 머니맵 (MoneyMap) 개발 가이드

> 다음 세션에서 이 프로젝트를 이어서 작업할 때 먼저 읽는 문서.

## 한 줄 요약
**머니맵(MoneyMap)** — 엑셀 매크로로 쓰던 가족 자산관리 시스템을 **단일 HTML 웹앱**으로 옮긴 것. 설치·서버 불필요, 브라우저에서 바로 실행. 현재 **v1.10.1**.

## 핵심 사실
- **메인 파일**: `index.html` — HTML/CSS/JS 전부 인라인. (이 파일 하나가 앱 전체)
- **라이브러리**: `xlsx.full.min.js` (SheetJS, 엑셀 입출력용, 같은 폴더에 두고 커밋됨). `index.html`과 항상 함께 둘 것.
- **저장**: 브라우저 `localStorage` 키 `assetSystem_v1` (미로그인) + Firebase Firestore (Google 로그인 시).
- **GitHub 레포**: `prjessy/finance` (PUBLIC). main 브랜치에 push하면 GitHub Pages 자동 배포.
- **라이브 URL**: https://prjessy.github.io/finance/  (1~2분 후 반영)
- **Firebase 프로젝트**: `finance-49c1b` (config는 index.html 하단 module script에 포함 — 공개돼도 안전, 보안은 규칙+인증).
- **계정**: GitHub `prjessy`, gh CLI 로그인됨.

## 빌드/배포 워크플로 (매 수정 시)
1. `index.html` 편집
2. **문법 검사** (PowerShell):
   ```
   $html = Get-Content index.html -Raw -Encoding UTF8
   $start = $html.IndexOf("const STORE_KEY"); $end = $html.IndexOf("</script>", $start)
   $html.Substring($start,$end-$start) | Set-Content check.js -Encoding UTF8
   node --check check.js
   ```
3. `VERSION` + `RELEASE_NOTES` 갱신 (index.html 상단)
4. 커밋(메시지 끝에 `Co-Authored-By: Claude ...`) → `git push`
   - PowerShell 커밋 메시지에 `"큰따옴표"` 넣지 말 것(파싱 깨짐). here-string `@'...'@` 사용, 닫는 `'@`는 0열에.
5. 로컬 확인: `Start-Process index.html`

## 코드 구조 (index.html 내부, 위→아래 순서)
- `<head>`: Google Fonts(Noto Sans KR/본고딕), xlsx 스크립트, `<style>`(CSS 변수 `--sky` 등, 다크모드 `body.dark`, 모바일 `@media(max-width:600px)`, 인쇄 `@media print`)
- `<header>`: 제목 + 버전배지(`#verbadge`) + 툴버튼들(통합검색·가져오기·백업·샘플·인쇄·🌙·초기화·`#authbox`)
- `<nav id="tabs">`: 2단 그룹 메뉴 (그룹행 + 서브탭행)
- `<script>` (메인, 비모듈 — onclick 핸들러가 전역 함수 호출):
  - `STORE_KEY`, `VERSION`, `RELEASE_NOTES`, `showReleaseNotes()`
  - `TABS`(탭 정의), `GROUPS`(모바일 그룹), `OPT`(드롭다운), `CATS`(레코드 카테고리), `INCOME_CATS`/`EXPENSE_CATS`, `TAXRATE`
  - `DATA`(전역 상태), `save()`/`load()`/`uid()`
  - 유틸: `won/num/eok/pct/rate/addMonths/daysUntil/esc/maturityCalc`
  - `renderTabs()`/`curGroup()`/`goGroup()`/`go()`/`render()`(탭→렌더함수 dispatch) + 포커스복원
  - 검색/정렬 공통: `tstate/onSearch/sortBy/searchBar/sTh/processRows`
  - 탭 렌더 함수: `rSummary`(대시보드), `rCashTable`, `rInput`(동산 입력 두 패널), `rCashflow`, `rDongsan`, `rBudongsan`, `rLoans`, `rInvest`, `rIPO`(공모주), `rPension`, `rMaturity`, `rGoals`(목표·예산·세액공제·저축목표·자산배분·일일사용액), `rReport`, `rCalendar`, `rPivot`
  - 차트(SVG, 의존성 없음): `svgDonut/svgLine/svgBars/monthlyCash`
  - 기능 함수: 스냅샷`takeSnapshot`, 반복거래`generateRecurring/manageRecurring`, 저축목표`saveGoalForm`, 세액공제`taxDeductionHTML`, 자동분류`autoCat/manageRules`, 구독`subscriptions`, 캘린더/알림`collectEvents/checkReminders/toggleReminders`, 통합검색`openSearch/runGlobalSearch`, 엑셀`importExcel/exportExcel/downloadCategory/downloadTemplate/importCategoryFile`, 모달`openModal/closeModal/genForm/saveGen/delRow`, 샘플`loadDemo`, 초기화`resetAll`, 인쇄`printReport`, 다크`toggleDark`
  - 시작부: `load(); generateRecurring(); (자동스냅샷); renderTabs(); render(); renderAuth(); checkReminders();`
- `<script type="module">` (하단): Firebase 연동(Google 로그인 + Firestore 실시간 동기화). `window.fb`/`window.onAuthChange`/`window.onCloudData`/`window.getLocalData` 브리지로 메인 스크립트와 통신.

## 데이터 모델 (`DATA`)
레코드 배열: `동산, 부동산, 투자자산, 공모주, 연금보험, 대출, 현금흐름, 용돈, 저축목표, 스냅샷`
객체: `반복`(배열), `설정`(순자산목표/예산/배분/공제율구간/분류규칙/용돈지급/lastGen 등)
- Firestore 저장 형태: `users/{uid}` 문서의 `data` 필드에 `DATA` 통째로(JSON). 단일 문서 = 동시편집 시 마지막 저장 우선(가족 동시편집 충돌 주의).

## 흔한 작업 레시피
**새 탭 추가**: `TABS`에 항목 → `GROUPS` 해당 그룹 tabs에 id → `render()` dispatch에 `id:r함수` → `r함수(c)` 작성.
**새 카테고리/필드 추가**: `FORMDEF`(폼) → `CATS`(검색/피벗) → `SHEET_MAP`(엑셀 시트명 매핑) → `DATA` 기본값 **3곳**(`let DATA`, `onCloudData`, `resetAll`) → `exportExcel`의 `sheets`에 추가 → 목록 렌더 함수.
**현금흐름 분류 추가**: `OPT.현금분류`에 추가(수입은 `INCOME_CATS`에도).

## Firebase 보안 규칙 (콘솔에 게시되어 있어야 함)
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```
- 콘솔: Authentication에서 Google 제공자 사용, 승인된 도메인에 `prjessy.github.io` 포함.
- 프라이버시 모델: 미로그인=로컬 전용(아무도 못 봄), 로그인=클라우드(본인 + 프로젝트 주인만). 사용자마다 데이터 격리(서로 못 봄).

## .gitignore (개인정보 보호 — 공개 레포)
`*.jpg/png/xlsx/xls/csv`, `firebase.txt` 제외. **실제 금융 데이터·스크린샷은 커밋 금지.**

## 향후 로드맵
`IDEAS.md` 참고. 남은 후보: 모바일 헤더 ⋯메뉴, 코인 시세 자동갱신(업비트 공개 API, 무료), PWA, 고정/변동 지출 분석, 가족 공유 가계부(구조 변경), AI 코치(서버 필요).
