# Datastone

연구 PDF는 **[연구 자료실](research/README.md)**에서 찾습니다. lab에서 가져온 도표·학회 작성 양식·설계 명세를 종류별로 정리하고 원본 위치와 해시를 기록했습니다.

자료 정리 담당은 **[control-center의 ADA-N·PAP-O 에이전트](../control-center/agents/README.md)**입니다. [library 팀](../control-center/team/library/team.md)에 편성하며, 에이전트 설정·지침은 control-center에서만 관리합니다. 캐릭터 원전·카드·자료는 datastone에 유지합니다.

캐릭터 이미지는 **[캐릭터 이미지 보기](catalog/character-images.md)**에서 이미지와 함께 찾을 수 있습니다. 실제 파일은 `ip/series/<분류>/characters/<코드>/`에 있습니다.

뉴스·IP·전자상거래 데이터와 발행 자료를 관리하는 중앙 저장소입니다. 최소 구조로 시작하고 실제 자료와 작업에 맞춰 확장합니다.

- 새 자료는 `news/`, `ip/`, `commerce/`에 분야별로 보관합니다.
- 발행 구성·사용 자료 기록·최종 발행본은 `publisher/`에 보관합니다.
- 통합 목록은 `catalog/`, 관리 규칙은 `docs/`, 데이터 관리 도구는 `tools/`에 둡니다.
- 최소 구조, 전체 확장 레이아웃, 확장 기준은 [데이터 폴더 관리](docs/data-layout.md)를 따릅니다.
- 사용하지 않던 중복 계층·빈 IP 폴더는 정리했습니다. 삭제한 README의 설명은 [폴더 정리 기록](docs/folder-cleanup-2026-10-27.md)에 보존했습니다.
- 이전한 IP 자료는 [통합 색인](catalog/ip-index.md)과 [이전 내역](docs/ip-migration-2026-10-19.md)에서 확인합니다.
- 서버 코드와 세션·캐시 등 실행 데이터는 별도 서버 프로젝트에서 관리합니다.

기존 뉴스 수집 프로그램과 데이터는 아직 새 경로로 이전하지 않았습니다. 아래는 기존 프로그램의 설명이며, 기존 경로를 사용하는 코드·설정은 그대로 유지합니다.

## 기존 Eukk Newsletter MVP

한국 시장의 뉴스 흐름과 심리를 축적하기 위한 첫 번째 Eukk-commerce 구성 요소입니다. 실시간 알림은 Rolling Z-score로 결정하고, LightRAG는 뉴스가 발송된 뒤 비동기로 지식 그래프에 등록합니다. GraphRAG는 7일·30일 원본 스냅샷을 대상으로 주간 복기와 월간 리딩에 사용합니다.

## 현재 구현 범위

- RSS/Atom 수집
- 추적 파라미터를 제거한 URL 중복 방지
- 섹터·테마·긴급성·신규성·출처 신뢰도 원시 점수
- 섹터별 Rolling Z-score와 초기 `warming` 구간
- `urgent`, `standard`, `digest`, `archive` 분류
- Telegram Bot API 발송
- Obsidian Markdown 원본 저장
- LightRAG `/documents/text` 비동기 등록 요청 및 상태 저장
- GraphRAG용 7일/30일 원본 스냅샷
- 주간·월간 복기 프롬프트
- LeafLetter 개인 키워드·필터·피드백·RSS 캐시 마이그레이션
- LeafLetter 11개 섹터·10개 테마·3개 사건 분류체계
- LeafLetter 절대 관련도 점수를 섹터별 Rolling Z-score로 표준화
- 동일 사건 클러스터와 복수 출처 확인 점수

현재 단계에서는 기사 본문 크롤링, 의미 기반 사건 클러스터링, 가격 데이터 결합, 스케줄러 상시 실행은 포함하지 않습니다.

## 설계 원칙

```text
news/seeddata(원시) → news/leafdata(전처리) → news/fruitdata(정제)
                                         ├→ golden (S)
                                         ├→ prime (A)
                                         ├→ ripe (B)
                                         ├→ unripe (C)
                                         └→ rotten (SHADOW)

RSS → 정규화 → 스코어링 → SQLite/Obsidian
                    ├→ Telegram 즉시 발송
                    └→ LightRAG 등록

SQLite 원본 → 7일/30일 스냅샷 → GraphRAG → 주·월간 복기
```

`fruitdata` 등급은 종합 Z-score 임계값에 따라 결정합니다. 차단 데이터와
최저 임계값 미만 데이터는 `rotten`으로 격리합니다. 기존 발송 밴드는 이전
설정과의 호환을 위해 유지합니다.

