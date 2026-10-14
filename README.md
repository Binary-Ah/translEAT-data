# TranslEAT — Cross-cultural Rating Gap Data Pipeline

같은 LA 식당에 대한 **한국어 리뷰와 영어 리뷰의 Google 평점 차이**를 UCSD Google Local 데이터셋에서 집계·검정한 데이터 파이프라인입니다.
USC IMSC Hackathon(2026.07.25)에서 팀 La La LAN이 만든 Chrome 확장 **TranslEAT**의 데이터 측 작업입니다.

> **메인 앱(Chrome 확장 + FastAPI 백엔드 + 크롤러)은 팀 레포 [14EE4/gmaps-culturate-extension](https://github.com/14EE4/gmaps-culturate-extension)에 있습니다.**
> 이 저장소는 이진아가 작성한 데이터 전처리 스크립트와 분석 결과 요약만 담습니다.

- 프로그램: 숭실대 2026 글로벌SW교육(SPP 5기) — USC Viterbi IMSC AI·Data Science 수업 수료(2026.07.24) + IMSC 해커톤 참가(2026.07.25)
- 팀: La La LAN (최종 발표 Speaker: 이진아)

## Problem

Google Maps 평점은 모든 언어의 리뷰를 합산한다. 한국인 사용자가 미국 식당을 고를 때 "한국어 리뷰어들은 이 집을 어떻게 봤는가"는 평균에 묻힌다.
같은 식당 안에서 한국어 vs 영어 리뷰의 평점 격차가 실제로 존재하는지, 업종별로 다른지, 시간에 따라 변하는지를 통계적으로 검증하고
그 결과를 확장 프로그램이 참조할 데이터(`extension_data.json`)로 만들었다.

## Data

| 항목 | 값 |
|---|---|
| 소스 | UCSD Google Local Reviews (California) — `meta-California.json.gz`, `review-California.json.gz` (원본 리뷰 약 7천만 건, 2021년까지) |
| 필터 | LA 위경도 범위(lat 33.7~34.35, lon −118.7~−118.1) + 음식점 카테고리 키워드 + 최소 리뷰 수(기본 50) |
| 결과 규모 | LA 음식점 17,090곳, 리뷰 약 3.5M건 |
| 한국어 리뷰 | 처음 1,797건 → **17,622건** (약 10배). 원인: 데이터의 `text` 필드에 원문과 Google 번역본이 함께 들어 있어 언어 감지가 영어로 판정하던 것을 번역 필드 분리로 복구 |
| 확장 참조 데이터 | 실측 장소 388곳(Tier 1) + 업종 42개 보정값(Tier 2), 검색 인덱스 17,090개, baseline_gap −0.2097 |

## Method

1. `01_filter_la_places.py` — 메타데이터를 gzip 스트리밍으로 한 줄씩 읽어 LA bbox·음식점 카테고리·최소 리뷰 수 조건으로 필터, `gmap_id` 목록 저장.
2. `02_extract_reviews.py` — 7천만 건 리뷰 파일을 스트리밍하며 1단계 `gmap_id`에 해당하는 리뷰만 추출(빈 텍스트 제외).
3. 언어 감지 + 번역 필드 분리 → 한국어/영어 리뷰 분리
4. 집계·검정:
   - 장소별 한국어/영어 평균 평점과 격차(gap, rel_gap)
   - **동일 가게 내(within-place) 순열 검정** 1,000회 — 식당 자체의 품질 차이를 통제
   - 업종별 격차에 **FDR 보정**
   - 리뷰 텍스트에서 맛/서비스 언급 비율, 연도별 격차 추세
5. 확장 프로그램용 3-tier 데이터: 장소 실측값(Tier 1, 388곳) → 업종 보정값(Tier 2, 13,515곳) → N/A.

## Results

| 분석 | 수치 |
|---|---|
| 전체 평균 평점 | 한국어 4.00 vs 영어 4.21 (격차 0.21) |
| 예시: BCD Tofu House | 격차 0.36 |
| within-place 순열 검정 (1,000회) | 실제 격차 0.404, p = 0.001 |
| FDR 보정 후 유의한 업종 | 21개 (격차 −0.515 ~ +0.134, 아시안 음식 카테고리에 집중) |
| 리뷰에서 맛 언급 비율 | 한국어 46.5% vs 영어 19.7% |
| 리뷰에서 서비스 언급 비율 | 한국어 30.6% vs 영어 18.6% |
| 커버리지 | 81.4% (Tier 1 388곳 2.3% + Tier 2 13,515곳 79.1%) |
| 연도별 추세 (2016–2021) | 격차가 연 0.021점씩 확대, p = 0.0157 |

## My role

- 데이터 전처리 스크립트(`01_filter_la_places.py`, `02_extract_reviews.py`) 작성 — 7천만 건 리뷰를 메모리에 올리지 않고 스트리밍 필터.
- 번역 필드가 섞여 한국어 리뷰가 10분의 1만 잡히던 파싱 문제를 발견하고 분리해 17,622건으로 복구.
- 동일 가게 내 순열 검정 + FDR 보정 분석 설계 (결과보고서 기준).
- 최종 발표 Speaker.
- 확장 프로그램(MV3, Shadow DOM UI), FastAPI 백엔드, Selenium 크롤러는 팀원 작업입니다(팀 레포 14EE4/gmaps-culturate-extension).

## How to run

```bash
# UCSD Google Local 데이터(California) 다운로드 후 data/raw/에 배치 (저장소에 포함하지 않음)
python scripts/01_filter_la_places.py \
    --meta-path data/raw/meta-California.json.gz \
    --out-path data/processed/la_restaurants_meta.jsonl \
    --min-reviews 50
# → data/processed/la_restaurant_gmap_ids.txt 도 생성

python scripts/02_extract_reviews.py \
    --review-path data/raw/review-California.json.gz \
    --gmap-ids-path data/processed/la_restaurant_gmap_ids.txt \
    --out-path data/processed/la_restaurant_reviews.jsonl
```


## Files

```
README.md
.gitignore
scripts/
  01_filter_la_places.py   # 메타 스트리밍 필터 (LA bbox · 음식점 · 최소 리뷰 수)
  02_extract_reviews.py    # 리뷰 스트리밍 추출 (gmap_id 매칭, 빈 텍스트 제외)
```
