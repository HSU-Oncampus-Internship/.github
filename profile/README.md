<div align="center">
<h1>🔎 굿모니터링 (Good Monitoring)</h1>
 
**지자체 캐릭터 IP를 무단으로 쓴 상품을 온라인에서 찾아내고, 담당자의 검토부터 신고까지 이어 주는 모니터링 플랫폼**
 
2026-2 AI/클라우드 온캠퍼스 인턴십 · Hansung University · 2026.09.15 ~ 2026.11.17 (10주)
 
![Python](https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat-square&logo=mysql&logoColor=white)
![Apify](https://img.shields.io/badge/Apify-97D700?style=flat-square&logoColor=white)
![Gemini](https://img.shields.io/badge/Gemini_API-4285F4?style=flat-square&logo=googlegemini&logoColor=white)
![DINOv2](https://img.shields.io/badge/DINOv2-0467DF?style=flat-square&logoColor=white)
![FAISS](https://img.shields.io/badge/FAISS-0467DF?style=flat-square&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=flat-square&logo=react&logoColor=black)
![MUI](https://img.shields.io/badge/MUI-007FFF?style=flat-square&logo=mui&logoColor=white)
 
</div>

 
## 📌 Overview
 
공공 캐릭터는 지자체의 자산이지만, 온라인 마켓에는 승인 없이 만든 상품이 섞여 유통됩니다.
상품은 여러 플랫폼에 흩어져 있고, 판매자는 계정이나 상호를 바꿔 다시 나타나며, 신고하려면 근거를 일일이 모아야 합니다.
 
굿모니터링은 **상품 수집 → 의심 신호 탐지 → 이미지 유사도 → 담당자 검토 → 신고 지원**을 하나의 흐름으로 연결합니다.
시스템은 검토 후보와 근거를 정리하고, **위반 여부의 판단과 신고 제출은 지자체 담당자가 직접 합니다.**
 
| 대상 캐릭터 | 기관 |
| --- | --- |
| 꿈돌이 · 꿈씨패밀리 | 대전광역시 |
| 해치 | 서울특별시 |
| 고양고양이 | 고양시 |
 
<br/>

**Goals**
 
| | |
| --- | --- |
| 🛒 **상품 수집 자동화** | 여러 마켓의 캐릭터 상품과 판매자 사업자정보를 매일 수집 |
| 🚨 **의심 신호 탐지** | 승인 사업자 대조와 가격 이상 규칙으로 검토 대상을 사유와 함께 선별 |
| 🖼️ **이미지 유사도** | 공식 캐릭터 이미지와 상품 이미지의 유사도를 판단 근거로 제공 |
| 🏛️ **기관별 검토** | 지자체 담당자가 자기 기관의 상품만 보고 정품·침해 여부를 직접 판정 |
| 🧾 **신고 지원** | 검토가 끝난 상품의 URL·증거·근거를 모아 신고문 초안 작성 |
 
---
 
## 🏗️ Architecture
 
수집·탐지(AI-DATA), 이미지 유사도 AI, 백엔드, 프론트엔드의 **4개 구성요소**로 이루어져 있습니다.
 
```text
 [네이버 · 쿠팡 · 테무 · 알리익스프레스]
          │  수집 (Apify API / 수동 입력)
          ▼
 ┌─────────────────────┐      ┌─────────────────────┐
 │  AI-DATA            │      │  이미지 유사도 AI    │
 │  수집 · 탐지 · 신고  │      │  DINOv2 · FAISS     │
 │  MySQL (원본 DB)     │      │                     │
 └──────────┬──────────┘      └──────────┬──────────┘
            │ sync-ai-data (SELECT 전용)  │
            ▼                             ▼
 ┌──────────────────────────────────────────────────┐
 │  BE · Spring Boot 3.5 · Java 21 · MySQL          │
 │  인증 · 기관별 권한 · 검토 · 판매자 · 신고 API    │
 └──────────────────────┬───────────────────────────┘
                        │ REST API ({data, meta})
                        ▼
 ┌──────────────────────────────────────────────────┐
 │  FE · React · MUI                                │
 │  지자체 담당자 검토 화면 · 대시보드                │
 └──────────────────────────────────────────────────┘
```
 
- BE는 AI-DATA의 원본 DB에 **직접 쓰지 않습니다.** 별도의 SELECT 전용 계정으로 조회한 뒤 BE 전용 DB에 매핑해 저장합니다.
- 성공 응답은 `{data, meta}`, 오류는 `{error: {code, message, fields}, meta}` 형식으로 통일했습니다.
---
 
## ⚙️ Core Components
 
<details open>
<summary><b>수집 — 막히는 마켓을 우회하지 않고 수집 가능한 경로로 설계</b></summary>
<br/>

 | 채널 | 방식 | 비고 |
| --- | --- | --- |
| 쿠팡 | Apify로 검색 → 상세 조회 2단계 자동 수집 (매일 오전 11시) | 상호·대표자·사업자번호·통신판매업번호까지 수집 |
| 테무 · 알리익스프레스 | Apify 자동 수집 (매일 오전 11시) | 영문 검색어와 제목 필터로 캐릭터 상품만 저장. 판매자 사업자정보는 자동 수집 불가, 사람이 확인 |
| 네이버 스마트스토어 | 수동 입력 도구 | 자동 접근 불가. 스토어 사업자정보는 store_name으로 자동 연결 |
 
- **쿠팡**은 Selenium, requests, Scrapling, Playwright(headless), AdsPower + Playwright를 모두 시도했지만 Akamai Bot Manager에 차단되었습니다. 그래서 Apify 수집기를 API로 호출하는 방식으로 바꿨습니다.
- **네이버**는 자동 접근이 로그인 리다이렉트와 캡차에 막혔고, 쇼핑 검색 오픈API도 2026년 7월에 종료되어 수동 입력 도구(`product_manual_input.py`, `seller_manual_input.py`)를 만들었습니다.
- 테무·알리 가격은 USD라 원화 가격 규칙과 섞이지 않도록 DB `price`에는 넣지 않고 CSV에만 남깁니다.

</details>
<details open>
<summary><b>의심 신호 탐지 — 점수 하나가 아니라 규칙별 사유</b></summary>
<br/>

 **미승인 사업자** — 판매자가 그 캐릭터의 승인 사업자로 확인되지 않으면 검토 대상으로 올립니다.
 
| 상태 | 의미 |
| --- | --- |
| 무단 판매 확인 | 사업자 정보 확인 후 사람이 무단으로 기록한 판매자 |
| 승인 여부 미확인 | 승인도, 무단 확인도 없는 판매자 |
| 판매자 정보 없음 | 판매자 정보가 연결되지 않은 상품 |
 
**가격 이상** — 규칙별로 사유를 그대로 기록합니다.
 
| 규칙 | 내용 |
| --- | --- |
| A | 같은 사업자가 플랫폼별로 같은 상품을 다른 가격에 파는 경우 |
| B | 품목군 내 통계적 이상치 (참고용) |
| C | 공식 정가 대비 이상 (정가표가 있을 때만 동작) |
| D | 승인된 공식 판매처의 같은 상품과 가격 비교 |
 
묶음 상품은 개당 가격으로 환산하고, 일반명(예: "인형")만 겹치는 상품이 서로 매칭되지 않도록 막았습니다.
 
</details>
<details open>
<summary><b>판매자 추적 · 증거 · 신고 지원</b></summary>
<br/>

- **판매자 추적:** 사업자등록번호가 같으면 플랫폼이 달라도 같은 판매자로 묶고, 상호가 바뀌어도 이전 상호를 남겨 다시 나타난 판매자를 이어서 봅니다.
- **증거:** URL, 확보 시각, 스크린샷을 SHA-256 해시와 함께 저장해 변조 여부를 확인할 수 있습니다.
- **신고문:** Gemini API로 초안을 쓰고, 상품 URL·상품명·증거 해시가 그대로 들어갔는지 검증합니다. 실패하면 템플릿 문구로 대체합니다.
- **접수 상태:** DRAFT → SUBMITTED → PROCESSING → DONE / REJECTED로 기록합니다.
</details>
<details open>
<summary><b>이미지 유사도 AI</b></summary>
<br/>

- 사전학습 모델 **DINOv2**로 공식 캐릭터 이미지와 상품 이미지의 임베딩을 추출합니다.
- **FAISS**로 공식 이미지와 가까운 상품 이미지를 검색합니다.
- 결과 점수는 BE를 통해 검토 화면에 판단 근거로 표시됩니다.

</details>
<details open>
<summary><b>백엔드 · 프론트엔드</b></summary>
<br/>

- **BE:** Java 21 · Spring Boot 3.5 · Spring JDBC · MySQL 8. API 명세 38개의 경로·요청·응답·오류·권한·인증 계약을 지킵니다. 세션 쿠키, CSRF 토큰, `Idempotency-Key`로 변경 요청을 보호하고 기관(대전·고양·서울)별로 접근 범위를 나눕니다.
- **FE:** React · MUI. 로그인 응답의 `csrfToken`과 `withCredentials: true`로 BE 세션 인증을 사용합니다. 담당자는 의심 상품의 URL과 근거(판매자, 가격, 이미지 유사도)를 보고 정품·침해 여부를 판정합니다.
- Docker 기반 데모 DB 스크립트로 팀원 모두가 같은 초기 데이터(꿈돌이 46건, 고양고양이 24건, 3개 기관 계정)를 각자 PC에 만들 수 있습니다.

</details>
---
 
## ✨ Key Features
 
| | 기능 | 설명 |
| --- | --- | --- |
| ① | **일일 자동 수집** | 쿠팡·테무·알리익스프레스를 매일 오전 11시에 수집하고 이어서 탐지까지 실행 |
| ② | **미승인 사업자 탐지** | 캐릭터별 승인 사업자 명단과 대조해 상태를 3단계로 분류 |
| ③ | **가격 이상 탐지** | 4개 규칙(A~D)으로 검토 대상을 선별하고 사유를 함께 기록 |
| ④ | **이미지 유사도** | DINOv2 임베딩과 FAISS 검색으로 공식 이미지와 가까운 상품을 근거로 제공 |
| ⑤ | **기관별 검토** | 담당자는 자기 기관의 상품만 조회하고 직접 판정 |
| ⑥ | **판매자 추적** | 사업자등록번호로 플랫폼이 달라도 같은 판매자를 묶고 상호 변경 이력 보존 |
| ⑦ | **증거 확보** | URL·확보 시각·스크린샷을 SHA-256 해시와 함께 저장 |
| ⑧ | **신고문 초안** | Gemini 초안 + 사실 검증, 실패 시 템플릿 문구로 대체 |
 
---
 
## 🚀 Engineering Highlights
 
**수집 · 탐지**
- 직접 접근이 차단되는 쿠팡을 **Apify 기반 2단계 수집**으로 전환하고, `--detail-new-only`로 새 상품만 상세 수집해 비용을 줄였습니다.
- 수집 Actor가 오류만 돌려줄 때도 정상 종료되던 문제를 **종료 코드와 재시도, 단계별 로그**로 드러냈습니다. 네트워크 대기와 실패 시 1회 재시도를 포함해 일일 자동 실행(`run_daily.sh`, launchd)의 안정성을 높였습니다.
- 위험 점수 하나로 단정하지 않고 **규칙별 사유를 그대로 보여 주어**, 담당자가 근거를 보고 판단하게 했습니다.
**증거 · 신고**
- 증거에 **SHA-256 해시**를 저장해 파일이 변경되지 않았음을 확인할 수 있습니다.
- LLM이 쓴 신고문이라도 **상품 URL·상품명·증거 해시가 그대로 들어갔는지 검증**하고, 실패하면 템플릿으로 대체해 잘못된 사실이 신고문에 들어가지 않게 했습니다.
**백엔드**
- AI-DATA 원본에는 **SELECT 전용 계정**으로만 접근하고 BE 전용 DB에 매핑해, 데이터 파이프라인과 서비스 DB를 분리했습니다.
- 기관별 접근 범위, 세션·CSRF·멱등 키를 갖춘 **38개 API 계약**을 일관된 응답 형식으로 제공합니다.

---
 
## 🛠️ Tech Stack
 
| 구분 | 기술 |
| --- | --- |
| **수집 · 탐지** | Python · MySQL · Apify · launchd |
| **신고 지원** | Gemini API |
| **이미지 유사도** | DINOv2 · FAISS |
| **Backend** | Java 21 · Spring Boot 3.5 · Spring JDBC · MySQL 8 · Docker |
| **Frontend** | React · MUI |
| **협업** | GitHub (이슈 · PR) |
 
---
 
## 📦 저장소
 
| 저장소 | 내용 |
| --- | --- |
| [AI-DATA](https://github.com/HSU-Oncampus-Internship/AI-DATA) | 상품 수집(쿠팡·테무·알리·네이버), 의심 신호 탐지, 판매자 추적, 증거·신고 지원 |
| [AI-Similarity](https://github.com/HSU-Oncampus-Internship/AI-SIMILARITY) | 이미지 유사도 |
| [BE](https://github.com/HSU-Oncampus-Internship/BE) | 인증, 기관별 권한, 검토·판매자·신고 API |
| [FE](https://github.com/HSU-Oncampus-Internship/FE) | 담당자 검토 화면, 대시보드 |
 
자세한 실행 방법은 각 저장소의 README를 참고하세요.
 
---
 
## ⚠️ 한계와 향후 과제
 
- **기준 데이터:** 기관의 실제 승인 사업자 명단과 공식 정가표는 기간 내에 받을 수 없어 예시 데이터로 동작을 검증했습니다. 실제 기준이 들어오기 전까지 결과는 신고 근거가 아니라 **검토 후보**입니다.
- **판매자 정보:** 테무·알리익스프레스는 판매자 사업자정보를 자동으로 수집하지 못해 사람이 확인해야 합니다.
- **네이버:** 자동 수집이 막혀 있어 수동 입력에 의존합니다.
- **측정하지 못한 항목:** 현재 판매 중인 상품의 비율(링크 재확인이 플랫폼에 차단됨)과 검토에 걸리는 시간(여러 담당자를 대상으로 조사하지 못함)은 측정하지 못했습니다.
- **스크린샷:** 자동 촬영은 보류했으며, 사람이 캡처해 등록합니다.
- **향후:** 기관 실제 명단·정가표 반영, 실제 담당자 사용성 조사, 서버·클라우드 스케줄러로 자동 수집 이전을 과제로 남깁니다.
---
 
## 👥 Team
 
| 이름 | 역할 | 주요 담당 |
| --- | --- | --- |
| [양다원](https://github.com/Dawon-Y) | 팀장 · AI/Data | 상품 수집(쿠팡·테무·알리·네이버), 의심 신호 탐지, 판매자 추적, 증거·신고 지원 |
| [문선우](https://github.com/moonsun01) | AI/Data | 이미지 유사도 AI |
| [정민지](https://github.com/mint0326) | Backend | Spring Boot API, DB 설계 |
| [송지원](https://github.com/songjiwon17) | Frontend | 검토 화면, 대시보드 |
 
---
 
## 🤝 협업 규칙
 
- 브랜치: `develop` → `feature/{이슈번호}-{작업내용}`에서 작업합니다.
- 커밋은 `type: 설명` 형식(`feat`, `fix`, `docs`, `refactor`, `chore`)을 따릅니다.
- PR은 승인을 받은 뒤 Squash merge하며 자동 병합하지 않습니다.
- `.env`, API 토큰, DB 비밀번호는 커밋하지 않습니다.
<br/>
<div align="center">
<sub>2026-2 AI/클라우드 온캠퍼스 인턴십 · Hansung University</sub>
</div>
 
