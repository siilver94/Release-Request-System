# _README.md — 릴리즈 상태 요청 시스템 구조 명세

> 상태: **핵심 워크플로우 개조 완료**. 요청 등록→진행중→수정완료→작업완료/강제완료까지 전체 사이클 구현됨(필드 편집만 Studio에서 확인 필요). 남음: DX 편의 기능(일괄/선택 강제완료, Export), 이메일 4종 중 나머지 실물 테스트, 3일 리마인드 flow.
> 개조가 진행되면 이 문서를 계속 최신화한다 (변경마다 X, 큰 단계 끝날 때 배치로).

---

## 1. 데이터 소스

### 1-1. `ReleaseRequest_DB` (신규 · 메인 리스트)
릴리즈 상태 변경 요청 1건 = 1행.

| 컬럼(내부명) | 표시명 | 타입 | 입력주체 | 설명 |
|---|---|---|---|---|
| `Title` | 제목 | 단일텍스트 | 자동 | 품번 값 복사(SharePoint 필수 기본컬럼, 검색·정렬용) |
| `PartNo` | 품번 | 단일텍스트 | 요청자 | **필수**. PLM 품번 |
| `DrawingType` | 도면종류 | 선택(복수 허용) | 요청자 | **필수**. 3D / 2D / WTPart 다중선택 |
| `RequesterName` | 요청자명 | 단일텍스트 | 자동 | varMyName |
| `RequesterTeam` | 요청팀 | 단일텍스트 | 자동 | varMyTeam |
| `RequesterEmail` | 요청자메일 | 단일텍스트 | 자동 | varMyEmail |
| `Stage` | 단계 | 선택 | 시스템 | 진행중 / 완료 |
| `Status` | 상태 | 선택 | 시스템 | 요청됨 / 수정완료 / 작업완료 / 강제완료 |
| `SubmittedAt` | 요청일시 | 날짜및시간 | 자동 | 3일 경과 계산 기준 |
| `ModifiedByDXAt` | DX수정완료일시 | 날짜및시간 | 시스템 | DX가 "수정완료" 누른 시각 |
| `CompletedAt` | 완료일시 | 날짜및시간 | 시스템 | 완료로 넘어간 시각 |
| `HandlerEmail` | DX담당자메일 | 단일텍스트 | 시스템 | 처리한 DX 담당자 |
| `Note` | 비고 | 여러줄텍스트 | 요청자(선택) | 변경 사유 등. 선택 입력 |

> ⚠️ `DrawingType`은 SharePoint에서 **"여러 값 선택 가능"을 반드시 켜야** 함 (안 켜면 Concat 등에서 오류). `Stage`/`Status`/`DrawingType`은 Choice 타입이라 읽을 땐 `.Value`, Patch로 쓸 땐 `{Value: "텍스트"}`로 감싸야 함.

### 1-2. `UserInfo_DB` (원본 재사용)
로그인 사용자 조직정보 조회용. 컬럼: `e-mail`, `이름_한글`, `부문`, `팀`, `직급2` 등.

### 1-3. 원본에서 **제거**할 데이터 소스
- `das` (작업의뢰 메인) → `ReleaseRequest_DB`로 교체
- `Project_DB`, `WorkRequestComments_DB` → 릴리즈 시스템에서 미사용, 참조 제거

---

## 2. 상태 모델

| 컬럼 위치 | Stage | Status(배지) | 트리거 | 발송 메일 |
|---|---|---|---|---|
| 진행중 | `진행중` | `요청됨` | 요청자 제출 | → DX팀 |
| 진행중 | `진행중` | `수정완료` | DX "수정완료" | → 요청자 |
| 완료 | `완료` | `작업완료` | 요청자 "작업완료" | → DX팀 |
| 완료 | `완료` | `강제완료` | DX "강제완료" | → 요청자 |

- 승인/반려 상태 **없음**.
- "3일 경과" = `DateDiff(SubmittedAt, Now(), Days) >= 3` 이고 아직 `완료`가 아닌 건.

---

## 3. 전역 변수 (App.OnStart / Screen.OnVisible)

### 3-1. 유지 (원본 재사용)
| 변수 | 용도 |
|---|---|
| `varMyEmail` | 로그인 사용자 메일 (소문자) |
| `varMyInfo` | UserInfo_DB 조회 결과 |
| `varMyName` | 최종 표시 이름 |
| `varMyDept`, `varMyTeam`, `varMyJobGrade` | 조직정보 |
| `varIsViewer` | 관리자(전체조회) 여부 |
| `varShowRequestPopup` | 요청 팝업 열림 |
| `varRequestMode` | New / View / Edit |
| `varSelectedRequest` | 현재 선택 요청 |
| `varSearchText` | 검색어 |
| `varShowToast`, `varToastMessage` | 토스트 알림 |

