# 3D Page Interaction Diary — Portfolio

> 오프라인 다이어리를 쓰는 감각을 온라인에서 그대로 재현한 웹앱  
> **개발 형태:** 1인 풀스택 개발 | **개발 기간:** 진행 중

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Key Features & Technical Specs](#3-key-features--technical-specs)
4. [Trouble Shooting](#4-trouble-shooting)
5. [Project Roadmap](#5-project-roadmap)

---

## 1. Project Overview

### What I Built

오프라인 다이어리를 쓸 때의 감각 — 페이지를 손으로 넘기는 물성, 빈 페이지를 채워가는 행위, 영수증과 티켓을 끼워 넣는 습관 — 을 웹 환경에서 동일하게 재현하는 것을 목표로 설계한 다이어리 웹앱입니다.

기존 온라인 다이어리 서비스들이 "얼마나 효율적으로 기록하느냐"에 집중한다면, 이 프로젝트는 **"오프라인에서 다이어리를 쓸 때 이 행동이 존재하는가?"** 라는 하나의 질문을 설계 원칙으로 삼아, 모든 기능과 UI 결정의 기준으로 적용했습니다.

### Why I Built This

> **"3D는 수단이지 목적이 아닙니다."**

3D 페이지 렌더링과 인터랙션은 오프라인 감각을 재현하기 위해 선택한 기술 수단입니다. 화려한 3D 효과를 보여주기 위한 프로젝트가 아니라, **감각의 이식(Sensory Transplantation)** 이라는 UX 목표를 달성하기 위해 3D가 필요했습니다.

### Core Concept Map

| 오프라인 행동 | 온라인 구현 |
|------------|-----------|
| 페이지를 손으로 넘긴다 | Three.js 3D 페이지 넘김 애니메이션 (종이 휘어짐·그림자) |
| 빈 페이지에 글을 쓴다 | 페이지 위 직접 텍스트 입력 편집 UI |
| 사진을 인화해서 붙인다 | 이미지를 페이지 위 자유 위치에 배치 |
| 영수증·티켓을 끼워 넣는다 | OCR 인식 후 페이지에 자동 삽입 |

### Tech Stack

| 영역 | 기술 |
|------|------|
| Framework | Next.js 14 (App Router) |
| 3D Rendering | Three.js + React Three Fiber |
| State Management | Zustand |
| Database | Supabase (PostgreSQL) |
| File Storage | Supabase Storage |
| OCR | Naver CLOVA OCR |
| Deployment | Vercel |
| CI/CD | GitHub Actions |
| Error Tracking | Sentry |

---

## 2. System Architecture

### 2.1 전체 구조

```
┌─────────────────────────────────────────────────────┐
│                      Client                         │
│                                                     │
│   ┌──────────────┐      ┌────────────────────────┐  │
│   │  3D Renderer │      │     Page Editor        │  │
│   │  (Three.js + │      │  (Text / Image / OCR)  │  │
│   │   R3F)       │      │                        │  │
│   └──────┬───────┘      └──────────┬─────────────┘  │
│          │                         │                 │
│          └──────────┬──────────────┘                 │
│                     │                               │
│              Zustand Store                          │
│           (diary / page state)                      │
└─────────────────────┬───────────────────────────────┘
                      │ HTTP (Next.js API Routes)
┌─────────────────────▼───────────────────────────────┐
│                  Server (Vercel)                     │
│                                                     │
│   ┌────────────┐  ┌──────────────┐  ┌────────────┐  │
│   │ Diary CRUD │  │ Image Upload │  │ OCR API    │  │
│   │ API Routes │  │ + Resize     │  │ (CLOVA)    │  │
│   └─────┬──────┘  └──────┬───────┘  └─────┬──────┘  │
└─────────┼────────────────┼────────────────┼──────────┘
          │                │                │
┌─────────▼────────┐  ┌────▼──────────┐     │
│  Supabase DB     │  │  Supabase     │     │
│  (PostgreSQL)    │  │  Storage      │     │
│                  │  │  (WebP 이미지) │     │
│  - RLS 정책 적용  │  │               │  Naver
│  - URL만 저장    │  └───────────────┘  CLOVA API
└──────────────────┘
```

### 2.2 데이터 흐름 — 이미지 업로드

```
사용자가 이미지 선택
       │
       ▼
클라이언트: 파일 크기·형식 유효성 검증
       │
       ▼
API Route: sharp로 리사이징 + WebP 변환
       │
       ▼
Supabase Storage: 변환된 파일 저장
       │
       ▼
Supabase DB: file_url, position_x, position_y만 저장
       │
       ▼
클라이언트: URL로 이미지 불러와 3D 페이지 텍스처에 적용
```

### 2.3 데이터 흐름 — OCR

```
사용자가 영수증·티켓 이미지 업로드
       │
       ▼
클라이언트: 이미지 유효성 사전 검증 (OCR 호출 횟수 절약)
       │
       ▼
API Route: CLOVA OCR API 호출
       │
       ▼
추출된 텍스트 → 사용자 편집 UI 표시
       │
       ▼
확정 후 Page.text_content에 저장
```

### 2.4 ERD

```
users
  └─ diaries
       │  id, user_id, title, start_date, end_date
       │  location, cover_image_url, created_at
       │
       └─ pages
            │  id, diary_id, page_number
            │  text_content, created_at
            │
            └─ media
                 id, page_id, type (image | ocr)
                 file_url, ocr_text
                 position_x, position_y   ← 페이지 비율 기준 상대값
```

> **설계 원칙:** DB에는 이미지 파일을 직접 저장하지 않습니다. 파일은 Supabase Storage에, DB에는 URL 문자열만 저장하여 DB 성능을 유지합니다.

> **position_x/y:** 절대 픽셀값이 아닌 페이지 크기 대비 비율값(0.0~1.0)으로 저장합니다. 화면 해상도가 달라져도 이미지 배치 위치가 일관되게 유지됩니다.

---

## 3. Key Features & Technical Specs

### Feature 1 — 3D 페이지 넘김 인터랙션

**구현 목표:** 실제 종이 다이어리를 넘기는 물성을 웹에서 재현

**기술 구현:**
- Three.js + React Three Fiber로 다이어리 3D 모델 렌더링
- PlaneGeometry에 페이지 콘텐츠를 텍스처로 매핑
- 마우스 드래그·터치 이벤트 → 3D 모델 회전값 변환으로 페이지 넘김 구현
- 페이지 넘김 시 종이 휘어짐(Bend) 효과와 그림자로 물성 표현

**성능 설계:**
- 폴리곤 수 최소화로 렌더링 부하 감소
- 화면 이동 시 `geometry.dispose()`, `texture.dispose()` 명시적 호출로 메모리 누수 방지
- 저사양 기기·WebGL 미지원 환경에서는 2D 슬라이드 방식으로 자동 fallback

```javascript
// WebGL 지원 여부 체크 및 fallback 처리 예시
const isWebGLAvailable = () => {
  try {
    const canvas = document.createElement('canvas');
    return !!(
      window.WebGLRenderingContext &&
      (canvas.getContext('webgl') || canvas.getContext('experimental-webgl'))
    );
  } catch (e) {
    return false;
  }
};
```

---

### Feature 2 — 페이지 위 자유 배치 편집 UI

**구현 목표:** 사진을 인화해서 다이어리에 붙이는 행위를 재현

**기술 구현:**
- 이미지 업로드 후 드래그로 페이지 내 원하는 위치에 자유 배치
- 배치 좌표를 페이지 비율 기준 상대값(position_x, position_y)으로 저장
- 해상도·화면 크기가 달라져도 배치 위치 일관성 유지

---

### Feature 3 — OCR 오프라인 파편 흡수

**구현 목표:** 영수증·티켓을 다이어리에 끼워 넣는 행위를 재현

**기술 구현:**
- Naver CLOVA OCR API 연동 (한국어 영수증 인식 최적화)
- 클라이언트 사전 검증으로 불필요한 API 호출 방지
- 추출 텍스트 사용자 편집 후 페이지 저장

**비용 관리:**
- 이미지 형식·최소 크기 유효성 검증을 클라이언트에서 먼저 수행
- API 호출 전 중복 요청 방지 로직 적용

---

### Feature 4 — 이미지 최적화 파이프라인

**구현 목표:** DB 용량 부담 없이 고품질 이미지 제공

**기술 구현:**
- 업로드 시 서버에서 `sharp` 라이브러리로 리사이징 + WebP 변환
- DB에는 Supabase Storage URL만 저장 (Blob 직접 저장 없음)
- 프론트엔드는 URL 기반으로 이미지 로드 후 3D 텍스처에 적용

---

## 4. Trouble Shooting

> 이 섹션은 구현 과정에서 마주친 문제와 해결 과정을 기록합니다.  
> 현재 개발 진행 중으로, 문제 발생 시 순서대로 추가 예정입니다.

---

### [예정] TS-001 — Three.js 메모리 누수

**상황:** 페이지 이동 시 3D 텍스처와 geometry가 메모리에 계속 쌓이는 문제  
**원인:** React 컴포넌트가 unmount될 때 Three.js 리소스가 자동으로 해제되지 않음  
**해결 방향:** `useEffect` cleanup 함수에서 `geometry.dispose()`, `material.dispose()`, `texture.dispose()` 명시적 호출

```javascript
useEffect(() => {
  return () => {
    geometry.dispose();
    material.dispose();
    texture.dispose();
  };
}, []);
```

---

### [예정] TS-002 — 이미지 자유 배치 좌표 해상도 불일치

**상황:** 데스크탑에서 배치한 이미지 위치가 모바일에서 어긋나는 문제  
**원인:** 절대 픽셀값으로 position_x/y 저장 시 화면 크기가 달라지면 위치 틀어짐  
**해결 방향:** 페이지 컨테이너 크기 대비 비율값(0.0~1.0)으로 저장, 렌더링 시 실제 크기로 역산

---

### [예정] TS-003 — 모바일 WebGL 성능 저하

**상황:** 저사양 모바일에서 3D 렌더링 시 프레임 드롭 발생  
**원인:** Three.js 기본 설정은 고성능 환경을 전제로 함  
**해결 방향:** User-Agent 및 GPU tier 감지 후 저사양 환경에서 2D fallback 자동 전환

---

### [예정] TS-004 — OCR API 과호출

**상황:** 유효하지 않은 이미지에도 OCR API가 호출되어 비용 낭비 발생  
**원인:** 클라이언트 유효성 검증 없이 바로 API 호출  
**해결 방향:** 파일 형식(jpg/png), 최소 크기, 이미지 블러 정도를 클라이언트에서 사전 검증 후 API 호출

---

## 5. Project Roadmap

### Phase 0 — 기획 및 설계 `완료`

- [x] 요구사항 정의 및 핵심 컨셉 확정
- [x] 오프라인 감각 재현 설계 원칙 수립
- [x] ERD 설계
- [ ] Figma UI 와이어프레임 작성
- [ ] Supabase 초기 스키마 작성

---

### Phase 1 — MVP 개발 `진행 중`

**목표:** 핵심 컨셉을 증명하는 3가지 기능 완성

```
① 3D 페이지 넘김 인터랙션
② 텍스트 + 이미지 자유 배치 입력
③ OCR 영수증·티켓 자동 인식
```

**프론트엔드**
- [ ] Next.js 프로젝트 초기 설정 (TypeScript, ESLint, Prettier)
- [ ] Supabase 인증 연동 (이메일 로그인)
- [ ] 다이어리 목록·생성·삭제 UI
- [ ] 페이지 편집 UI (텍스트 입력, 이미지 자유 배치)
- [ ] Three.js + React Three Fiber 3D 페이지 넘김 구현
- [ ] WebGL 미지원 환경 2D fallback 처리

**백엔드 (Next.js API Routes)**
- [ ] 다이어리 CRUD API
- [ ] 이미지 업로드 → Supabase Storage 연동
- [ ] sharp 이미지 리사이징 + WebP 변환
- [ ] CLOVA OCR API 연동

**인프라**
- [ ] Supabase RLS 정책 설정
- [ ] Vercel 배포 + GitHub Actions CI/CD
- [ ] Sentry 오류 추적 설정

---

### Phase 2 — 오프라인 감각 심화 `예정`

**목표:** MVP에서 재현한 시각 감각을 청각·촉각으로 확장

| 기능 | 오프라인 감각 재현 목표 |
|------|----------------------|
| 종이 질감 효과 (빛 반사·그림자) | 시각적 물성 강화 |
| 페이지 넘김 효과음 | 청각적 물성 재현 |
| 다이어리 커버 커스터마이징 | 소유감·개인화 강화 |
| 페이지 북마크 | 특정 페이지를 펼쳐두는 행위 재현 |
| BGM 연동 | 여행지 공간감 재현 |
| PDF 내보내기·인쇄 | 온·오프라인 순환 완성 |

---

### Phase 3 — 고급 편집 기능 `장기 계획`

> 각 기능이 독립 프로젝트 수준의 공수를 요구하므로 Phase 2 완성 후 우선순위 재검토

- [ ] 손글씨 입력 (Canvas + 필압 처리)
- [ ] 스티커·펜·스탬프 꾸미기 에디터
- [ ] 동영상·음성 기록

---

*문서 버전: v1.0 | 개발 진행에 따라 Trouble Shooting 섹션 지속 업데이트 예정*
