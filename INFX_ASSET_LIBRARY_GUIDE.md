# 에셋 라이브러리 가이드

ShotGrid `TOTAL_LIBRARY` 프로젝트 기반 에셋 라이브러리의 구조와,
펍툴(에셋펍툴/리깅펍툴)에서 라이브러리로 자동 아카이빙하는 설계·계획을 정리한 문서입니다.

---

## 1. 개요

- `TOTAL_LIBRARY`는 소스, 에셋, 매터리얼, HDRI, 군중, 언리얼, 누크 노드 등을 아카이브해두었다가
  필요할 때 원하는 프로젝트에서 쉽게 임포트해 재사용하기 위한 ShotGrid 프로젝트(저장소)입니다.
- 에셋은 한 번 만들면 다른 프로젝트에서도 사용할 일이 많으므로, 프로젝트 종료 후가 아니라
  **각 에셋을 펍하는 시점에 아카이빙하는 것**을 원칙으로 합니다.
- 이 가이드는 두 부분으로 구성됩니다.
  - 현재 TOTAL_LIBRARY의 구조 (2026-08-26 라이브 ShotGrid 실측 기준)
  - 펍툴 자동 아카이빙 설계와 구현 계획

---

## 2. TOTAL_LIBRARY 프로젝트 구조

### 2.1 기본 구조

- ShotGrid Project, id `2795`, 이름 `TOTAL_LIBRARY`
- 라이브러리 항목 1건 = `Version` 엔티티 1건
- 페이지(라이브러리 종류) 구분은 `Type`(`sg_version_type`, list) 필드로 합니다.

| sg_version_type | 등록 건수(실측) | 비고 |
| --- | --- | --- |
| `Source Library` | 15,883 | 등록·소비 툴 모두 운영 중 |
| `Asset Library` | 158 | 수동 등록 툴만 존재. 이번 자동 아카이빙 대상 |
| `Material Library` | 13 | 수동 등록 (코드/이름 정도만 기록됨) |
| `HDRI Library` | 0 | 페이지만 존재 |
| `Crowd Library` | 0 | 페이지만 존재 |
| `Unreal Library` | 0 | 페이지만 존재 |
| `Nuke Node Library` | 0 | 페이지만 존재 |

> ⚠️ **주의**: `flova/shotgrid/cache.py`의 `SGLibraryCache.SG_LIBRARY_PAGE_LIST`에는
> `Material`이 누락되어 있습니다. (SG에는 존재) 라이브러리 관련 작업 시 보완이 필요합니다.

### 2.2 Asset Library 항목의 필드 컨벤션 (기존 수동 등록 기준)

| 필드 (표시 이름 / field code) | 값 예시 | 설명 |
| --- | --- | --- |
| `Code` (`code`) | `volvo_001` | 라이브러리 코드. 수동 툴은 `_NNN` 접미사를 증가시키며 항상 새 항목 생성 |
| `Name` (`sg_name`) | 자유 입력 | 사람이 읽는 이름 (한글 가능) |
| `Asset Type` (`sg_asset_type`) | `veh` | list: `cha` / `env` / `prp` / `bld` / `veh` / `dum` |
| `DCC` (`sg_dcc`) | `Maya` | list: `Maya` / `Houdini` / `Katana` / `Nuke` |
| `Path to Package` (`sg_path_to_package`) | `S:/total_library/asset_library/veh/volvo_001` | 파일서버 패키지 폴더 |
| `Source Path` (`sg_scan_source_path`) | 원본 룩뎁 펍 씬 경로 | 출처 기록 |
| `Description` (`description`) | 자유 입력 | 설명 |
| `Tags` (`tags`) | 에셋명, 타입 등 | 검색용 태그 |
| `Artist` (`user`) | 등록자 | HumanUser |
| 썸네일 (`image`) | jpg | `upload_thumbnail`로 업로드 |

