# 테스트 결과서 생성기

## 개요

두 개의 단일 HTML 파일 앱 (외부 의존성 없음 또는 CDN).
CNA(Change) 단위 테스트 결과서를 `.xlsx` 파일로 생성한다 (Kbank QA 용).

## 파일 관리 규칙

| 파일 | 용도 |
|------|------|
| `k_test_result_generator.html` | 사내망용 — ExcelJS 번들 인라인, Pretendard CDN 없음 (완전 오프라인) |
| `k_test_result_generator_cdn.html` | 사외용 — ExcelJS CDN + Pretendard CDN + 사용 가이드 버튼 + 첫 방문 배너 |
| `guide.html` | 사용 가이드 — Netlify 배포용 (`https://k-test-result-generator-guide.netlify.app`) |
| `guide-images/` | 가이드용 원본 스크린샷 (guide.html에 base64 인라인 포함됨 — 배포 불필요) |

- **사내망·사외용 두 파일은 항상 같이 수정한다.** 기능/버그/스타일 변경 시 양쪽 동시 적용.
- CDN 버전 재생성 방법: **반드시 Python 스크립트 사용** — 단순 번들 교체만 하면 CDN 전용 기능(가이드 버튼, 첫 방문 배너)이 소실됨. 스크립트가 ① ExcelJS 번들→CDN 교체 ② 가이드 버튼 CSS/HTML 주입 ③ 첫 방문 배너 HTML 주입 ④ localStorage JS 주입 ⑤ `dismissFirstVisit()` 함수 주입을 모두 자동 처리.
- 사내망 버전에는 `@import url('.../pretendard.css')` 없음 → 폰트 폴백: `'Pretendard', sans-serif` (시스템 기본 폰트)
- **사용 가이드 버튼**은 CDN 버전 헤더에만 존재 (`📖 사용 가이드` → `https://k-test-result-generator-guide.netlify.app`, `target="_blank"`). 사내망 버전에는 없음.
- `guide.html` 수정 시 Netlify Drop(`https://app.netlify.com/drop`)에 `guide.html` 단일 파일만 올리면 됨 (이미지 base64 인라인).

## 사용 가이드 (`guide.html`)

- 단일 HTML 파일, 외부 의존성 없음 (이미지 5장 base64 인라인, ~2MB)
- 섹션 구성: 소개 → 시작하기 → 기본 정보 → TC 관리 → 이미지 첨부 → 어노테이션 → TC별 날짜 → 임시저장 → 엑셀 내보내기 → 다크모드
- 각 섹션에 실제 스크린샷 포함 (클릭 시 라이트박스 확대)
- `guide-images/` 폴더의 원본 이미지를 교체할 경우 Python으로 base64 재임베딩 필요:
  ```bash
  python3 -c "
  import base64, re
  images = {'guide-images/tc-card.png':None,'guide-images/annot-box.png':None,'guide-images/annot-mosaic.png':None,'guide-images/excel-sheet1.png':None,'guide-images/excel-sheet2.png':None}
  for p in images:
      images[p] = 'data:image/png;base64,' + base64.b64encode(open(p,'rb').read()).decode()
  html = open('guide.html',encoding='utf-8').read()
  for p,d in images.items(): html=html.replace(f'src=\"{p}\"',f'src=\"{d}\"')
  open('guide.html','w',encoding='utf-8').write(html)
  "
  ```

## CDN 버전 전용 기능

### 첫 방문 배너 (`#firstVisitBanner`)
- **최초 2회 진입 시**에만 표시: 기존 웰컴 배너(`#welcomeBanner`) 대신 파란 그라디언트 배너 노출
- "👋 이 프로그램이 처음이신가요?" + `📖 사용 가이드 보기` + `이미 알고 있어요`
- localStorage 키: `k-test-guide-visits` (방문마다 +1, 2 이상이면 일반 웰컴 배너만 표시)
- 관련 함수: `dismissFirstVisit()` — 배너 페이드아웃 후 `#welcomeBanner` 표시

