# USD 파이프라인

inFX 제작 파이프라인의 USD(OpenUSD) 전환을 위한 자료조사, 준비, 계획 정리 문서

- 상태: 초안 (3장 USD 자료조사 작성 완료, 그 외 섹션 진행 중)
- 최종 수정: 2026-09-08

---

## 0. 문서 작성 지침

### 0.1 문체

- 개조식 문체 + 명사형 종결 문체로만 작성
- 한 문장에 여러 정보를 담지 않음. 한 문장 = 한 사실
- 처음 등장하는 개념은 결론 먼저 제시, 그 뒤에 근거·비유로 단계적 보강

### 0.2 미비 항목 검토 방식

- 개발 착수 전, 문서 내 "(작성 예정)"·"미결정" 항목을 하나씩 순서대로 검토
- 항목마다 아래 두 가지 설명을 함께 제공한 뒤 결정
  - 기술 설명: 단순 기술 관점의 설명
  - 쉬운 설명: 비유·예시를 포함한 알아듣기 쉬운 설명
- 결정 근거가 어렵지 않도록 선택지와 권장안을 함께 제시
- 결정된 내용은 해당 절에 반영하고, 6장 미결정 사항에서 제거

## 1. 목표와 범위

### 1.1 목표

- 에셋 퍼블리시 표준화: 모델·룩뎁·리깅 결과물을 USD 레이어로 저장
- 씬 조립 협업 개선: 부서별 레이어를 겹쳐 샷을 구성, 대기·재전달·재작업 구간 축소
- 라이팅 워크플로우 개선: 앞 단계 레이어를 실시간 참조하며 라이팅 세팅 선행 가능

### 1.2 설계 범위와 구현 범위의 분리

