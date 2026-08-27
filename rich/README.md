# 지현제 가계부

단일 HTML 파일(`index.html`) + 바닐라 JS + Firebase Firestore로 만든 가계부 앱입니다.
빌드 도구 없이 정적 파일을 그대로 서빙하면 동작합니다. 지금은 아래 두 화면만 구현되어 있습니다.

- 화면 1: 캘린더 (가계부별 월별 달력, 날짜 클릭 시 수입/지출 입력)
- 화면 2: 수입/지출 대시보드 (카테고리별 도넛 차트 + 리스트)

자산 항목 관리, 자산 대시보드(3~4번 화면)는 아직 만들지 않았습니다.

## 실행 방법 (VS Code Live Server)

1. VS Code에서 `Live Server` 확장을 설치합니다 (Ritwick Dey 제작, 이미 설치되어 있다면 생략).
2. `ledger-app/index.html` 파일을 열고, 우클릭 → **Open with Live Server** 선택
   (또는 하단 상태바의 "Go Live" 버튼 클릭).
3. 브라우저가 자동으로 열리며 `http://127.0.0.1:5500/ledger-app/` 형태의 주소로 접속됩니다.

`file://`로 직접 열어도 UI 자체는 동작하지만, 브라우저 보안 정책상 Firestore 요청이 막힐 수 있으니
Live Server(로컬 서버) 사용을 권장합니다.

## Firebase 설정

기존 `family-meeting-log` Firebase 프로젝트를 재사용하도록 `index.html`에 이미 설정되어 있습니다.
Firestore 컬렉션 이름은 `ledger_v1`로 분리되어 있어 회의록 앱의 기존 데이터와 겹치지 않습니다.

```js
const firebaseConfig = {
  apiKey: "AIzaSyB-ShWlxTiyT_hUZNN0U7Oss9j3r4ak5ek",
  authDomain: "family-meeting-log.firebaseapp.com",
  projectId: "family-meeting-log"
};
```

(Firestore만 사용하므로 `storageBucket`/`messagingSenderId`/`appId`는 필요하지 않아 생략했습니다.)

Firestore 보안 규칙은 테스트 단계이므로 아래처럼 테스트 모드(누구나 읽기/쓰기 가능)로 열어두면 됩니다.
(운영 배포 전에는 반드시 규칙을 강화해야 합니다.)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if true;
    }
  }
}
```

## 데이터 구조 (월별 문서 샤딩)

컬렉션: `ledger_v1`
문서 ID: `{ledgerId}_{YYYY-MM}` — 가계부(`shared`/`jihyeon`/`hyeonje`) × 월(月) 조합마다 문서 하나.
예: `shared_2026-08`, `jihyeon_2026-08`, `hyeonje_2026-09`

한 문서에는 해당 가계부의 **그 달**에 등록된 내역만 들어갑니다. 달력/대시보드 화면은 각자 보고 있는
월에 맞는 문서를 실시간 구독(`onSnapshot`)하며, 두 화면이 서로 다른 달을 보고 있으면 두 개의
구독이 동시에 유지됩니다.

각 문서 구조:
```json
{
  "entries": [
    {
      "id": "uuid",
      "date": "2026-08-25",
      "type": "expense",
      "amount": 12000,
      "category": "food",
      "payer": "shared",
      "memo": "",
      "createdAt": 1756000000000
    }
  ]
}
```

새 내역을 저장할 때는 내역의 `date` 값에서 연-월을 뽑아 해당 월 문서에 `arrayUnion`으로 추가합니다.

## 기타

- 마지막으로 보고 있던 가계부는 `localStorage`(`ledger_selected` 키)에 저장되어, 앱을 다시 열면
  자동으로 복원됩니다.
- 카테고리 목록(지출 10개, 수입 5개)은 `index.html` 안 `CATEGORIES` 상수에서 자유롭게 수정할 수 있습니다.
