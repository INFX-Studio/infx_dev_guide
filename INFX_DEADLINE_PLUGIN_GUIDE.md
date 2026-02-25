# INFX 데드라인 플러그인 가이드

## 파일명 규칙

패턴: `dl_{dcc}_{기능}.py`

| DCC      | 접두사     | 예시                       | 설명                               |
|----------|------------|----------------------------|-----------------------------------|
| Maya     | `dl_maya_` | `dl_maya_seq_to_mov.py`    | Maya 내에서 실행되는 플러그인      |
| Nuke     | `dl_nuke_` | `dl_nuke_seq_to_mov.py`    | Nuke 내에서 실행되는 플러그인      |
| Houdini  | `dl_hou_`  | `dl_hou_seq_to_mov.py`     | Houdini 내에서 실행되는 플러그인   |
| Katana   | `dl_kat_`  | `dl_kat_seq_to_mov.py`     | Katana 내에서 실행되는 플러그인    |
| Python   | `dl_py_`   | `dl_py_seq_to_mov.py`      | DCC 없이 Python으로 실행되는 플러그인 |

## 폴더 구조

서브폴더를 사용할 수 있다. 서브폴더는 **그룹**(프로젝트, 기능 등)을 나누기 위한 용도로 사용한다. 그룹의 주제는 자유지만, 똑같은 플러그인이 서로 다른 그룹에 존재해서는 안되므로 신중히 고려하여 설계해야 한다.

```
flova/deadline/plugins/
├── dl_py_sg_upload_mov.py
├── dl_py_make_mp4.py
├── dl_maya_cleanup_scene.py
├── dl_nuke_seq_to_mov.py
├── swc_lor/
│   ├── dl_maya_export_cam.py
│   └── dl_py_analyze_scene.py
└── test/
    ├── dl_py_sg_conn_test.py
    └── dl_py_sg_image_upload_test.py
```

## 사용방법

#### 그룹이 없을 때
- `{플러그인 이름}`
- 예시 : `dl_maya_export_cam`

#### 그룹이 있을 때
- `{그룹}/{플러그인 이름}`
- 예시 : `test/dl_py_sg_conn_test`