### 3-2. 신규 추가

**App.OnStart (전역, 앱 시작 시 1회 설정 · 반영 완료)**
| 변수/컬렉션 | 용도 |
|---|---|
| `colDxTeam` | DX팀 담당자 목록 {DxEmail, DxName}. 현재: 김봉진(bongjin.kim@tym.world), 최인호(inho.choi@tym.world) |
| `varDxRecipients` | DX팀 메일 발송용 수신자 문자열 (세미콜론 구분, colDxTeam.DxEmail 결합) |
| `colDrawingTypeChoices` | 도면종류 선택지 {Value}: 3D / 2D / WTPart |

**Screen.OnVisible (사용자별, 반영 완료)**
| 변수 | 용도 |
|---|---|
| `varIsDX` | 로그인 사용자가 DX팀인지: varMyEmail in colDxTeam.DxEmail |
| `varPeriodFilter` | 기간 필터: 이번주 / 이번달 / 날짜선택 / 전체 (기본값 "이번달") |
| `varDateFrom`, `varDateTo` | 조회 기간 범위 (날짜선택 모드는 dpDateFrom/dpDateTo가 직접 갱신) |
| `varOnlyOverdue` | "3일 경과 건만" 필터 토글 (con_searchFilter의 tglOnlyOverdue와 연결, 반영 완료) |
| `colSelectedForce` | 선택 강제완료용 체크된 건 컬렉션 (변수는 선언됨, 실제 체크박스 UI는 미구현 - DX 편의 기능 단계에서 작업) |
| `varIsViewer` 판별 이메일 | `eunseong.kim@tym.world` 1개만 (본인) |

### 3-3. 제거 (승인 흐름 잔재)
`varIsWorker`, `varIsTeamLeader`, `varWorkerEmail`, `varShowSelfApproveConfirmPopup`, 반려 관련 상태 → 릴리즈 시스템에서 재설계/제거.

---

## 4. 스크린 목록

| 스크린/컴포넌트 | 상태 | 비고 |
|---|---|---|
| `App.OnStart.yaml.txt` | ✅ 완료 | colDxTeam, varDxRecipients, colDrawingTypeChoices |
| `WorkRequest_Screen.yaml.txt` | 개조 중 | → 릴리즈 요청 메인 화면. 추후 파일명 변경 검토 |

### 메인 화면 주요 컨트롤 (진행 현황)

