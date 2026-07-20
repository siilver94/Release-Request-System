# _README.md — 릴리즈 상태 요청 시스템 구조 명세

> 상태: **설계 확정 / 개조 착수 전**. 실제 화면(`WorkRequest_Screen.yaml.txt`)은 아직 원본(시작실 작업의뢰) 상태다.
> 개조가 진행되면 이 문서를 계속 최신화한다.

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
| `varShowDeleteConfirmPopup` | 삭제 확인 팝업 |

### 3-2. 신규 추가
| 변수 | 용도 |
|---|---|
| `varDxHandlerEmail` | DX팀 수신용 메일 (요청 알림 대상) |
| `varIsDX` | 현재 사용자가 DX팀 담당자인지 |
| `varPeriodFilter` | 기간 필터: 이번주 / 이번달 / 날짜선택 / 전체 |
| `varDateFrom`, `varDateTo` | 날짜선택 시 범위 |
| `varOnlyOverdue` | "3일 경과 건만" 필터 토글 |
| `colSelectedForce` | 선택 강제완료용 체크된 건 컬렉션 |

### 3-3. 제거 (승인 흐름 잔재)
`varIsWorker`, `varIsTeamLeader`, `varWorkerEmail`, `varShowSelfApproveConfirmPopup`, 반려 관련 상태 → 릴리즈 시스템에서 재설계/제거.

---

## 4. 스크린 목록

| 스크린/컴포넌트 | 상태 | 비고 |
|---|---|---|
| `App.OnStart.yaml.txt` | 신규 작성 | 데이터소스 연결·전역변수 초기화 |
| `WorkRequest_Screen.yaml.txt` | 개조 중 | → 릴리즈 요청 메인 화면. 추후 파일명 변경 검토 |

### 메인 화면 주요 컨트롤 (개조 계획)
| 원본 컨트롤 | 조치 |
|---|---|
| `Container7` / `beforeTask_con` / `galStart` (시작전) | **삭제** |
| `Container8` / `galInProgress` (진행중) | 유지 + 체크박스·강제완료 추가 |
| `Container9` / `galCompleted` (완료) | 유지 |
| `con_searchFilter` | 기간필터 교체 + 3일경과 토글 추가 |
| `frmRequest` (요청 폼) | 품번 + 도면종류(다중) 카드로 교체, 나머지 제거 |
| `conLeaderAction` / `conWorkerAction` 등 승인·반려 UI | **제거/재설계** |
| 신규 버튼 | 단건 강제완료 / 일괄 강제완료 / 선택 강제완료 / Export(CSV) |

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
