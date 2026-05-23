# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**SmartScheduler** — 단일 `index.html` 파일로 동작하는 AI 기반 일정관리 웹 애플리케이션.

- 빌드 도구, 프레임워크, 외부 라이브러리 없음 (Google Fonts CDN 제외)
- 브라우저에서 `index.html`을 직접 열면 즉시 실행
- 데이터는 `localStorage`에 JSON으로 영속 저장

## Running the App

```bash
# 개발 서버 (선택 사항 — 파일을 직접 열어도 됨)
python -m http.server 8080
# 또는
npx serve .

# 브라우저에서 직접 열기
start index.html          # Windows
open index.html           # macOS
```

테스트 도구, 빌드 스크립트, 패키지 매니저 없음. `index.html` 하나가 전체 앱.

## Architecture

단일 파일 `index.html` 안에 세 레이어가 순서대로 위치:

```
index.html
├── <head>  — 메타태그, Google Fonts CDN, <style> 전체 CSS
├── <body>  — HTML 구조 (헤더 / AI 배너 / 2단 레이아웃)
│   ├── #app-header        — 앱 제목, AI 요약 버튼, 설정 버튼
│   ├── #ai-summary-banner — AI 오늘 일정 요약 (조건부 노출)
│   ├── #left-panel        — 입력 폼, 탭 네비게이션, 필터 바, 일정 목록
│   ├── #right-panel       — 월간 캘린더
│   ├── #settings-modal    — API 키 설정 모달
│   └── #confirm-modal     — 삭제 확인 모달
└── <script> — Vanilla JS 전체 로직
    ├── 상태 관리  (state 전역 객체)
    ├── localStorage 영속성  (STORAGE_KEY = 'smartscheduler_data')
    ├── CRUD 핸들러  (handleAddSchedule / handleEditSchedule / handleDeleteSchedule / handleToggleComplete)
    ├── 필터링 로직  (getFilteredSchedules)
    ├── 렌더링 함수  (render → renderScheduleList + renderCalendar + renderTabBadges)
    └── AI 모듈  (폴백 체인: OpenRouter → OpenAI → 로컬 규칙)
```

## Data Schema

`localStorage` 키: `smartscheduler_data`

```json
{
  "schedules": [
    {
      "id": "uuid-v4",
      "title": "string (필수, 최대 100자)",
      "date": "YYYY-MM-DD (필수)",
      "time": "HH:MM (선택)",
      "category": "work | personal | other",
      "priority": "high | medium | low",
      "memo": "string (최대 500자)",
      "completed": false,
      "aiRecommendation": {
        "suggestedPriority": "high | medium | low | null",
        "reason": "string | null",
        "source": "openrouter | openai | local | null",
        "generatedAt": "ISO 8601 | null"
      },
      "createdAt": "ISO 8601",
      "updatedAt": "ISO 8601"
    }
  ],
  "settings": {
    "openrouterApiKey": "string",
    "openaiApiKey": "string"
  }
}
```

## AI Integration

세 단계 폴백 체인 — 앞 단계가 실패하면 자동으로 다음 단계로 전환:

| 순위 | 서비스 | 모델 | 조건 |
|------|--------|------|------|
| 1 | OpenRouter | `deepseek/deepseek-v4-flash:free` | `settings.openrouterApiKey` 존재 |
| 2 | OpenAI | `gpt-4o-mini` | `settings.openaiApiKey` 존재 |
| 3 | 로컬 규칙 기반 | — | 항상 사용 가능 |

API 키는 설정 모달에서 입력 → `localStorage`에 저장. `.env` 파일은 브라우저에서 직접 읽을 수 없으므로 `config.js`를 통한 주입 또는 UI 입력 방식 사용.

AI 기능:
- **우선순위 추천**: 일정 카드의 🤖 버튼 → JSON `{suggestedPriority, reason}` 응답 파싱
- **오늘 일정 요약**: 헤더의 "AI 요약" 버튼 → `#ai-summary-banner`에 표시

## Design System

CSS 변수 (`--primary`, `--success`, `--warning`, `--danger` 등) 를 수정하면 전체 테마 변경:

```css
--primary: #4f46e5;   /* 인디고 — 주 색상 */
--success: #10b981;   /* 에메랄드 — 완료 */
--warning: #f59e0b;   /* 앰버 — 중간 우선순위 */
--danger:  #ef4444;   /* 레드 — 높은 우선순위 */
```

우선순위별 카드 왼쪽 보더: `priority-high`(red) / `priority-medium`(amber) / `priority-low`(green).
캘린더 점 색상: `dot-work`(indigo) / `dot-personal`(green) / `dot-other`(amber).

반응형 분기:
- `≥ 1024px` — 좌(일정) + 우(캘린더) 2단 그리드
- `768px ~ 1023px` — 1단, 캘린더 상단
- `≤ 767px` — 1단, 캘린더 하단

## Sub-Agents

`.claude/agents/`에 역할별 서브 에이전트가 정의되어 있음:

| 파일 | 역할 | 색상 |
|------|------|------|
| `product-planning-manager.md` | PRD 작성, 요구사항 정의 | Red |
| `backend-developer.md` | localStorage CRUD, 데이터 스키마 | Blue |
| `frontend-developer.md` | UI/UX, 반응형 CSS | Green |
| `qa-quality-engineer.md` | 기능 테스트, 버그 수정 | Yellow |
| `ai-integration-specialist.md` | OpenRouter/OpenAI 연동, 폴백 | Purple |

## Key Constraints

- **단일 파일**: 모든 HTML/CSS/JS는 `index.html` 하나에 인라인. 외부 `.js`/`.css` 파일 추가 금지.
- **XSS 방지**: 사용자 입력은 반드시 `escapeHtml()` 처리 후 `innerHTML` 삽입.
- **날짜 처리**: 날짜 비교는 `YYYY-MM-DD` 문자열 비교 사용 (타임존 문제 방지).
- **렌더링**: 상태 변경마다 `render()` 호출로 전체 재렌더링 (Virtual DOM 없음).
