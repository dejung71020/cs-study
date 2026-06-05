# ERP · SAP · SI 용어 정리

## 내가 공부 전 아는 상식
* SAP는 기업에서 많이 쓰는 소프트웨어라고 들었다.
* ERP가 회사의 데이터를 중앙 관리하는 것이라고 들었다.
* SI는 솔루션 관련이라고는 하는데 정확한 이해를 하지 못 했다.
* 이 용어들이 취업 공고에 자주 등장하는데 정확한 차이를 모른다.

## 핵심 개념

| 용어 | 풀네임 | 한 줄 설명 |
|---|---|---|
| ERP | Enterprise Resource Planning | 기업 전체 자원(회계·HR·물류·생산)을 하나의 시스템으로 통합 |
| SAP | Systems, Applications & Products | 세계 1위 ERP 소프트웨어 벤더 (독일 기업) |
| SI | System Integration | 여러 시스템을 고객 요구에 맞게 연결·구축하는 사업 |
| SM | System Maintenance | 구축된 시스템의 운영·유지보수 |
| ISP | Information Strategy Planning | IT 도입 전 전략·로드맵 수립 컨설팅 |

## ERP

기업의 모든 부서를 하나의 DB로 연결해 실시간으로 데이터를 공유하는 시스템.

```
[구매] ─┐
[생산] ─┤
[물류] ─┼─→ ERP (단일 DB) → 경영진 대시보드
[회계] ─┤
[HR]  ─┘
```

- **도입 전**: 부서마다 엑셀/별도 시스템 → 데이터 불일치, 수작업 많음
- **도입 후**: 구매 발주 → 재고 자동 반영 → 회계 자동 처리

**주요 ERP 벤더:**

| 벤더 | 제품 | 특징 |
|---|---|---|
| SAP | S/4HANA | 글로벌 대기업 표준, ABAP 언어 |
| Oracle | Oracle ERP Cloud | DB 강점, 클라우드 전환 중 |
| Microsoft | Dynamics 365 | 중소기업 타겟, Office 연동 |
| 더존비즈온 | WEHAGO | 국내 중소기업 특화 |

## SAP

독일 기업이 만든 ERP 소프트웨어. 전 세계 Fortune 500의 87%가 사용.

- **SAP R/3**: 기존 온프레미스 버전
- **SAP S/4HANA**: 최신 클라우드 버전 (인메모리 DB HANA 사용)
- **ABAP**: SAP 전용 프로그래밍 언어 (SAP 개발자가 커스터마이징에 사용)
- **SAP Module**: FI(재무), CO(관리회계), MM(자재관리), SD(영업), PP(생산) 등 모듈별 구성

```
SAP 시스템
├── FI (Financial Accounting) - 재무회계
├── CO (Controlling) - 관리회계
├── MM (Materials Management) - 구매·재고
├── SD (Sales & Distribution) - 영업·판매
└── PP (Production Planning) - 생산계획
```

## SI (System Integration)

고객사의 요구에 맞게 여러 시스템을 설계·구축·통합하는 사업.

**SI 프로젝트 흐름:**
```
ISP(전략수립) → 분석/설계 → 개발 → 테스트 → 이행(Go-Live) → SM(유지보수)
```

**국내 주요 SI 업체:** 삼성SDS, LG CNS, SK C&C, 롯데정보통신, 포스코ICT

**SI vs SM:**
- SI: 새 시스템 구축 (프로젝트성, 기간 한정)
- SM: 기존 시스템 운영·유지보수 (상시, 장기 계약)

## 관련 용어

| 용어 | 설명 |
|---|---|
| SCM | Supply Chain Management — 공급망(조달~납품) 관리 |
| CRM | Customer Relationship Management — 고객 데이터 관리·마케팅 |
| MES | Manufacturing Execution System — 공장 생산 실행 시스템 |
| Legacy | 오래된 기존 시스템. SI 프로젝트의 주요 대상 |
| BPR | Business Process Reengineering — ERP 도입 전 업무 프로세스 재설계 |
| API 연동 | 서로 다른 시스템(ERP·CRM·MES 등)을 REST API로 연결하는 SI 핵심 작업 |

## 면접 한 줄 답변
> "ERP는 기업 자원 전체를 하나의 시스템으로 통합·관리하는 플랫폼이고, SAP는 그 ERP 소프트웨어의 세계 1위 벤더이며, SI는 고객사 요구에 맞게 이런 시스템들을 설계·구축·연결하는 사업입니다."

## 프로젝트 적용
- 에스프로넥스트 지원: 'ERP & AI 비즈니스 컨설팅' 직무 — ERP 시스템에 AI를 연동하는 업무. WorkB에서 Slack·Jira·Google Calendar 세 시스템을 API로 통합한 경험이 SI의 API 연동 작업과 구조적으로 동일함.
- 팡팡팡: 기상청 API·카카오맵 API·Gemini API를 하나의 파이프라인으로 통합한 것이 소규모 SI 개념에 해당.