| 컨트롤 | 상태 | 내용 |
|---|---|---|
| `Container7` / `beforeTask_con` / `galStart` (시작전) | ✅ 삭제 완료 | |
| `Container8`(진행중), `Container9`(완료) | ✅ 위치 이동 완료 | X: 293 / 657 (필터 우측부터 순서대로) |
| `galInProgress`, `galCompleted` | ✅ 데이터 바인딩 완료 | `ReleaseRequest_DB` 기준, Stage.Value 필터, Status.Value 배지(요청됨/수정완료/작업완료/강제완료), 품번+도면종류 표시, 3일경과 빨간 강조 |
| `con_searchFilter` | ✅ 개조 완료 | `ddPeriodFilter`(이번주/이번달/날짜선택/전체) + `dpDateFrom`/`dpDateTo`(신규, 날짜선택 모드에서만 표시) + `txtSearch`(품번/요청자 검색) + `tglOnlyOverdue`(기존 tglIncludeCompleted 재활용, "3일 경과 건만"). `tglIncludeRejected`는 삭제 |
| `new_btn`(헤더 "요청 등록") | ✅ 완료 | `Defaults(ReleaseRequest_DB)` 기준으로 New 모드 시작 |
| `OuterRactangle`(팝업 바깥 클릭 닫기) | ✅ 완료 | 첨부파일 로직 제거, ReleaseRequest_DB 기준 |
| `frmRequest`(요청 등록 폼) | ✅ 완료 | DataSource=ReleaseRequest_DB. **필드 구성은 Studio "필드 편집"으로 직접 교체**(품번/도면종류/비고, 필수=품번·도면종류) → 이 파일의 DataCard 자식 노드는 원본(작업의뢰) 그대로라 실제 Studio 상태와 다름(의도된 것). OnSuccess에서 Stage/Status 초기화 + DX팀 메일 발송 |
| `btnRequestSave`, `btnRequestCancel` | ✅ 완료 | 필수값 검증은 폼 Required로 위임, SubmitForm만 호출 |
| `Container5`(등록 팝업 헤더) | ✅ 완료 | 제목 "릴리즈 상태 변경 요청"으로 교체 |
| `conRequestView`(상세보기), `Container2`(상세보기 헤더) | ✅ 완료 | 요약정보 박스: 팀=RequesterTeam, "희망완료일"→"도면종류". 비고 박스로 단순화. 헤더 배지/제목 Status.Value·PartNo 기준 |
| `Container3_2`(frmRequestAttachmentView, 첨부파일) | ✅ 삭제 완료 | 첨부파일 기능 없음 |
| `lblRejectReasonTitle` 등 반려 라벨 3종 | ✅ 삭제 완료 | 반려 없음 |
| `conRequesterAction` (재구성) | ✅ 완료 | 요청자 "작업완료" 버튼. Visible: 본인 요청 && Status=수정완료 |
| `conDxAction` (신규, conLeaderAction+conWorkerAction 대체) | ✅ 완료 | DX "수정완료"(Status=요청됨일 때만)/"강제완료"(항상) 버튼. Visible: varIsDX && Stage=진행중 |
| `OuterRactangle_1`, `conSelfApproveConfirmPopup`, `conDeleteConfirmPopup` | ✅ 삭제 완료 | 트리거 버튼 소실로 죽은 코드였음 (원본의 "의뢰 삭제/대리승인" 기능. 필요시 추후 "요청 취소" 기능으로 재설계 가능) |
| `conDxHeaderTools`("일괄 강제완료"/"Export" 버튼, 헤더) | ✅ 완료 | Visible=varIsDX. 일괄강제완료: 임시 컬렉션(colBulkOverdue)에 먼저 담고 ForAll+Patch (원본 데이터소스 직접 순회 Patch 불가 이슈로) |
| `btnSelectedForceComplete`(진행중 헤더 바 안, 체크된 건만 강제완료) | ✅ 완료 | Visible=varIsDX && 선택건수>0 |
| `chkSelect`(진행중 카드 체크박스) | ✅ 완료 | colSelectedForce에 담김. **주의**: colSelectedForce는 `ClearCollect(colSelectedForce, Filter(ReleaseRequest_DB, false))`로 초기화해야 스키마 인식됨(빈 배열 `[]`로 초기화하면 필드 인식 안 됨) |
| `btnExportCsv` / `conExportPopup` | ⏸️ 보류 | 코드는 작성됐으나(팝업에 CSV 텍스트 표시 후 복사-붙여넣기 방식) Studio 미반영. **Power Automate로 실제 파일 다운로드 vs 지금의 복사-붙여넣기 방식 vs SharePoint에서 수동 export, 방식 결정 필요** |

---

## 5. 기능 명세

### 5-1. 필터
- 기간: 이번주 / 이번달 / 날짜선택 / 전체
- "3일 경과 건만" 토글
- 검색: 품번 / 요청자

### 5-2. DX 편의 기능
- **단건 강제완료**: 진행중 카드에서 개별 완료 처리 → 요청자 메일
- **일괄 강제완료**: 요청 후 3일 경과한 진행중 건 전체를 완료 처리
- **선택 강제완료**: 진행중 카드 체크박스로 선택한 건들만 완료 처리
- 강제완료 시 `Status = 강제완료`, `CompletedAt = Now()`, 요청자 메일

### 5-3. Export
- 현재 목록을 **CSV**로 다운로드 (앱 내 생성, 조회 확인용)

---

## 6. 이메일 (4종, 앱 내 `Office365Outlook.SendEmailV2`)
| # | 시점 | 수신 | 내용 |
|---|---|---|---|
| 1 | 요청 제출 | DX팀 | 새 릴리즈 상태 변경 요청 도착 |
| 2 | DX 수정완료 | 요청자 | 수정 완료됨, 작업완료 확인 요망 |
| 3 | 요청자 작업완료 | DX팀 | 요청자 작업완료 확인 |
| 4 | 강제완료 | 요청자 | 강제완료 처리됨 |

## 7. 별도 flow (앱 외부)
- **3일 경과 리마인드**: Power Automate 스케줄 flow (매일 반복). `Stage != 완료` & 3일 경과 건에 요청자 리마인드 메일.