- 기존 방식은 파일을 전부 **파일서버(S:/total_library)** 에 보관하며, ShotGrid File(Attachment)은
  사용하지 않습니다.

### 2.3 기존 등록·소비 경로 (코드 위치)

| 구분 | 위치 | 설명 |
| --- | --- | --- |
| 수동 등록 툴 | `flova/app/asset_library.py` | 버전코드 분석 → Deadline 제출 UI |
| 등록 Deadline 플러그인 | `flova/deadline/plugins/dl_register_asset_library_for_maya.py` | 레퍼런스 임포트 → 클린업 → 텍스쳐를 S:에 복사 → Version 생성 |
| 라이브러리 캐시 | `flova/shotgrid/cache.py` `SGLibraryCache` | Redis 캐시. 페이지 이름으로 조회 |
| 소스 소비 툴 | `flova/app/flova_source_library/tool.py`, `flova/nuke/tool/source_library_panel/` | `Source` 페이지만 소비 툴 존재 |

- Asset Library 페이지의 **임포트(소비) 툴은 아직 없습니다.** 이번 설계의 manifest 스키마가
  향후 임포트 툴의 계약이 됩니다.

### 2.4 Attachment(File) 엔티티 메모

- `attachment_links`(multi_entity)로 임의 엔티티(Version 등)에 링크됩니다.
- 역할 태깅에 쓸 수 있는 `Type`(`sg_type`, text) 필드가 이미 존재하므로 스키마 추가 없이
  사용 가능합니다.
- `sg.upload(entity_type, entity_id, path)`처럼 `field_name` 없이 업로드하면 Attachment가
  생성되어 해당 엔티티의 Files 탭에 링크됩니다. 반환값은 Attachment id입니다.

> ⚠️ **주의**: ShotGrid는 링크가 끊긴 Attachment를 자동 삭제하지 않습니다.
> 덮어쓰기 갱신 시 기존 Attachment를 API로 명시적으로 `delete` 해야 하며,
> delete된 파일은 SG 휴지통으로 이동한 뒤 SG가 정리합니다.

---

## 3. 펍툴 자동 아카이빙 설계

### 3.1 대상과 현재 펍 산출물

- 대상: 에셋펍툴(`AssetPubToolsWindow`), 리깅펍툴(`RiggingPubToolsWindow`)의 일반 publish 경로
- 제외: Migration publish 경로(별도 whitelist 정책 존재), EnvAsset 등 다른 펍툴(향후 확장)
- 에셋 펍은 Deadline `dl_publish_for_asset`이 아래 산출물을 이미 생성하므로,
  아카이빙에 필요한 파일 대부분이 펍 시점에 준비되어 있습니다.

| 산출물 | 파일 | 비고 |
| --- | --- | --- |
| 모델링 | `{acode}_model_{ver}.abc`, `.fbx`, `_blendshape.mb` | DCC 중립 포맷 포함 |
| 쉐이더 | `{acode}_shader_{ver}.mb`, `.ass`, `_data.json`, `_meta.json` | `_data.json` = 쉐이딩엔진→오브젝트 매핑, `_meta.json` = 메쉬별 Arnold 속성 |
| 텍스쳐 | 텍스쳐 버전 폴더 전체 (+`.tx`) | 씬에서 실제 사용된 파일만 복사됨 |
| 룩뎁 씬 | `{acode}_{tcode}_{ver}.mb` | 클린업 완료본 |

- 리깅 펍은 로컬(Maya 세션)에서 `.mb` 저장·복사 후 SG Version을 갱신하며, 캡쳐 이미지를
  업로드합니다. Deadline을 경유하지 않습니다.
- 텍스쳐 노드별 colorspace / uvTilingMode / UDIM 정보는 펍 과정에서 읽기만 하고 기록하지
  않으므로, manifest에 기록을 추가해야 합니다.

### 3.2 라이브러리 Version 데이터 모델 (Asset Library 페이지)