RAG가 생성한 문서는 원본 뉴스량이나 Z-score에 다시 포함하지 않습니다. 생성물이 자기 자신을 근거로 반복 증폭하는 피드백 루프를 막기 위해서입니다.

## 시작하기

Python 3.12 이상이 필요하며 MVP 자체는 외부 Python 패키지를 요구하지 않습니다.

```powershell
Copy-Item config.example.toml config.toml
$env:PYTHONPATH = "src"
python -m eukk_newsletter.cli --config config.toml init-db
python -m eukk_newsletter.cli --config config.toml run-once --dry-run
```

`--dry-run`은 Telegram과 LightRAG 호출을 생략하지만 수집 결과는 SQLite와 Obsidian에 저장합니다. 실제 RSS 주소를 `config.toml`의 `[[sources]]`에 추가해야 합니다.

## Telegram 설정

`config.toml`에서 `telegram.enabled = true`로 바꾸고 현재 셸에 비밀값을 설정합니다. 토큰은 설정 파일이나 Markdown에 저장하지 않습니다.

```powershell
$env:TELEGRAM_BOT_TOKEN = "..."
$env:TELEGRAM_CHAT_ID = "..."
python -m eukk_newsletter.cli --config config.toml run-once
```

기본값은 `urgent`와 `standard`만 즉시 발송합니다. `digest`는 이후 뉴스레터 생성 후보입니다.
첫 수집 때 오래된 기사가 쏟아지는 것을 막기 위해 기본적으로 발행 후 6시간이 지난 기사는 발송하지 않고 저장만 합니다.

## LeafLetter 가져오기

기존 DB의 채팅 ID는 복사하지 않고 개인 키워드, 최신 점수 설정, 피드백, 과거 수집 제목만 단일 개인 프로필로 가져옵니다. 기존 RSS 53개는 안전을 위해 비활성 상태로 등록됩니다.

```powershell
$env:PYTHONPATH = "src"
python -m eukk_newsletter.cli --config config.toml import-leafletter `
  --source-root "C:\Discovery\Land\Leaf-letter" `
  --source-db "C:\Discovery\Land\News-Leafletter\data\news_leafletter.db"

python -m eukk_newsletter.cli --config config.toml list-sources
python -m eukk_newsletter.cli --config config.toml set-source `
  --match coindesk --enable --sector FINANCIALS --reliability 0.75
```

기존 점수는 다음 순서로 결합됩니다.

```text
Leaf 관련도 = 속보·단독 + 개인 키워드 + 만료형 단기 테마 + 섹터 선호
Leaf 관련도 Z = 같은 섹터 내 최근 분포로 표준화
최종 Z = Leaf 관련도 Z + 섹터/테마 속도 + 긴급성 + 신규성 + 출처 신뢰도 + 복수 출처 확인
```

## LightRAG 설정

LightRAG Server를 별도 실행한 뒤 `lightrag.enabled = true`로 변경합니다. API 키를 사용하는 경우 다음 환경변수를 설정합니다.

```powershell
$env:LIGHTRAG_API_KEY = "..."
```

발송은 LightRAG 처리 완료를 기다리지 않습니다. 등록 실패는 `articles.lightrag_status`와 `lightrag_error`에 기록됩니다.

## GraphRAG 주·월간 입력 생성

```powershell
python -m eukk_newsletter.cli --config config.toml export-graphrag --days 7
python -m eukk_newsletter.cli --config config.toml export-graphrag --days 30
```

각 출력 폴더에는 원본 뉴스별 `.txt`와 출처 추적용 `manifest.json`이 생성됩니다. 분석 지침은 `prompts/graphrag_weekly.md`와 `prompts/graphrag_monthly.md`에 있습니다.

## 테스트

```powershell
$env:PYTHONPATH = "src"
python -m unittest discover -s tests -v
```

## 다음 구현 순서

1. 실제 RSS 목록과 기존 Z-score/섹터/단기 테마 스코어 이식
2. 동일 사건을 묶는 클러스터 ID 및 복수 출처 확인 점수
3. Telegram 발송 큐, 속도 제한, 재시도 워커
4. LightRAG 실패 문서 재처리 워커
5. 주·월간 GraphRAG 실행 및 결과를 `vault/generated/`에 저장
6. 사용자 클릭·저장·무시 피드백으로 임계값 보정

첫 실제 워밍업 결과는 [`docs/warmup-2026-08-11.md`](docs/warmup-2026-08-11.md)에 기록되어 있습니다.
# datastone