## 파일 구조

### 사내망용 (`k_test_result_generator.html`, ~1507줄)

- **8~55줄**: ExcelJS + JSZip 번들 (minified, **수정 금지**)
- **56~560줄**: CSS + HTML 마크업
- **561~1507줄**: 앱 JS 로직 (`<script>` 블록)

### 사외용 (`k_test_result_generator_cdn.html`, ~60KB)

- **8줄**: ExcelJS CDN `<script src>` 한 줄
- **9줄~**: CSS + HTML + 앱 JS (사내망 버전과 동일)


## 입력 필드

| id | 필수 | 설명 |
|----|------|------|
| `#chaName` | ✅ | CHA명 (예: CHA-1234) |
| `#testerName` | ✅ | 테스터 이름 |
| `#testDate` | ✅ | 테스트 날짜 (기본값: 오늘) |
| `#testTitle` | ✅ | 테스트 제목 (Sheet1에 출력) |
| `#testDesc` | ✅ | 테스트 설명 (Sheet1에 출력) |

- 필수 필드는 라벨에 빨간 `*` 표시
- TC 제목도 필수 (`*` 표시), 엑셀 내보내기 시 미입력 TC 번호 알림

## 핵심 데이터 구조

```js
testCases = [{ id, title, desc, testDate, customValues, images: [{ name, dataUrl, type, nw, nh, origDataUrl, rects, annotated }] }]
```

- `testDate`: TC별 테스트 날짜 (전역 `perTcDate` ON 시 사용, 기본값 = 전역 testDate)
- `customValues`: 커스텀 필드 값 `{ '테스트 환경': 'STG', '성공여부': 'O' }` — 전역 `customFields` 배열 기준
- `nw`/`nh`: 리사이즈 후 픽셀 크기 (엑셀 이미지 높이 자동 계산용)
- `origDataUrl`: 어노테이션 전 원본 (재편집 가능하도록 보존)
- `rects`: 어노테이션 목록 `[{x,y,w,h,color,lineWidth,type}]` — **자연 이미지 좌표** (scale 독립적)
  - `type`: `'box'` (테두리 강조) | `'mosaic'` (픽셀화 모자이크)
- `annotated`: 어노테이션 적용 여부 (썸네일 빨간 테두리 표시)

## 주요 함수

| 함수 | 역할 |
|------|------|
| `addTC()` / `removeTC(id)` | TC 추가/삭제 (새 TC의 testDate = 전역 testDate, customValues = `{}` 기본값) |
| `updateField(id, field, val)` | 필드 실시간 업데이트 (title → 헤더 미리보기, testDate → 헤더 뱃지 동기화) |
| `togglePerTcDate(checked)` | TC별 날짜 모드 ON/OFF, ON 시 기존 TC에 전역 날짜 기본값 채움 |
| `addCustomField()` | 커스텀 필드 추가 → 전체 TC에 빈 값 초기화 후 재렌더 |
| `removeCustomField(idx)` | 커스텀 필드 삭제 (값 있으면 confirm 경고) |
| `renderCustomFieldsMgr()` | 커스텀 필드 chip 목록 재렌더 |
| `updateCustomValue(id, cfIdx, val)` | TC별 커스텀 필드 값 실시간 업데이트 |
| `renderList()` | 전체 TC 목록 재렌더링 |
| `renderImgs(id)` | 특정 TC 이미지 썸네일 재렌더링 |
| `getMaxImgPx()` | 품질 설정 radio 읽어 최대 px 반환 (0→Infinity) |
| `resizeDataUrl(dataUrl, type, cb)` | 업로드 이미지를 `getMaxImgPx()` 기준으로 리사이즈 후 콜백 |
| `handleFiles(files, id)` | FileReader → resizeDataUrl → tc.images.push |
| `openAnnotEditor(tcId, imgIdx)` | 어노테이션 모달 열기 |
| `annotSave()` | 오프스크린 캔버스에 원본 해상도로 합성 후 저장 |
| `exportExcel()` | 2시트 xlsx 생성 및 base64 다운로드 |
| `toggleTheme()` | 라이트/다크 모드 전환 (localStorage 저장) |
| `updateFab()` | FAB 버튼 표시 여부 갱신 (isDirty && scrollY > 80) |
| `hideClipboardFloat()` | 클립보드 플로팅 패널 숨김 + `clipboardFloatFile` 초기화 |