- 식별 키: `project=TOTAL_LIBRARY` + `sg_version_type='Asset Library'` + `code`
- `code` = `{프로젝트코드}_{에셋코드}` (예: `GNS_volvo`)
  - find → 있으면 update(덮어쓰기), 없으면 create
  - 기존 수동 등록 툴의 `{코드}_NNN` 컨벤션과 충돌하지 않습니다.
- 필드 매핑 (모두 기존 필드 재사용):

| 필드 | 값 |
| --- | --- |
| `sg_name` | 에셋 코드 (사람이 읽는 이름, 수동 수정 허용) |
| `sg_asset_type` | 프로젝트 Asset의 `sg_asset_type` |
| `sg_dcc` | `Maya` |
| `sg_path_to_package` | 프로젝트 룩뎁 펍 폴더 (참고용 — 원본 파일은 Attachment) |
| `sg_scan_source_path` | 펍 원본 마야 씬 파일 |
| `description` | 자동 기입: 소스 프로젝트/태스크/버전 + 갱신 일시 |
| `tags` | 에셋코드, 에셋타입, 프로젝트코드 자동 태그 |
| `user` | 퍼블리시한 작업자 |
| 썸네일 (`image`) | 리깅 캡쳐 이미지 또는 프로젝트 Version 썸네일(있는 경우) |

- 신규 필드(선택 권장, **코드 배포 전 생성 필요**):
  - `Source Project` (`sg_source_project`, text): 예 `GNS`
  - `Source Version` (`sg_source_version`, text): 예 `volvo_lookdev_v002`
  - 생략 시 description/tags로 대체 가능하나, 페이지 필터링에는 전용 필드가 유리합니다.

### 3.3 Attachment(File) 구성 — 역할별 `sg_type` 태깅

| sg_type | 파일 | 생성 주체 |
| --- | --- | --- |
| `manifest_asset` | `{acode}_asset_manifest.json` (통합 메타데이터) | 에셋 펍 |
| `model` | `{acode}_model_{ver}.abc`, `.fbx`, `_blendshape.mb`(있으면) | 에셋 펍 |
| `shader` | `{acode}_shader_{ver}.mb`, `.ass`, `_data.json`, `_meta.json` | 에셋 펍 |
| `texture` | 씬에서 사용된 모든 텍스쳐 원본 (UDIM 전체 타일 포함) | 에셋 펍 |
| `manifest_rig` | `{acode}_rig_manifest.json` | 리깅 펍 |
| `rig` | 리깅 펍 `.mb` | 리깅 펍 |

- 텍스쳐 업로드 원칙: **파일이름이 다른 에셋과 같아도 해당 에셋 항목에 전부 업로드**합니다.
  (Attachment는 엔티티별 독립 저장이므로 자연 충족, 에셋 간 중복 제거 없음)
- `.tx`는 원본 텍스쳐에서 재생성 가능한 파생물이므로 업로드 제외를 권장합니다. (결정 대기)
- 프로젝트 폴더가 삭제되어도 ShotGrid에 파일이 남는 것이 이 방식의 핵심입니다.

### 3.4 asset_manifest.json 스키마 (schema_version 1)

- 스키마 정의·생성·검증·입출력은 공용 모듈 `flova/shotgrid/total_library.py`가 담당합니다.
  (`build_asset_manifest`, `validate_asset_manifest`, `write_manifest`, `read_manifest`)