- 설계: USD로 대체 가능한 **모든 구간**을 대상으로 전체 그림을 그림
- 구현: 상황에 맞게 구간별로 나눠 단계적으로 진행 (→ [5.2 단계별 로드맵](#52-단계별-로드맵-권장-순서))
- 이유: 구간별로 따로 설계하면 경로·레이어 구조가 구간마다 어긋남. 전체를 보고 설계해야 나중에 붙일 때 재설계가 없음

### 1.3 설계 대상 구간

| 구간 | 내용 | 구현 순서 |
| --- | --- | --- |
| Asset 퍼블리시 | 모델·룩뎁·리깅 결과물의 USD 저장, 애셋 진입점 구성 | 1순위 |
| Shot 레이아웃 조립 | 애셋 진입점을 Reference로 모아 샷 구성, 카메라·레이아웃 레이어 | 2순위 |
| Animation | 애니메이션 결과를 레이어로 퍼블리시 (캐시는 .abc 유지, USD가 참조) | 3순위 |
| FX | 시뮬 결과를 레이어로 퍼블리시 | 3순위 |
| Lighting | 앞 단계 레이어를 참조해 라이트·머티리얼 오버라이드 레이어 구성 | 4순위 |

### 1.4 제외 범위

- 렌더링 파이프라인 교체 (Hydra 렌더 딜리게이트로 최종 렌더 수행)
- 기존 렌더러·렌더팜 연동 방식은 유지
- 이유: 렌더러 연동은 리스크가 가장 크고, 라이팅까지 USD로 구성돼도 최종 렌더는 기존 방식으로 가능

## 2. 현행 파이프라인 현황

- 조사 기준: `flova` 저장소 코드·템플릿 (2026-09-08)

### 2.1 사용 중인 DCC와 버전

| DCC | 버전 | 용도 | USD 지원 여부 |
| --- | --- | --- | --- |
| Maya | 2024 (전 사이트 설치 완료), 2022 코드 잔존 | 모델·룩뎁·리깅·레이아웃·애니메이션·매치무브·라이팅 | mayaUsd 지원 (2023+) |
| Houdini | 20.5 (일부 19.5), Python 3.11.7, USD 24.03 | 라이팅, FX | Solaris 네이티브 지원 |
| Nuke | 14.1 주력, 15.x 일부 | 컴프, 매치무브 언디스토트, 프리컴프 | 15.0+ 기본 지원, 14.x 미지원 |
| 3DEqualizer | - | 매치무브 | 해당 없음 (카메라는 Maya 경유) |
| Unreal | - | 별도 연동 모듈 존재 | 네이티브 USD 지원 |
| Katana | 7.0v4, Python 3.10, USD 23.05 (`fnpxr`) | 라이팅·렌더 (Arnold) | 네이티브 USD 지원 |

### 2.2 데이터 교환 방식

- Maya 씬: `.mb` 주력, `.ma` 일부
- 지오메트리·애니메이션 캐시: `.abc` (Alembic)
- 카메라: `.abc` + `.fbx`
- 텍스처·셰이더: 버전 폴더 단위 파일 묶음
- 리뷰: `.mov`
- 메타데이터: `.json`
- USD: 현재 미사용

### 2.3 퍼블리시 도구와 ShotGrid 연동

- 퍼블리시 도구: `flova.maya.app.pub_tools` (Maya 내 실행)

| 도구 클래스 | 대상 스텝 | 주요 산출물 |
| --- | --- | --- |
| `AssetPubToolsWindow` | model, lookdev | Maya 씬, 모델 .abc 캐시, 셰이더, 텍스처 |
| `RiggingPubToolsWindow` | rig | Maya 씬, 리깅 캡처 이미지 |
| `EnvAssetPubToolsWindow` | env | Maya 씬, .fbx |
| `AnimationPubToolsWindow` | layout, animation | Maya 씬, 카메라, 애니메이션 .abc 캐시 |
| `MatchmovePubToolsWindow` | layout, matchmove | Maya·Nuke 씬, 카메라, 언디스토트, .mov |
| `EnvShotPubToolsWindow` | env, env_layout | Maya 씬, 캐시 |

- ShotGrid 연동 흐름
  - 작업 파일명에서 에셋/샷·태스크·버전을 역매핑 (`ASSET_FILENAME_REG`, `SHOT_FILENAME_REG`)
  - 퍼블리시 시 ShotGrid Version 생성·등록 (`create_registration_version`)
  - PublishedFile 생성 (`flova.shotgrid` 모듈)
  - 일부 작업은 Deadline 플러그인으로 위임 (`dl_publish_for_asset` 등)
- 오픈망·폐쇄망 모두 같은 ShotGrid 인스턴스 사용

### 2.4 경로 규칙

- 정의 위치: `flova/template/<PROJECT_CODE>.yaml`
- 프로젝트 루트: `%DRIVE%/show/%PROJECT_CODE%`
- 에셋: `%PROJECT_PATH%/assets/%ASSET_TYPE%/%ASSET_CODE%/%STEP_CODE%/{wip|pub}`
- 샷: `%PROJECT_PATH%/seq/%SEQUENCE_CODE%/%SHOT_CODE%/%STEP_CODE%/{wip|pub}`
- 파일명: `%ASSET_CODE%_%TASK_CODE%_%VERSION%`, `%SHOT_CODE%_%TASK_CODE%_%VERSION%`
- 버전: `v` + 3자리 (`v001`)
- 스텝 코드 실제 값: `model`, `lookdev`, `rig`, `env`, `layout`, `animation`, `matchmove`, `env_layout`, `lighting`, `comp` 등
- 주요 캐시 경로
  - 모델 캐시: `%ASSET_PATH%/model/pub/data/abc`
  - 애니메이션 캐시: `%SHOT_PUB_PATH%/cache/%VERSION%`
  - 매치무브·Env 캐시: `%SHOT_PUB_PATH%/cache/%VERSION%`
  - 셰이더·텍스처: `%ASSET_PUB_PATH%/{shader|tex}/%VERSION%`

### 2.5 USD 전환 관점의 시사점

- 경로 역매핑이 폴더 인덱스 기반 (`ASSET_STEP_CODE_INDEX: 6` 등) → `%ASSET_PATH%/usd` 같은 비스텝 폴더 추가 시 역매핑 로직 영향 검토 필요
- Nuke 14.1은 USD 미지원 → 컴프 단계에 USD를 넣으려면 Nuke 15+ 필요 (이번 범위 제외)
- Maya 2022 기준 코드 잔존 → Python 3.10 이관과 병행 (→ [6장](#6-미결정-사항))

## 3. USD 자료조사

### 3.0 USD란

* Pixar가 만듦
* 지금은 오픈소스
* 한마디로: 여러 사람이 같은 3D 씬을 동시에 작업하고, 하나로 합칠 수 있게 해주는 방식

표준화 현황

* 2023년, Pixar·Adobe·Apple·Autodesk·NVIDIA 등이 모여 **AOUSD**(Alliance for OpenUSD) 결성
* 2025년 12월, 정식 표준 1.0판 발표
* 2026년 현재도 계속 기능 추가 중
* ([발표 글](https://aousd.org/news/core-spec-announcement/), [2026년 근황](https://www.linuxfoundation.org/press/aousd_prmarch2026))

### 3.1 USD 특징

* 기존 방식: 애니메이터가 씬 생성. 라이팅 팀이 그 씬을 이어받음. 서로 파일을 주고받고 덮어씀
* USD 방식: 투명 필름(레이어) 여러 장을 겹쳐놓는 방식
  * 각자 자기 필름 한 장만 그림
  * 다 겹쳐서 보면 완성된 그림이 나옴
  * 한 사람이 자기 필름을 고쳐도, 다른 사람 필름은 그대로 남음

#### 오해하기 쉬운 부분

* "각자 알아서 작업" ≠ 순서가 없어진다는 뜻
* 모델링 → 리깅 → 애니메이션 → 라이팅 순서는 USD를 써도 그대로임
* USD가 바꾸는 건 순서가 아니라 **기다리는 방식**

기존(.abc) 방식

* 애니메이션이 다 끝나야 캐시를 구워서 넘김
* 라이팅팀은 그 파일을 받아야 시작 가능
* 애니메이션이 조금만 바뀌어도, 다시 굽고 다시 넘겨야 함

USD 방식

* 라이팅팀이 애니메이션팀의 작업 중인 레이어를 실시간으로 참조 가능
* 그래서 완성되기 전에 미리 세팅을 시작할 수 있음
* 애니메이션이 바뀌면 자동으로 반영됨. 다시 받을 필요 없음

한 줄 요약

* USD는 순서를 없애는 기술이 아님
* **기다림·재전달·재작업 비용을 줄이는 기술**
* ([관련 설명](https://openusd.org/release/intro.html))

### 3.2 핵심 개념

* **Prim**
  * 씬 안에 있는 물체 하나하나(캐릭터, 소품, 카메라, 조명 등)
  * 폴더처럼 그 안에 또 다른 물체를 담을 수도 있음
* **Layer**
  * "투명 필름 한 장"에 해당
  * 파일 하나 = 필름 한 장
* **Stage**
  * 필름을 전부 겹쳐서 완성한 최종 그림
  * 실제로 화면에 보이는 결과물
* **Composition Arc (합성 방식)**: 필름들을 겹치는 여러 가지 방법
  * **Reference**: 다른 파일의 물체를 여기에도 가져다 놓기 (같은 나무 모델을 숲 곳곳에 복사해서 놓는 것과 비슷)
  * **Payload**: 무거운 파일을 필요할 때만 불러오기 (평소엔 접어두고, 열어볼 때만 펼치는 것과 비슷)
  * **Variant**: 같은 물체의 여러 버전 중 하나 고르기 (같은 캐릭터의 옷 색깔을 A/B/C 중에서 고르는 것과 비슷)
* **필름 우선순위**
  * 여러 필름이 같은 부분을 다르게 그리면, 어느 걸 우선할지 정해진 순서가 있음
  * 최근 이름이 **LIVRPS → LIVERPS**로 변경 (새 규칙 하나 추가됨)
  * 세부 순서는 실제 작업 시 참고자료 확인 필요
  * ([관련 설명](https://docs.nvidia.com/learn-openusd/latest/creating-composition-arcs/strength-ordering/what-is-liverps.html))

### 3.3 관련 기술

* **UsdGeom / UsdShade / UsdSkel / UsdLux**
  * 각각 "모양", "재질", "뼈대·애니메이션", "조명"을 표현하는 규칙 모음
  * 도면 그릴 때 벽은 실선, 문은 점선으로 약속하는 것과 비슷
* **MaterialX**
  * 재질(질감, 색상, 반사 정도 등)을 표현하는 공통 언어
  * 어떤 렌더러를 쓰든 같은 재질 정의를 재사용 가능
* **Hydra**
  * USD로 만든 씬을 실제 화면에 그려주는 엔진
  * 일종의 "번역기" 역할
  * USD 데이터를 각 렌더러(Arnold, V-Ray 등)가 이해할 수 있는 형태로 변환
  * 덕분에 렌더러를 바꿔도 씬 데이터는 그대로 재사용 가능

### 3.4 각 프로그램(DCC)의 지원 상황

* **Maya**
  * `mayaUsd`라는 무료 플러그인으로 USD를 다룸
  * 2026년 8월 현재, **Maya 2023 이상만 공식 지원**
  * Maya 2022는 지원 대상에서 빠져 있음
  * ([참고](https://github.com/autodesk/maya-usd))
  * ✅ inFX는 전 사이트 Maya 2024로 업그레이드 및 설치 완료. mayaUsd 공식 지원 범위 안으로 들어옴
* **Houdini**
  * "Solaris"라는 화면(LOPs)에서 USD를 기본 언어처럼 다룸
  * 현재 USD를 가장 잘 지원하는 프로그램으로 평가받음
* **Nuke, Unreal, Blender**
  * 셋 다 USD 파일을 읽고 쓰는 기능을 갖추고 있음
  * 계속 기능 추가되는 중
* **명령어 도구**
  * `usdview`(USD 파일 미리보기), `usdchecker`(파일 규칙 검사), `usdcat`(내용을 텍스트로 확인) 등이 기본 제공

### 3.5 DCC별 USD 최소 버전 · Python 버전 요구사항

* USD를 쓰려면 DCC 자체 버전뿐 아니라, 그 DCC가 내부적으로 쓰는 Python 버전도 같이 맞아야 함
* Python 버전이 안 맞으면 USD 플러그인이 아예 설치되지 않거나, 스크립트가 깨질 수 있음

| DCC | USD 사용 가능 최소 버전 | 해당 버전의 Python | 비고 |
| --- | --- | --- | --- |
| **Maya** | 2023 (mayaUsd 공식 지원 시작) | 2023부터 Python 3.9 전용(Python 2 지원 종료) | Maya 2022는 Python 2.7 / 3.7.7 혼용 체제라 mayaUsd 공식 지원 대상 아님. Maya 2024는 Python 3.10, 2025~2026은 Python 3.11 계열 사용 ([참고](https://github.com/Autodesk/maya-usd/blob/dev/doc/build.md), [참고](https://help.autodesk.com/cloudhelp/2023/ENU/Maya-WhatsNewPR/files/GUID-DF43840B-4DB1-43F8-BFD1-97D8D031B91D.htm)) |
| **Houdini** | 18.0부터 USD(Solaris) 도입, 18.5~19부터 Python 3 빌드로 완전 전환 | 버전마다 다름(최근 버전은 Python 3.9~3.11 계열) | 현재 신규 배포판은 사실상 전부 Python 3 빌드. Solaris는 Houdini의 USD 작업 화면 이름 ([참고](https://www.sidefx.com/docs/houdini/solaris/usd.html)) |
| **Nuke** | 15.0부터 기본 USD 기능 제공, 16.0부터 정식 권장 | Nuke 15~16은 내장 Python 3 사용(버전은 릴리즈별 상이) | 이전 Nuke(14 이하)는 USD 기능 없음. 실무 적용은 16.0 이상 권장 ([참고](https://learn.foundry.com/nuke-stage/current/Content/stage_environment/loading_stage/usd_export_nuke.html)) |
| **Unreal Engine** | 4.24부터 USD 임포트 기능 도입(당시는 실험적), 4.27 이후 실무 수준으로 안정화 | 엔진 내장 Python 3(버전은 UE 릴리즈별 상이) | 5.4 이후는 Epic 자체 USD 플러그인 사용 권장(과거 NVIDIA Omniverse 커넥터 방식과 별개) ([참고](https://dev.epicgames.com/documentation/en-us/unreal-engine/universal-scene-description-usd-in-unreal-engine)) |
| **Blender** | 2.82부터 USD 익스포트 지원 시작(당시 실험적), 4.0부터 정식 기능으로 안정화 | 4.0은 Python 3.10, 4.1 이후는 Python 3.11 | 초기 버전은 익스포트 위주였고 임포트·핫업데이트 기능은 이후 버전에서 보강됨 ([참고](https://developer.blender.org/docs/release_notes/4.0/import_export/)) |

* inFX는 전 사이트 **Maya 2024로 업그레이드 및 설치 완료**. mayaUsd 공식 지원 범위(2023+) 안으로 들어옴
* Maya 2024는 **Python 3.10** 사용 → 기존 Python 3.7.7 기준으로 작성된 코드/모듈은 3.10 호환성 재검토 필요 (→ [4장](#4-전환-준비))

### 3.6 지금 쓰는 .abc(Alembic) 방식 vs USD 방식

* .abc(Alembic) 방식
  * 각 부서가 작업을 끝내면 완성된 사진을 찍어서 다음 부서에 넘기는 방식
  * 사진은 이미 구워진 결과물
  * 나중에 뭔가 고치려면 원본 작업 파일로 돌아가서 다시 찍어야 함
* USD 방식
  * 여러 사람이 같은 도면 위에서 각자 트레이싱지(반투명 종이) 한 장씩 겹쳐서 그리는 방식
  * 레이아웃팀은 1번 종이, 애니메이션팀은 2번 종이, 라이팅팀은 3번 종이
  * 다 겹쳐 보면 완성된 그림이 나옴
  * 누가 자기 종이를 고쳐도 다른 사람 종이는 그대로 남음

한눈에 비교

| 구분 | .abc(Alembic) 방식 | USD 방식 |
| --- | --- | --- |
| 결과물 | 이미 다 구워진 "완성 사진" | 겹쳐서 완성하는 "여러 장의 종이" |
| 수정할 때 | 원본에서 다시 찍어야 함(재작업) | 내 종이 한 장만 고치면 끝 |
| 다음 부서가 작업 시작하는 시점 | 앞 부서가 "완성 사진"을 넘겨줘야만 시작 가능 | 앞 부서가 "작업 중인 종이"를 실시간으로 비쳐 보면서 미리 시작 가능 (완성 여부와 무관하게 최신 상태가 바로 보임) |
| 앞 단계가 수정되면 | 다시 사진을 찍어서 다시 전달받아야 반영됨 | 자동으로 반영됨(다시 받을 필요 없음) |
| 큰 장면 다룰 때 | 전체를 다 불러와야 해서 무거움 | 필요한 부분만 골라 불러올 수 있어 가벼움 |
| 배우는 난이도 | 쉬움, 개념이 단순함 | 어려움, 새로 익힐 개념이 많음 |
| 지원 프로그램 | 거의 모든 프로그램에서 오래전부터 지원 | 빠르게 늘고 있지만 프로그램마다 지원 수준 다름 |

* ⚠️ 주의: "동시 작업 가능"은 "순서 없이 아무 때나 작업해도 된다"는 뜻이 아님. 모델링 → 리깅 → 애니메이션 → 라이팅 순서는 그대로 유지됨. 다만 "완성본을 기다렸다가 통째로 다시 받는" 대기·재작업 구간이 줄어드는 것 (자세한 설명 → [3.1절](#31-usd-특징))
* 둘은 완전히 갈아타는 관계가 아님
* USD 안에서도 무거운 애니메이션 데이터는 여전히 .abc 파일을 그대로 가져다 씀
* 즉 "사진(.abc)은 그대로 쓰되, 사진들을 겹쳐서 관리하는 방식만 USD로 바꾸는" 조합이 흔함
* ([참고](https://docs.nvidia.com/learn-openusd/latest/composition-basics/layers.html))

### 3.7 장단점 분석

USD 장점

* 앞 단계가 "완전히 끝나기 전"에도 뒷 단계가 미리 작업을 시작할 수 있음 (작업 순서 자체는 그대로, 대기 시간만 줄어듦)
* 여러 부서가 같은 씬의 서로 다른 레이어를 건드려도 서로 안 망가짐
* 큰 장면도 필요한 부분만 불러와서 가볍게 작업 가능
* 재질(질감) 정보를 렌더러 바꿔도 재사용 가능
* 세계적으로 표준으로 자리잡는 중이라, 나중에 다른 스튜디오·업체와 호환성이 좋아짐

USD 단점

* 배워야 할 새 개념이 많음. 종이를 겹치는 규칙 자체가 복잡함
* Maya는 아직 최신 버전(2023 이상)에서만 잘 지원됨
* "기술보다 사람들 작업 습관 바꾸는 게 더 힘들다"는 얘기가 많음. 파이프라인이 클수록 전환이 오래 걸림
* 기존 폴더 구조, ShotGrid 퍼블리시 방식도 다시 설계해야 함
* 파일 경로를 관리해주는 도구(Asset Resolver)를 직접 만들거나 손봐야 함

.abc 방식 장점

* 개념이 단순해서 배우기 쉬움
* 웬만한 프로그램·렌더러에서 다 안정적으로 지원
* 단순한 캐시 주고받기에는 USD보다 오히려 간편함
* 지금 쓰는 ShotGrid, 폴더 구조를 그대로 계속 쓸 수 있음

.abc 방식 단점

* 남의 작업 결과를 살짝 고치려 해도, 새로 다시 찍어야(재작업) 함
* 여러 파일을 조립하는 방법이 스튜디오마다 제각각임. 다른 곳과 호환이 어려움
* 큰 장면을 다룰 때 비효율적임. 필요 없는 부분까지 다 불러옴

### 3.8 inFX 파이프라인 시사점

* 한 번에 다 바꾸기보다는 이렇게 접근하는 게 현실적
  * 애니메이션 캐시는 지금처럼 .abc를 그대로 씀
  * 여러 부서의 씬을 조립하는 부분만 USD로 조금씩 도입함
  * (→ [5장](#5-계획과-방법)에서 자세히 정리 예정)
* Maya 2024 업그레이드는 완료됨 (전 사이트 설치 완료)
  * 다음 과제는 Python 3.7.7 → 3.10 전환에 따른 기존 코드/모듈 호환성 점검 (→ [4장](#4-전환-준비))

### 3.9 참고 자료

- [PixarAnimationStudios/OpenUSD (공식 GitHub)](https://github.com/PixarAnimationStudios/OpenUSD) — Pixar가 관리하는 USD 소스코드 저장소
- [OpenUSD 공식 문서](https://openusd.org/release/intro.html) — Pixar 공식 레퍼런스 사이트
- [AOUSD 공식 사이트](https://aousd.org/) — Alliance for OpenUSD, Core Specification
- [NVIDIA Learn OpenUSD](https://docs.nvidia.com/learn-openusd/latest/) — 합성, Hydra 등 심화 가이드
- [Autodesk/maya-usd GitHub](https://github.com/autodesk/maya-usd) — Maya USD 플러그인 소스/릴리즈
- [USD Survival Guide](https://lucascheller.github.io/VFX-UsdSurvivalGuide/) — 실무자 작성 비공식 가이드 (합성, Asset Resolver 등)
- [VFX-UsdAssetResolver](https://github.com/LucaScheller/VFX-UsdAssetResolver) — Asset Resolver 레퍼런스 구현
- [SideFX Houdini USD Basics](https://www.sidefx.com/docs/houdini/solaris/usd.html) — Houdini USD/Solaris 공식 문서
- [Nuke USD Export 공식 문서](https://learn.foundry.com/nuke-stage/current/Content/stage_environment/loading_stage/usd_export_nuke.html) — Nuke USD 지원 버전 안내
- [Unreal Engine USD 공식 문서](https://dev.epicgames.com/documentation/en-us/unreal-engine/universal-scene-description-usd-in-unreal-engine) — UE USD 지원 안내
- [Blender 4.0 Import & Export 릴리즈 노트](https://developer.blender.org/docs/release_notes/4.0/import_export/) — Blender USD 지원 현황
- [Foundry: How USD is set to change the face of VFX](https://www.foundry.com/insights/film-tv/usd-explainer-guide) — USD 도입 전략, file-by-file 접근 권장 근거
- [Implementing USD: A Case Study in Incremental Adoption (SIGGRAPH Educators Forum)](https://dl.acm.org/doi/10.1145/3721242.3734008) — BYU 애니메이션 스튜디오 단계적 도입 사례
- [Usd Asset Resolver Overview](https://lucascheller.github.io/VFX-UsdAssetResolver/overview.html) — Asset Resolver 단계적 구축 가이드

## 4. 전환 준비

### 4.0 빌드/배포 환경

#### 4.0.1 USD 실행 환경 현황

| 환경 | Python | 내장 USD | Python 패키지명 | 비고 |
| --- | --- | --- | --- | --- |
| Maya 2024 + mayaUsd 0.25.0 | 3.10.8 | 22.11 | `pxr` | 실측 (`mayapy`) |
| Houdini 20.5 | 3.11.7 | 24.03 | `pxr` | 문서 기준, 로컬 미설치로 미실측 ([참고](https://www.sidefx.com/docs/houdini/news/20_5/platforms.html)) |
| Katana 7.0 | 3.10 | 23.05 | `fnpxr` | `pxr`와 이름 충돌 회피용으로 Foundry가 의도적으로 분리 ([참고](https://learn.foundry.com/katana/Content/release_notes/7.0/Katana_7.0v1_ReleaseNotes.html)) |
| 사내 표준 Python | 3.10.11 | 없음 | - | `C:\Programs\Python310`, Deadline 플러그인은 `W:\inhouse\python\python3.10.11` 사용 |

#### 4.0.2 DCC 밖 USD 처리용 패키지: `usd-core` (pip)

- 실측 (2026-09-08, Python 3.10, Windows)
  - `usd_core-26.8-cp310-none-win_amd64.whl` 13.8 MB, 의존 패키지 없음
  - `pip download` → `--no-index --find-links` 오프라인 설치 성공 (기존 `W:\inhouse\pypi-wheels` 절차 그대로)
  - `from pxr import Usd` → Stage 생성·usda 출력 정상. 설치 용량 49 MB
  - Rocky 8 서버용 manylinux_2_28 wheel 제공 (glibc 2.28 호환)
- 결론: 폐쇄망 배포에 장애 없음. wheel 한 개 추가로 끝

#### 4.0.3 DCC 내장 USD와의 충돌 (실측)

- 충돌 원리: `pxr` 패키지 이름이 같음. `sys.path`에서 먼저 잡히는 쪽이 import됨. DCC 내장 USD와 다른 버전의 `pxr`가 먼저 로드되면 DCC 플러그인의 Python 바인딩이 깨짐
- Maya 2024 실측 결과

| 케이스 | 결과 |
| --- | --- |
| usd-core `pxr`가 mayaUsd보다 먼저 잡힘 (예: `flova_libs_maya2024`에 설치) | **충돌**. `mayaUsd.lib` import 시 `Tf_PyEnumWrapper has not been created yet` RuntimeError, 이중 등록 RuntimeWarning 다수. `Sdf`는 26.8, 플러그인은 22.11로 뒤섞임 |
| mayaUsd 먼저 로드 후 usd-core 경로를 뒤에 추가 | 정상. 이미 import된 Maya `pxr` 유지 |
| 외부 Python이 usd-core import 후 `mayapy` 자식 프로세스 실행 (PYTHONPATH 미상속) | 정상. usd-core가 부모 `PATH`에 자기 `pxr` 폴더를 추가하나, mayaUsd `.mod`가 자기 lib를 앞에 두므로 자식은 22.11 로드 |
| 외부 Python이 usd-core import 후 `mayapy` 실행 (PYTHONPATH 상속) | **충돌**. 자식이 26.8 `pxr` 로드 |

- Houdini 20.5 (문서·커뮤니티 근거, 미실측)
  - Houdini는 `PYTHONPATH` 항목을 자기 라이브러리보다 앞에 둠. `pxr`가 `PYTHONPATH`에 있으면 Solaris가 그것을 먼저 잡음
  - Python 3.11이라 cp310 wheel은 로드 자체가 실패 → ABI 크래시 대신 ImportError로 LOP Python 노드·Solaris 도구 오작동
  - SideFX·커뮤니티 모두 "외부 USD를 `PYTHONPATH`·`PATH`에 두지 말 것"을 권고 ([참고 1](https://www.sidefx.com/forum/post/383527/), [참고 2](https://www.sidefx.com/forum/topic/81272/))
- Katana 7.0: `fnpxr`로 이름이 분리돼 `pxr` 충돌 없음. 단 `katana7.0_v4.bat`이 `C:\Programs\Python310\Lib\site-packages`를 `PYTHONPATH`에 넣고 있어, 다른 바이너리 패키지 충돌 위험은 별도 존재 (pip 가이드 6.2 규칙 위반)

#### 4.0.4 현재 런처의 `PYTHONPATH` 구성

| 런처 | PYTHONPATH | usd-core 노출 여부 |
| --- | --- | --- |
| `houdini20.5.bat` | `W:\inhouselova;W:\inhouselova_libs` | flova_libs에만 안 넣으면 안전 |
| `maya2024*.bat` | `flova;flova\maya\startup;W:\inhouse\pymel;W:\inhouselova_libs_maya2024` | flova_libs_maya2024에만 안 넣으면 안전 |
| `katana7.0_v4.bat` | `flova;W:\inhouselova_libs;C:\Programs\Python310\Lib\site-packages` | 로컬 site-packages 노출. `fnpxr`라 usd-core 자체는 무해 |
| 일반 Python 도구 (`*.bat` 23종) | `flova;W:\inhouselova_libs` | usd-core를 쓰려면 별도 경로 추가 필요 |

#### 4.0.5 Maya 동봉 USD를 Maya 없이 사용 (실측)

- 사내 표준 Python 3.10.11에서 Maya 2024 동봉 USD(22.11) 직접 사용 가능
  - `PYTHONPATH`: `C:\Program Files\Autodesk\MayaUSD\Maya2024\0.25.0\mayausd\USD\lib\python`
  - `os.add_dll_directory()` 3곳: `...\USD\lib`, `...\USD\bin`, `C:\Program Files\Autodesk\Maya2024\bin`
  - Python 3.8+는 `PATH`로 DLL을 찾지 않으므로 `add_dll_directory` 필수
- Maya 실행·라이선스 체크아웃 없음. 추가 패키지 배포 없음
- `usdcat`·`usdchecker`·`usdview` 동봉 (`...\USD\bin\*.cmd`, 내부에서 `mayapy` 호출하므로 `PATH`에 Maya `bin` 필요)
- 전제: 실행 머신에 Maya 2024 + mayaUsd 설치 (전 자리·Deadline 워커 해당)

#### 4.0.6 배치 원칙 (결정)

- 외부 USD 패키지(usd-core)는 **도입하지 않음**
  - 이유: DCC 내장과 버전 혼재·`pxr` 충돌 위험을 원천 제거. 배포 항목 추가 없음
  - usd-core 조사 결과(4.0.2~4.0.4)는 Maya 없는 서버에서 USD 처리가 필요해질 때의 대안으로 보존
- DCC 안: 각 DCC 내장 USD 사용 (Maya 22.11, Houdini 24.03, Katana 23.05)
- DCC 밖 (표준 Python 3.10.11, Deadline 플러그인): Maya 동봉 USD 재사용 (4.0.5)
  - `flova`에 경로·DLL 디렉터리 설정 헬퍼 1개를 두고 모든 standalone 진입점이 공유
- USD 버전 기준: **22.11 (Maya 2024)**. 모든 DCC·도구가 쓰는 파일은 22.x 범위 스키마만 사용. Houdini 24.03 신규 기능은 Maya에서 무시되므로 사용 금지
- 외부 Python에서 DCC를 자식 프로세스로 띄울 때: Maya USD 경로가 `PYTHONPATH`로 Houdini에 상속되지 않도록 env에서 제거
- `katana7.0_v4.bat`의 로컬 site-packages 노출은 별도 정리 대상

### 4.0.7 Asset Resolver 전략 (결정)

**Asset Resolver란**

- USD 파일 안에 적힌 경로 문자열을 실제 파일 경로로 바꿔주는 부품
- 기본 제공 `ArDefaultResolver`가 절대 경로, 파일 기준 상대 경로, `PXR_AR_DEFAULT_SEARCH_PATH` 기준 검색 경로를 지원
- 커스텀 Resolver는 DCC마다 USD 버전이 달라(22.11 / 24.03 / 23.05) 3벌 빌드·배포 필요 → 도입하지 않음

**결정: 기본 Resolver + 상대 경로 + 검색 경로**

- 검색 경로: `PXR_AR_DEFAULT_SEARCH_PATH=%DRIVE%/show` (예: `M:/show`)
  - 각 DCC 런처 `.bat`과 standalone 진입점에 한 줄 추가
- 에셋 내부 참조 (진입점 → 스텝 결과물): 파일 기준 상대 경로
  - 예: `bus.usd` 안에서 `../model/pub/data/usd/v002/bus_model_v002.usd`
- 샷 → 에셋 참조: 프로젝트 코드부터 시작하는 검색 경로 기준 상대 경로
  - 예: `TEST_TH/assets/cha/bus/usd/bus.usd`
  - 드라이브 문자에 의존하지 않음. 리눅스 렌더팜은 `PXR_AR_DEFAULT_SEARCH_PATH`만 마운트 경로로 바꾸면 동일 파일 사용
- 절대 경로(`M:/show/...`)는 파일 안에 쓰지 않음

**"최신 버전" 처리**

- Resolver가 아니라 진입점 파일이 담당
- 펍툴이 퍼블리시 시 `%ASSET_PATH%/usd/%ASSET_CODE%.usd`의 참조를 새 버전 경로로 재작성
- 샷에서 특정 버전 고정이 필요하면 샷 레이어에서 참조 경로를 해당 버전 폴더로 직접 지정

**실측 (USD 22.11)**

- 상대 경로 참조 정상
- `PXR_AR_DEFAULT_SEARCH_PATH` + `TEST_TH/assets/...` 경로 정상 해석
- 샷 레이어에서 참조를 v001로 바꾸면 즉시 v001 반영 (버전 고정 동작 확인)

**커스텀 Resolver 재검토 시점**

- ShotGrid 승인 상태로 버전을 고르는 등 DB 조회가 경로 해석에 들어갈 때
- 그 전까지는 불필요

### 4.0.8 레이어 구조 표준안 (결정)

**전제 (조사 결과)**

- 렌더러: Arnold. 룩뎁 산출물은 Arnold 셰이더 + 쉐이딩 엔진 할당(json) + 텍스처
- MtoA(Maya 2024)에 arnold-usd 동봉 (USD 22.11 빌드): `usd_proc.dll`(USD 직접 렌더), Hydra 딜리게이트, mayaUsd용 Arnold 머티리얼 익스포터
- mayaUsd에 `usdAbc`(.abc 직접 참조), `usdMtlx` 동봉
- 리그는 Maya 디포머·컨트롤러라 USD 스키마로 표현 불가

**에셋 레이어 (위가 강함)**

```
%ASSET_PATH%/usd/%ASSET_CODE%.usd            ← 진입점. /%ASSET_CODE% prim + payload
└ %ASSET_PATH%/usd/%ASSET_CODE%_payload.usd  ← sublayer 스택만 가진 빈 레이어
   ├ lookdev/pub/data/usd/%VERSION%/..._lookdev_....usd  (강) 머티리얼 + 바인딩 over
   └ model/pub/data/usd/%VERSION%/..._model_....usd      (약) 지오메트리 정의
```

- 진입점은 payload 하나만 가짐. 샷에서 수백 에셋을 열 때 지오메트리를 필요할 때만 로드
- 룩뎁 레이어는 지오메트리를 다시 쓰지 않고 `over`로 머티리얼만 얹음. 모델 버전이 올라가도 룩뎁 레이어 재사용
- 퍼블리시 시 펍툴이 `_payload.usd`의 sublayer 경로를 새 버전으로 재작성
- 리그 스텝: USD 산출물 없음. `.mb` 유지

**샷 레이어 (위가 강함)**

```
%SHOT_PATH%/usd/%SHOT_CODE%.usd              ← 샷 진입점. 부서별 sublayer 스택
 ├ lighting/pub/data/usd/%VERSION%/..._lighting_....usd   라이트, 머티리얼 override
 ├ fx/pub/data/usd/%VERSION%/..._fx_....usd               시뮬 결과 (.abc/.vdb 참조)
 ├ animation/pub/data/usd/%VERSION%/..._anim_....usd      geo를 애니 .abc로 교체(over), 카메라
 └ layout/pub/data/usd/%VERSION%/..._layout_....usd       에셋 배치: /shot/bus1 → reference TEST_TH/assets/cha/bus/usd/bus.usd
```

- 부서는 자기 레이어 파일만 씀. 샷 진입점은 sublayer 목록만 갱신
- 애니메이션 레이어: 레이아웃이 놓은 prim에 `over`로 애니 `.abc`를 참조시켜 정지 지오메트리를 캐시로 교체
- 샷 경로·파일명 규칙은 4.1의 에셋 규칙과 동일 패턴 (`%SHOT_PATH%/usd/`, `%SHOT_CODE%_%TASK_CODE%_%VERSION%.usd`)

**결정 사항**

| 항목 | 결정 |
| --- | --- |
| 에셋 합성 | payload + sublayer 스택 (model < lookdev) |
| 샷 합성 | 샷 진입점 하나에 부서별 sublayer |
| 룩뎁 머티리얼 | Arnold 노드 그대로 (MtoA 익스포터, 렌더 결과 동일) + 뷰포트용 UsdPreviewSurface 병기 |
| 리그 스텝 | USD 산출물 없음 |
| 애니 지오메트리 | `.abc` 캐시를 USD가 참조. USD 타임샘플 직접 export는 추후 이행 검토 |

**쉬운 설명**

- 에셋 = 겉봉투(`bus.usd`) 안에 속봉투(`bus_payload.usd`), 속봉투 안에 모델 종이와 색칠 종이(룩뎁). 색칠 종이는 모델 위에 덧대는 트레이싱지라 모델이 바뀌어도 다시 안 그림
- 샷 = 레이아웃 → 애니 → FX → 라이팅 순으로 트레이싱지를 쌓음. 위 종이가 아래를 덮음
- 리그는 Maya 안에서만 존재하는 조종 장치. 조종한 결과(애니 캐시)만 USD에 들어감

### 4.0.9 기존 데이터 마이그레이션 방안 (결정)

**조사 결과**

- 이전의 "에셋/리깅 펍툴 마이그레이션 퍼블리시"는 프로젝트 간 재퍼블리시 기능이었고 제거됨 (`6c5dad8`). 재사용 도구 없음
- 대상: 퍼블리시된 에셋의 모델 `.abc`, 룩뎁(Arnold 셰이더 + 할당 json + 텍스처). 리깅은 USD 산출물이 없어 대상 아님
- 모델: `usdAbc` 플러그인으로 USD가 `.abc`를 직접 참조 가능 → 변환 없이 참조 한 줄짜리 USD 레이어만 생성
- 룩뎁: Arnold 머티리얼 export는 MtoA 익스포터가 Maya 안에서만 동작 → 에셋당 `mayapy` 배치 1회

**결정: 온디맨드 에셋 단위 변환**

- 일괄 변환하지 않음. 필요한 에셋만 그때그때 변환
- 변환 내용
  - 모델 레이어: 기존 `.abc`를 참조하는 USD 생성 (`model/pub/data/usd/%VERSION%/`)
  - 룩뎁 레이어: `mayapy`로 룩뎁 씬을 열어 Arnold 머티리얼 USD export (`lookdev/pub/data/usd/%VERSION%/`)
  - 진입점·payload 생성 (`%ASSET_PATH%/usd/`)
- 실행 위치: Deadline (룩뎁 export는 `mayapy`, 나머지는 4.0.5 standalone)
- 종료 프로젝트: 변환하지 않음. 다음 퍼블리시부터 USD 생성

**쉬운 설명**

- 이미 찍어둔 사진(.abc)은 그대로 두고 "이 사진을 보세요"라는 표지판(USD)만 세움. 사진을 다시 찍지 않음
- 색칠 정보(룩뎁)만 Maya를 한 번 돌려 꺼냄
- 쓰지 않는 에셋·끝난 프로젝트는 손대지 않음

**검증 필요**

- `usdAbc`로 참조한 지오메트리를 Arnold `usd_proc`이 렌더할 수 있는지 (arnold-usd의 usdAbc 로드 여부). 불가 시 모델은 `.abc` → USD 실제 변환으로 대체

**미결정**

- 대상 프로젝트 범위: 진행 중 프로젝트 전체 vs 지정 프로젝트
- 트리거 위치: 에셋 로더에서 "USD 없음" 감지 시 자동 Deadline 제출 vs 펍툴 수동 버튼

### 4.1 애셋 퍼블리시 네이밍 규칙 및 디렉터리 구조 (초안)

`flova.maya.app.pub_tools`의 에셋 펍툴/리깅 펍툴이 USD로 저장할 때 적용할 경로·파일이름 규칙. 기존 템플릿(`flova/template/*.yaml`) 변수 체계를 그대로 따르고, USD 전용 요소만 추가하는 방식.

**기본 원칙**

* 새 체계를 만들지 않고, 기존 `%ASSET_PATH%`, `%STEP_CODE%`, `%ASSET_FILENAME%` 등 템플릿 변수 규칙을 그대로 따름
* 기존 `.abc` 캐시 규칙(`MODEL_CACHE_PUB_PATH: '%ASSET_PATH%/model/pub/data/abc'`)과 대구를 이루도록 설계
* `%ASSET_PATH%` 바로 아래는 스텝이 아닌 폴더(`thumbnail` 등)가 이미 존재 → 같은 자리에 `usd` 폴더를 추가해도 기존 구조와 자연스럽게 맞음

**1) 애셋 진입점 (여러 스텝의 USD를 최종적으로 겹쳐서 참조하는 파일)**

```
%ASSET_PATH%/usd/%ASSET_CODE%.usd
```

* `%STEP_CODE%` 서브폴더가 아닌, `%ASSET_PATH%` 바로 아래 별도 `usd` 폴더에 둠 (스텝 폴더와 혼동 방지)
* 이 파일이 각 스텝의 퍼블리시 결과(모델/룩뎁/리깅 등)를 Reference/Sublayer로 겹쳐서 최종 애셋을 구성

**2) 각 스텝의 퍼블리시 결과물**

```
%ASSET_PATH%/%STEP_CODE%/pub/data/usd/%VERSION%/%ASSET_CODE%_%TASK_CODE%_%VERSION%.usd
```

* 기존 `data/abc`와 나란히 `data/usd`를 둬서, 같은 스텝 안에서 `.abc`와 `.usd`가 구조적으로 충돌하지 않음
* 버전을 폴더 단위(`%VERSION%`)로 관리

**왜 USD도 버전(`v001`, `v002`...)이 필요한가**

* USD는 "여러 레이어를 겹쳐 보여주는 방식"일 뿐, 자체적으로 이전 상태를 저장해주는 기능이 없음
* 파일을 열어서 그냥 저장하면, 예전 내용은 사라짐. Git처럼 되돌리기가 자동으로 되지 않음
* 오히려 USD는 다른 파일이 이 파일을 "참조"하는 구조라서, 버전 없이 파일 하나만 쓰면 더 위험함
  * 예: 애니메이션팀이 `bus_model.usd`를 참조해서 작업 중인데, 모델링팀이 그 파일을 덮어쓰면, 애니메이션팀도 모르는 사이에 씬이 바뀜
* 그래서 지금처럼 버전마다 새 폴더(`v001`, `v002`...)를 만들고, 예전 폴더는 그대로 남겨두는 방식이 여전히 필요함
* "진입점" 파일(`bus.usd`)은 그중 "최신 버전을 가리키는 표지판" 역할만 함. 필요하면 특정 버전을 고정해서 가리키게 할 수도 있음

**예시 (TEST_TH / cha / bus / model / model / v001)**

```
M:/show/TEST_TH/assets/cha/bus/
├── usd/
│   └── bus.usd                              ← 애셋 진입점
├── thumbnail/
├── model/pub/data/usd/v001/bus_model_v001.usd
├── lookdev/pub/data/usd/v001/bus_lookdev_v001.usd
└── rig/pub/data/usd/v001/bus_rig_v001.usd
```

**파일 확장자 (결정)**

* 모든 USD 파일 확장자: `.usd`
* 저장 인코딩은 파일 역할로 구분

| 파일 | 인코딩 | 이유 |
| --- | --- | --- |
| 진입점·payload·샷 진입점 | usda (텍스트) | 참조 몇 줄뿐. 메모장으로 바로 확인 가능 |
| 스텝 레이어 (model, lookdev, anim 등) | usdc (바이너리) | 지오메트리·머티리얼 데이터. 실측 기준 열기 100배 빠름 |

* `.usd`는 확장자만으로 포맷을 정하지 않음. 저장 시 기본 usdc, `args={'format': 'usda'}`로 텍스트 저장. 읽기는 헤더로 자동 판별

**결정 이유**

* 참조 경로에 확장자가 그대로 박힘. `.usdc`/`.usda`로 나누면 나중에 인코딩을 바꿀 때 그 파일을 참조하는 모든 파일의 경로를 다시 써야 함. `.usd`는 인코딩을 바꿔도 경로가 그대로임
* ASWF USD Working Group 공식 권고와 Pixar 프로덕션 관행이 `.usd` 단일 확장자. 업계 표준을 따르면 외부 스튜디오·벤더 데이터 교환 시 규칙 충돌이 없음
* 확장자 1종이라 템플릿(`flova/template/*.yaml`)·역매핑 정규식·펍툴 코드에서 분기가 없음
* 검토했던 다른 안을 택하지 않은 이유
  * `.usdc`/`.usda` 분리안: 확장자만 보고 포맷을 알 수 있는 장점은 있으나, 위 경로 재작성 문제와 업계 표준 불일치가 더 큼. 포맷 확인은 `usdcat`이나 파일 헤더 8바이트로 대신 가능
  * 전부 `.usdc`안: 진입점까지 바이너리라 문제 발생 시 메모장 확인이 불가. 진입점을 텍스트로 두는 이점을 잃음
* 실측 (USD 22.11, 1만 정점 메시 24프레임)

| 인코딩 | 크기 | 쓰기 | 열기+읽기 |
| --- | --- | --- | --- |
| usda | 3.77 MB | 101 ms | 116 ms |
| usdc | 2.88 MB | 18 ms | 2 ms |

* 업계 관행
  * ASWF 「Asset Structure Guidelines」: 진입점은 `.usd` 강력 권장, 나머지 레이어는 crate 바이너리. 일부 스튜디오는 전부 `.usd` ([링크](https://github.com/usd-wg/assets/blob/main/docs/asset-structure-guidelines.md))
  * Pixar 프로덕션: `.usd` 확장자에 바이너리 내용
  * ALab(공개 예제): 전부 `.usda`. 학습용이라 성능 기준 아님

## 5. 계획과 방법

### 5.1 기본 원칙

* 한 번에 전면 전환하지 않음
* "파일 단위, 필요한 곳부터" 조금씩 넓혀감
* 기존 .abc 캐시는 당장 걷어내지 않음. USD 안에서 계속 재사용
* 큰 파이프라인일수록 전환이 느림. 조급하게 밀어붙이지 않음

이렇게 접근하는 이유

* 업계 사례 다수가 "전면 전환"보다 "필요한 작업부터 file-by-file 도입"을 권장
* 파이프라인이 클수록 "기술보다 사람들 작업 습관 전환이 더 어렵다"는 보고가 많음
* ([참고](https://www.foundry.com/insights/film-tv/usd-explainer-guide))

### 5.2 단계별 로드맵 (권장 순서)

**1단계 · 기반 다지기**

* USD 개념 학습, 소규모 실험(usdview로 파일 열어보기 등)
* Asset Resolver 프로토타입 제작 (처음엔 Python으로 빠르게, 나중에 C++로 다듬는 방식 권장)
* ✅ Maya 2024 업그레이드 및 전 사이트 설치 완료 → mayaUsd 사용 가능 상태
* Python 3.7.7 → 3.10 전환에 따른 기존 코드/모듈 호환성 점검 (→ [4장](#4-전환-준비))
* ([Asset Resolver 시작 가이드](https://lucascheller.github.io/VFX-UsdAssetResolver/overview.html))

**2단계 · 애셋 퍼블리시부터 USD로**

* 가장 먼저 손대기 쉬운 지점: 모델/애셋을 USD 형식으로 저장하고, Reference로 조립하는 부분
* 애니메이션 캐시는 그대로 .abc 유지. USD가 그 .abc를 참조하는 방식으로 시작
* 이 단계는 리스크가 낮음. 최종 렌더 결과물에 영향이 적음

**3단계 · 레이아웃 · 씬 조립**

* 여러 애셋을 모아 샷을 구성하는 부분을 USD Composition으로 전환
* 부서 간 레이어 분리 구조(Layout / Anim / FX / Lighting)를 이 단계에서 설계

**4단계 · 라이팅 · 렌더링**

* 가장 마지막에 전환. 리스크가 크고, 렌더러 연동(Hydra, Render Delegate)까지 걸림
* 실제 사례에서도 "애셋 저장 → 레이아웃 조립 → 라이팅/렌더링" 순서로 단계를 넓혀감
* ([BYU 애니메이션 스튜디오 사례](https://dl.acm.org/doi/10.1145/3721242.3734008))

**참고 사례: BYU 애니메이션 스튜디오**

* 4개 작품에 걸쳐 순서대로 USD 범위를 넓힘
* 1번째 작품(Cenote): 애셋 저장, 레이아웃 조립만 USD로
* 2번째 작품(The Witch's Cat): 1번째 결과 위에 라이팅·렌더링까지 USD로 확장
* 3번째 작품(Student Accomplice): 대규모 환경, 여러 부서의 겹치는 작업 일정까지 USD로 소화
* 핵심 교훈: 작품(프로젝트) 단위로 "지난 번 범위 + 한 단계"씩 넓히는 방식이 팀 부담을 줄임

### 5.3 파일럿 프로젝트 선정 기준

* 일정이 급하지 않은 프로젝트
* 부서 수·샷 수가 적어 리스크가 작은 프로젝트
* 실패해도 다시 기존 방식(.abc)으로 되돌리기 쉬운 프로젝트
* 가능하면 Houdini 비중이 높은 프로젝트 (USD 지원이 가장 성숙함)

### 5.4 검증 방법과 성공 기준 (결정)

**합격 기준 (3개, 모두 충족 시 합격)**

| # | 기준 | 측정 방법 | 판정 |
| --- | --- | --- | --- |
| 1 | 렌더 결과 일치 | 같은 샷을 기존 `.abc` 경로와 USD 경로로 렌더해 비교 | 픽셀 차이 없음. 하나라도 다르면 불합격 |
| 2 | 에셋 수정 → 샷 반영 소요 시간 | ShotGrid Version 생성 시각 차이 (에셋 v+1 → 샷 Version) | 기존 대비 감소하면 합격 |
| 3 | 아티스트 지속 사용 의사 | 파일럿 참여자 설문 "계속 쓰겠다" 응답 | 과반이면 합격 |

**기준 선정 이유**

- 기준이 많으면 판정이 흐려짐. 3개로 제한
- 1번: 품질 안전장치. 룩뎁 Arnold 네이티브 export(4.0.8) 검증
- 2번: USD 도입 목적(대기·재전달·재작업 감소, 3.1절) 직결
- 3번: "기술보다 작업 습관 전환이 어렵다"는 조직 리스크(3.7절) 확인
- 2번 목표치를 수치로 두지 않는 이유: 파일럿 규모가 작아 기준선 편차가 큼. 방향만 확인

**참고값 (합격 판정에 쓰지 않음, 기록만)**

| 항목 | 측정 방법 |
| --- | --- |
| 재작업 횟수 | 같은 태스크의 Version 개수 |
| 퍼블리시 실패율 | Deadline 잡 실패 건수 / 전체 |
| 퍼블리시 산출물 크기 | `data/usd` vs `data/abc` 폴더 크기 |
| TD 유지보수 부담 | 파일럿 중 이슈 건수·해결 시간 |

**측정 원칙**

- 기존 자료(타임로그, ShotGrid Version 기록, Deadline 로그, 파일 크기)만 사용. 별도 측정 도구 없음
- 기준선: 파일럿 직전 같은 프로젝트의 `.abc` 방식 기록

### 5.5 리스크와 대응 방안

* **Maya 버전 갭**: ✅ 해소됨. Maya 2024로 전 사이트 업그레이드 및 설치 완료 (mayaUsd 공식 지원 범위 안)
* **Python 호환성**: Maya 2024는 Python 3.10 사용. 기존 Python 3.7.7 기준 코드/모듈 재검토 필요 (진행형 과제)
* **학습 곡선**: Composition Arc 등 새 개념 학습 부담 → 소수 TD 대상 파일럿 교육부터 시작, 이후 전체 확산
- **Asset Resolver 미비**: 표준 구현체가 없어 자체 구축 필요 → 오픈소스 레퍼런스(VFX-UsdAssetResolver)로 프로토타입 후 다듬는 방식 권장
- **기존 ShotGrid 퍼블리시 구조와의 충돌**: 전환 초기엔 기존 구조 유지, 필요한 부분만 USD 경로 추가하는 방식으로 병행

## 6. 미결정 사항

- Python 3.7.7 기준으로 작성된 기존 파이프라인 코드/모듈을 Maya 2024(Python 3.10)에서 어떻게 이관·재검증할지 (전면 재작성 vs 점진적 포팅 등 방식 미정)
- 마이그레이션 대상 프로젝트 범위: 진행 중 프로젝트 전체 vs 지정 프로젝트 (→ [4.0.9절](#409-기존-데이터-마이그레이션-방안-결정))
- 마이그레이션 트리거 위치: 에셋 로더 자동 제출 vs 펍툴 수동 버튼 (→ [4.0.9절](#409-기존-데이터-마이그레이션-방안-결정))
- (작성 예정) 그 외 미결정 항목