## 이미지 품질 설정

`input[name="imgQuality"]` radio 3종 (기본 정보 카드 하단):

| 값 | 표시 | 설명 |
|----|------|------|
| `600` | 빠름 · 최대 600px | 느린 PC, 사진 많을 때 |
| `800` | 기본 · 최대 800px **추천** | 기본 선택 (`checked`) |
| `0` | 고화질 · 원본 유지 | `getMaxImgPx()` → `Infinity` |

- JPEG는 quality `0.88`로 압축, PNG는 무손실
- **새로 추가하는 이미지부터** 적용 (기존 이미지 소급 없음)
- 화질↑ → JSON·엑셀 파일 크기↑, 내보내기 시간↑

## Excel 내보내기

### 파일명
`{chaName}_테스트결과서_{tester}_{yyyyMMdd}.xlsx`

### 유효성 검사 (exportExcel 진입 시)
1. 기본 정보 3개(chaName, tester, dateVal) 미입력 → 차단
2. testTitle / testDesc 미입력 → 차단
3. testCases 없음 → 차단
4. TC 제목 미입력 → "N번 제목을 입력해주세요" 차단

### Sheet 1 — 기본 정보 + TC 목록
- 컬럼: A(label/#, 18) / B(value/제목, 28) / C(value/설명, 38) — 3컬럼, 빈칸 없음
- 헬퍼: `s1SecHdr(text, bgHex)` — A:C 병합 / `s1InfoRow(label, value, h)` — B:C 병합

| 영역 | 내용 |
|------|------|
| 타이틀 | "테스트 결과서" (A:C 병합, ACCENT 배경) |
| 기본 정보 | CHA명 / 테스터 이름 / 테스트 날짜 / 테스트 제목 / 테스트 설명 |
| TC 목록 | 섹션 헤더 → 유의사항(HEADER_BG, 이탤릭) → 컬럼헤더 → TC 행 |

### Sheet 1 — 기본 정보 + TC 목록 (커스텀 필드 시)
- `cfCount = customFields.length`, `s1LastCol = String.fromCharCode(67 + cfCount)` ('C'~)
- 타이틀·섹션 헤더·유의사항·기본 정보 값 셀 모두 `A:s1LastCol` / `B:s1LastCol` 동적 병합
- TC 컬럼 헤더: `# | 제목 | 설명 | 커스텀필드1 | ...`
- TC 데이터 행: `idx+1 | title | desc | customValues[f1] | ...`

### Sheet 2 — 테스트 케이스
- 컬럼 (`perTcDate=false`, `cfCount=0`): A(#, 5) / B(제목, 28) / C(설명, 40) / D~(사진)
- 커스텀 필드 있으면 설명 다음에 커스텀 필드 컬럼들(각 22) 삽입 후 사진 컬럼
- `dateOffset = perTcDate ? 1 : 0`, `customOffset = cfCount`
- `lastImgColLetter = String.fromCharCode(68 + imgColCount - 1 + dateOffset + customOffset)`
- 날짜 셀 값: `tc.testDate || dateVal` (TC별 날짜 없으면 전역 날짜 폴백)
- 이미지 컬럼 수 = TC 중 최대 이미지 개수 (`imgColCount`)
- 이미지 크기: `IMG_W_PX=200`, 높이 = `IMG_W_PX * nh/nw` (비율 유지)
- 2-패스: Pass1(행·스타일) → Pass2(이미지 삽입) — ExcelJS twoCell 앵커 버그 회피
- 다운로드: **base64 data URL** (Blob URL 환경 차단 대응)

## 색상 팔레트 (Kbank)

| 변수 | hex | 용도 |
|------|-----|------|
| ACCENT | 0114A7 | 번호 셀, 타이틀 배경 |
| ACCENT_D | 000D7A | 섹션 헤더, 컬럼 헤더 |
| HEADER_BG | E0E6F1 | 메타 라벨 배경 |

## 이미지 어노테이션

- 모달: `#annotOverlay` — 캔버스 드래그로 어노테이션 그리기
- 툴바 모드 토글: **□ 박스** / **▦ 모자이크** (`setAnnotMode()`)
  - 박스 모드: 색상 프리셋 5종 + 커스텀 color picker, 선 굵기 range (1~12)
  - 모자이크 모드: 색상·선굵기 UI 자동 숨김, 영역을 픽셀화 처리
- `annot.mode`: `'box'` | `'mosaic'` (sticky — 모달 닫아도 유지)
- `annotUndo()` / `annotClear()` / `annotSave()`
- **캔버스 배경 자동**: 이미지 로드 시 60×60 다운샘플 → 평균 밝기 계산
  - 밝은 이미지(>128) → 배경 `#1C1C2E` (어두움)
  - 어두운 이미지(≤128) → 배경 `#E8E8EE` (밝음)
- **좌표 저장 방식**: rects는 **자연 이미지 좌표**로 저장 (scale 독립적)
  - `annotSave()`: display 좌표 `× inv(=1/scale)` → 자연 좌표로 저장
  - `openAnnotEditor()`: image.onload에서 자연 좌표 `× scale` → display 좌표 복원
  - JSON 불러와도 화면 크기 무관하게 박스 위치 정확

### 어노테이션 렌더링 함수

| 함수 | 역할 |
|------|------|
| `drawAnnotShape(ctx, r)` | `r.type`에 따라 박스 또는 모자이크 분기 |
| `drawAnnotRect(ctx, r)` | 테두리 박스 + 10% 투명 채우기 |
| `setAnnotMode(mode, btn)` | 모드 전환 + 색상/두께 UI 토글 + 힌트 문구 변경 |

- 모자이크 알고리즘: rect 영역을 `min(w,h)/10` 크기 블록으로 축소 후 `imageSmoothingEnabled=false`로 재확대 (픽셀화)
- `annotSave()` 오프스크린 캔버스에서도 동일 알고리즘으로 원본 해상도 적용

## 이미지 기타 기능

- **드래그 순서 변경**: `dragState.tcId/fromIdx` → drop 시 `splice` 재정렬
- **가로형 이미지 경고**: `nw > nh` → 6초 경고 토스트
- **클립보드 붙여넣기** (`Ctrl+V`): `paste` 이벤트로 클립보드 이미지 감지 → 우측 중앙 플로팅 패널(`#clipboardFloat`) 표시 → TC dropzone에 드래그앤드랍으로 추가 후 패널 자동 소멸
  - 재차 Ctrl+V 시 패널 이미지 덮어쓰기
  - 텍스트 input/textarea 포커스 중이거나 어노테이션 모달(`#annotOverlay.open`) 열린 상태에서는 무반응
  - `clipboardFloatFile`: 현재 클립보드 이미지 File 객체 (전역)
  - `clipboardDropSucceeded`: drop 성공 여부 플래그 — `dragend`에서 확인 후 패널 닫기
  - `onDrop`에서 `e.dataTransfer.getData('cf-source') === '1'` 로 클립보드 드랍 구분
  - **⚠️ 수정 시 주의**: `e.preventDefault()`는 반드시 `e.clipboardData` null 체크 + `getAsFile()` null 체크 이후에 호출할 것. macOS에서 Finder 드래그 중 paste 이벤트가 함께 발화될 수 있고, 이때 `getAsFile()`은 null을 반환함 — null 확인 전에 `preventDefault`를 호출하면 드래그앤드랍이 간헐적으로 차단됨 (macOS Safari/Chrome 공통)

## JSON 임시저장 / 불러오기

### UX 흐름
- **시작 시**: 웰컴 배너(`#welcomeBanner`) 표시, `.main.dimmed` (blur+반투명) → 선택 전 폼 비활성
- **선택 후**: `dismissWelcome()` → 배너 페이드아웃 + `.dimmed` 제거
- **작업 중**: `markDirty()` → 헤더 임시저장 버튼 빨간 점 + FAB 버튼 등장
- **FAB**: 스크롤 80px 이상 + isDirty → 우하단 고정 "💾 임시저장" 버튼 표시
- **임시저장**: `saveToJSON()` → Blob 다운로드 → `markClean()`
- **불러오기**: `openJSON()` → `#jsonFileInput` → `onJsonFileSelect()` → `restoreSnapshot()`
- **탭 닫기**: `beforeunload` — `isDirty && testCases.length > 0` 이면 경고
- **엑셀 내보내기**: 하단 "📊 엑셀로 내보내기" (최종 결과물)

### JSON 파일명
`{chaName}_테스트결과서_{tester}_{yyyyMMdd}_{HHmmss}.json`

### JSON 저장 구조
```js
{ chaName, testerName, testDate, testTitle, testDesc, perTcDate, customFields, testCases, tcCounter, savedAt }
```
- `perTcDate`: TC별 날짜 모드 플래그 (불러오기 시 체크박스 상태 복구)
- `customFields`: 커스텀 필드명 배열 (없으면 `[]` — 기존 JSON 하위 호환)
- TC별 `customValues` 포함, 기존 JSON의 TC에 없으면 `{}` 자동 초기화
- 이미지 base64 + rects(자연 좌표 + type) + TC별 testDate 포함 → 완전 복구

### 주요 함수
| 함수 | 역할 |
|------|------|
| `buildSnapshot()` | 현재 상태 직렬화 |
| `saveToJSON()` | Blob 다운로드 + markClean + updateFab |
| `restoreSnapshot(data)` | 폼·testCases 완전 복구 |
| `openJSON()` | `#jsonFileInput` 트리거 |
| `onJsonFileSelect(e)` | FileReader → restoreSnapshot |
| `markDirty()` / `markClean()` | isDirty 플래그 + 버튼 빨간 점 + updateFab |
| `updateFab()` | `#fabSave` 표시 여부 갱신 |
| `dismissWelcome()` | 배너 페이드아웃 + `.main.dimmed` 제거 |
| `dismissFirstVisit()` | 첫 방문 배너 페이드아웃 후 웰컴 배너 표시 (CDN 전용) |

## 수정 시 주의

- ExcelJS 번들(사내망 버전 8~55줄)은 건드리지 말 것.
- 이미지 컬럼 너비 변경 시 `IMG_COL_W = IMG_W_PX / 7` 상수만 수정.
- Sheet1 병합 범위 변경 시 `s1LastCol = String.fromCharCode(67 + cfCount)` 기준으로 타이틀·섹션헤더·infoRow 모두 수정할 것.
- Sheet2 이미지 컬럼 문자 계산: `String.fromCharCode(68 + imgColCount - 1 + dateOffset + customOffset)` — 날짜·커스텀 필드 수 합산.
- 커스텀 필드 관련 로직 변경 시 `addCustomField` / `removeCustomField` / `renderList` / `exportExcel` 모두 확인할 것.
- 저장 방식 변경 시 blob URL 환경 이슈 고려.
- rects 좌표 체계 변경 시 `annotSave` / `openAnnotEditor` 양쪽 모두 수정할 것.
- rects에 새 필드 추가 시 `annotSave`(저장) / `openAnnotEditor`(복원 시 map) 양쪽 모두 반영할 것.
- 어노테이션 렌더링 변경 시 `drawAnnotShape`(캔버스 미리보기) + `annotSave`(오프스크린 합성) 양쪽 동일하게 적용할 것.