```json
{
  "schema_version": 1,
  "kind": "asset",
  "source": {
    "project": "GNS", "asset_code": "volvo", "asset_type": "veh",
    "task": "lookdev", "version_code": "volvo_lookdev_v002",
    "sg_version_id": 123456, "published_at": "2026-08-26T10:00:00+09:00", "user": "홍길동"
  },
  "dcc": {"app": "Maya", "version": "2022", "renderer": "Arnold(mtoa 4.2.4)"},
  "paths": {
    "model": "M:/show/GNS/assets/veh/volvo/model/pub/data/abc",
    "shader": "M:/show/GNS/assets/veh/volvo/shader/pub/v002",
    "texture": "M:/show/GNS/assets/veh/volvo/texture/pub/v002",
    "lookdev": "M:/show/GNS/assets/veh/volvo/lookdev/pub/scenes"
  },
  "files": [
    {"role": "model", "filename": "volvo_model_v002.abc", "format": "abc"},
    {"role": "shader", "filename": "volvo_shader_v002.mb", "format": "mayaBinary"}
  ],
  "textures": [
    {
      "filename": "volvo_body_diff.1001.tif",
      "path": "M:/show/GNS/assets/veh/volvo/texture/pub/v002/volvo_body_diff.1001.tif",
      "file_node": "file1",
      "colorspace": "sRGB", "uv_tiling_mode": 3, "udim": true,
      "shading_engines": ["volvo_body_SG"]
    }
  ],
  "shader_assign": {"volvo_body_SG": ["body_geo"]},
  "mesh_meta_file": "volvo_shader_v002_meta.json"
}
```

- `paths`는 역할별 산출물 기준 폴더입니다. `files`의 각 항목은 `paths[role]` 아래에 있고,
  `textures`의 각 항목은 절대 경로(`path`)를 직접 가집니다. 후속 아카이빙 잡은 이 경로들로
  업로드할 파일을 찾습니다.
- `textures` 항목의 colorspace / uvTilingMode / UDIM / 쉐이딩엔진 연결 정보는 텍스쳐 퍼블리시
  이후의 씬을 다시 순회하여 수집합니다. (`_collect_texture_records()`)
  이 정보가 "다른 DCC·다른 프로젝트에서 달라붙이기"의 핵심입니다.
- file 노드는 원본 확장자 텍스쳐를 참조하므로 `.tx` 파생 파일은 목록에서 자연 제외됩니다.
- `shader_assign`은 기존 `_data.json`(쉐이딩엔진 → 오브젝트 매핑)과 동일 데이터입니다.
- `schema_version`으로 향후 스키마 진화를 관리합니다.

### 3.5 갱신(덮어쓰기) 규칙

1. `code`로 라이브러리 Version을 find. 없으면 create, 있으면 필드 update.
2. 이번 펍이 담당하는 역할(`sg_type`)의 기존 Attachment를 조회하여 명시적으로 delete.
   - 에셋 펍: `manifest_asset`, `model`, `shader`, `texture` 교체
   - 리깅 펍: `manifest_rig`, `rig` 교체 (에셋 펍 산출물은 보존)
3. 새 파일 업로드(`sg.upload` → Files 탭 링크) 후 각 Attachment에 `sg_type`을 기록.
4. 썸네일 갱신.

- 역할 단위 "delete → upload" 순서이므로 잡을 재시도해도 결과가 같습니다(멱등).

### 3.6 실행 구조

- 신규 Deadline 플러그인 `dl_register_total_library` (pure Python, `pool='sg'`, Maya 불필요):
  manifest를 읽어 3.5의 갱신 규칙을 수행하는 공용 실행기입니다.
- 에셋 펍: `AssetPubToolsWindow.publish_action()`에서 기존 펍 잡(`dl_publish_for_asset`)에
  `dependency_files`로 연결된 후속 잡으로 제출합니다. (기존 노트 잡과 같은 패턴)
  - `dl_publish_for_asset`에 manifest 작성 단계를 추가합니다. (쉐이더 펍 폴더에 저장)
- 리깅 펍: `RiggingPubToolsWindow.publish()`의 로컬 펍 성공 후 잡을 제출합니다.
  업로드가 커도 작업자 Maya 세션을 막지 않습니다.
