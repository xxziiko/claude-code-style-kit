# 테이블 가로스크롤 정책

## 결정된 UX 기준

- 사이드바 open이 기본 상태다.
- **1600px 이상 desktop + sidebar open** 환경에서는 일반 테이블이 가로스크롤 없이 컨테이너 폭에 맞춰 보여야 한다.
- **1600px 미만**에서는 컬럼 최소 가독 폭을 보존하기 위해 테이블 내부(`TableContainer`) 가로스크롤을 허용한다.
- 모든 viewport에서 **페이지 전체 horizontal scroll은 발생하지 않아야 한다**.

## 구현 방법

### 레이아웃 레벨 (`Layout.tsx`)

- `Main` 영역에 `overflowX: 'hidden'` 적용 → 페이지 전체 horizontal overflow 차단
- content Box의 `minWidth: 1400` 제거 → `width: '100%'`, `maxWidth: '100%'`, `boxSizing: 'border-box'` 로 대체
- `maxWidth: calc(100vw - 280px)` → `calc(100vw - ${LEFT_NAV_WIDTH}px)` (상수 기반으로 일관화)

### 공통 테이블 레벨 (`DataTable`, `ListTable`)

- `TableContainer`에 `overflowX: 'auto'` 기본 적용
- `maxHeight` 전달 시에는 `overflow: 'auto'` (기존 동작 유지)
- 이 방식으로 flex row (`MuiTableCell-flex` + `flexShrink: 0`)가 컨테이너보다 넓어질 경우 **테이블 내부에서만** 가로스크롤이 발생하고 페이지 전체 스크롤은 발생하지 않는다.

## wide/max-content 예외 테이블 목록

아래 테이블은 컬럼 수가 많거나 편집 가능한 인라인 필드가 포함되어 이번 정책 적용 대상에서 제외한다.
후속 리디자인(컬럼 정리, drawer 전환 등) 시 별도 작업이 필요하다.

| 파일 경로                                                                                    | 현재 폭 기준                | 예외 이유                                                           | 후속 리디자인 필요                            |
| -------------------------------------------------------------------------------------------- | --------------------------- | ------------------------------------------------------------------- | --------------------------------------------- |
| `src/components/business/orders/modals/addressModal/Table.tsx`                               | `minWidth: 3400`            | 23개 컬럼 + 인라인 주소 편집 필드 포함. 컨텐츠 양이 많아 압축 불가  | 필요 — 컬럼 재정의 또는 편집 drawer 전환 검토 |
| `src/components/wms/receipt/receiptGroup/modify/ReceiptGroupUpdateModal.tsx`                 | `minWidth: max-content`     | 입고 그룹 수정 모달 내 테이블. 동적 컬럼 수에 따라 폭이 변동됨      | 필요 — 컬럼 수 고정 또는 가변 레이아웃 재검토 |
| `src/components/wms/picking/modals/PickingListFormModal.tsx`                                 | `minWidth: 1240`            | 피킹 목록 생성 모달 내 테이블. 모달 폭이 고정돼 있어 별도 판단 필요 | 검토 필요                                     |
| `src/components/business/orders/estimate/Form.tsx`                                           | `minWidth: 1240`            | 견적서 폼 내 인라인 편집 테이블                                     | 검토 필요                                     |
| `src/components/wms/stockInquiry/receiptAudit/drawer/ReceiptAuditAdjustmentResultDrawer.tsx` | `minWidth: 900~1100` (동적) | 재고 실사 조정 결과 drawer 내 테이블. drawer 폭 내 스크롤 허용      | 검토 필요                                     |
| `src/components/wms/stockInquiry/receiptAudit/drawer/ReceiptSaveDrawer.tsx`                  | `minWidth: 1000`            | 재고 실사 저장 drawer 내 테이블                                     | 검토 필요                                     |

## 참고

- `src/components/wms/stockInquiry/frozenOem/FrozenOemTable.tsx`는 이미 `overflow: 'auto'` + `maxWidth: '100%'` + `tableLayout: 'fixed'`로 올바르게 처리되어 있다.
- Drawer 내부 테이블(`minWidth: 400~1000`)은 drawer 자체 폭 안에서 스크롤되므로 페이지 전체 스크롤에 영향을 주지 않는다.