- 실패 격리: 아카이빙 잡이 실패해도 프로젝트 퍼블리시는 성공으로 유지되고,
  Deadline에서 해당 잡만 재시도하면 복구됩니다.
- 공용 로직은 `flova/shotgrid/total_library.py`(가칭) `TotalLibraryRegistrar`로 모아
  펍툴·수동 툴·향후 다른 라이브러리 페이지(Material, Nuke Node 등)에서 재사용합니다.

---

## 4. 구현 마일스톤

1. **M1 — manifest 생성** ✅ (2026-08-26 완료): 공용 모듈 `flova/shotgrid/total_library.py` 추가,
   `dl_publish_for_asset`에 텍스쳐 노드 정보 수집(`_collect_texture_records`) +
   manifest 저장(`_publish_library_manifest`, 쉐이더 펍 폴더) 단계 연결. 단위 테스트 완료.
2. **M2 — TotalLibraryRegistrar**: find_or_create / 역할별 Attachment 교체 / 업로드 공용 모듈.
   SG mock 단위 테스트.
3. **M3 — dl_register_total_library**: Deadline 플러그인 구현.
4. **M4 — 펍툴 연결**: 에셋 펍 후속 잡 제출 + 리깅 펍 후 잡 제출. 리깅용 rig_manifest 생성.
5. **M5 — 검증·배포**: 신규 SG 필드 생성(코드 배포 전) → 테스트 에셋 e2e(등록 → 재펍
   덮어쓰기 → Attachment 교체 확인) → Oracle 검증 → 배포.

---

## 5. 운영 고려사항 / 주의사항

- ⚠️ **ShotGrid 스키마 작업 순서**: 신규 필드는 반드시 코드 배포보다 먼저 생성해야 합니다.
  없는 field code를 전송하면 업로드가 실패할 수 있습니다.
- ⚠️ **Attachment 자동 삭제 없음**: 2.4의 주의 참고. 덮어쓰기는 명시적 delete로 구현합니다.
- 업로드 용량: 텍스쳐 세트는 수백 MB~수 GB가 될 수 있어 `sg` 풀에서 처리하고,
  잡 로그에 파일 수/총 용량을 남깁니다. 사이트별 대용량 단일 파일 제한은 e2e에서 실측 확인합니다.
- `SGLibraryCache.SG_LIBRARY_PAGE_LIST`의 `Material` 누락 보완을 권장합니다.
- 📌 **참고**: ShotGrid 커넥션 사용법은 `INFX_SHOTGRID_GUIDE.md`, Deadline 플러그인 작성
  규칙은 `INFX_DEADLINE_PLUGIN_GUIDE.md`를 참고하십시오.

---

## 6. 확정된 설계 결정

| # | 항목 | 확정 내용 | 확정일 |
| --- | --- | --- | --- |
| 1 | 항목 식별 키 | `{프로젝트}_{에셋코드}` (프로젝트가 다르면 같은 에셋도 별도 항목) | 2026-08-26 |
| 2 | `.tx` 업로드 여부 | 제외 (재생성 가능한 파생물. file 노드가 원본 확장자를 참조하므로 manifest에서 자연 제외됨) | 2026-08-26 |
| 3 | 신규 필드 `Source Project`(`sg_source_project`), `Source Version`(`sg_source_version`) | 생성 (M5 배포 전 ShotGrid에서 필드 먼저 생성) | 2026-08-26 |
| 4 | 아카이빙 동작 방식 | 항상 자동 실행 (UI 체크박스 없음) | 2026-08-26 |

---

## 7. 문서 이력

| 날짜 | 내용 |
| --- | --- |
| 2026-08-26 | 최초 작성: TOTAL_LIBRARY 구조 실측 정리 + 펍툴 자동 아카이빙 설계·계획 수립 |
| 2026-08-26 | 설계 결정 4건 확정. manifest 스키마에 `paths` 섹션과 텍스쳐 `path` 필드 추가. M1 구현 완료 반영 |
