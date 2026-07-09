# INRoL-HSB BEVFusion Runtime Notes

이 저장소는 OpenMMLab MMDetection3D의 INRoL-HSB fork입니다. HSB
workspace에서 CARLA/ROS perception을 위해 BEVFusion runtime dependency로
사용합니다.

## Validated Local Setup

아래 환경은 RTX 5090 에서 실제로 검증한 구성입니다.
다른 구성일 경우 아래 링크를 참고하여 설치 진행하면 됩니다.
https://mmdetection3d.readthedocs.io/en/latest/get_started.html 

| 항목 | 값 |
| --- | --- |
| OS | Ubuntu 22.04 |
| Python | 3.10.12 |
| GPU | NVIDIA RTX 5090, compute capability 12.0 |
| CUDA toolkit | `/usr/local/cuda-12.8` |
| PyTorch | `2.7.1+cu128` |
| MMCV | `2.1.0`, CUDA ops 포함 빌드 |
| MMEngine | `0.10.7` |
| MMDetection | `3.3.0` |
| MMDetection3D | `1.4.0` fork baseline |

검증 당시 실제 import 경로는 `site-packages`의 wheel copy가 아니라 editable
checkout이었습니다.

## Workspace Layout

MMDetection3D는 ROS 2 `src/` tree 밖에 두는 것을 권장합니다. 권장 HSB
workspace 구조는 다음과 같습니다.

```text
inrol_hsb_ws/
|-- src/
|   |-- carla_hsb_project/
|   |-- carla_perception_mockup/
|   `-- ...
|-- ml/
|   `-- mmdetection3d/
|-- weights/
|   `-- bevfusion/
`-- .venv/
    `-- mmdet3d/
```

이렇게 두면 `colcon`은 ROS package 빌드에만 집중하고, Python/CUDA runtime
설정은 별도 단계로 명확하게 관리할 수 있습니다.

## Rebuild From a Fresh Workspace

workspace root에서 다음 순서로 구성합니다.

```bash
mkdir -p ml .venv

git clone https://github.com/INRoL-HSB/mmdetection3d.git ml/mmdetection3d
cd ml/mmdetection3d
git checkout hsb/bevfusion-runtime
mkdir -p checkpoints/bevfusion

python3 -m venv ../../.venv/mmdet3d
source ../../.venv/mmdet3d/bin/activate

python -m pip install --upgrade pip setuptools wheel
```

CUDA 12.8용 PyTorch를 설치합니다.

```bash
pip install --index-url https://download.pytorch.org/whl/cu128 \
  torch==2.7.1 torchvision torchaudio
```

OpenMMLab dependency를 설치합니다.

```bash
pip install mmengine==0.10.7 mmdet==3.3.0

export CUDA_HOME=/usr/local/cuda-12.8
export PATH="${CUDA_HOME}/bin:${PATH}"
export LD_LIBRARY_PATH="${CUDA_HOME}/lib64:${LD_LIBRARY_PATH:-}"
export FORCE_CUDA=1
export MMCV_WITH_OPS=1
export TORCH_CUDA_ARCH_LIST=12.0

pip install --no-build-isolation mmcv==2.1.0
pip install -e .
```

BEVFusion project ops를 빌드합니다.

```bash
cd projects/BEVFusion
python setup.py develop
```

Blackwell이 아닌 GPU를 사용하는 경우 CUDA extension을 빌드하기 전에 해당
GPU에 맞는 값으로 `TORCH_CUDA_ARCH_LIST`를 설정하세요. 모든 target build
machine이 CUDA 12.8+를 사용하는 것이 아니라면 shared source에 `sm_120`을
직접 hard-code하지 않습니다.

RTX 5090 / Blackwell 장비에서 CUDA 12.8+를 사용하는 경우,
`projects/BEVFusion/setup.py`가 CUDA arch list를 hard-code하고 있기 때문에
BEVFusion project ops 빌드에 local `sm_120` gencode entry가 필요할 수
있습니다. `compute_120`을 컴파일할 수 있는 장비에서만 아래 패치를 적용하세요.

```bash
python - <<'PY'
from pathlib import Path

path = Path("projects/BEVFusion/setup.py")
text = path.read_text()
needle = "            '-gencode=arch=compute_86,code=sm_86',\n"
patch = "            '-gencode=arch=compute_120,code=sm_120',\n"
if patch not in text:
    path.write_text(text.replace(needle, needle + patch))
PY
```

이후 다시 빌드합니다.

```bash
cd projects/BEVFusion
python setup.py develop
```

## Checkpoints

checkpoint는 용량이 커서 따로 다운로드받아야합니다. INRoL 서버의
`해수부 과제/프로젝트 파일들/perception_checkpoints` 아래의 파일들을 다운로드 받아
아래의 위치에 두면 됩니다.

```text
<mmdetection3d_root>/checkpoints/bevfusion/
```

예시는 다음과 같습니다.

```bash
cd <mmdetection3d_root>
mkdir -p checkpoints/bevfusion
cp <perception_checkpoints>/bevfusion/*.pth checkpoints/bevfusion/
```

## Runtime Boundary

이 저장소는 MMDetection3D와 BEVFusion model runtime 용도로만 사용합니다.

아래 항목들은 `carla_perception_mockup`에 둡니다.

- ROS 2 detection node.
- CARLA sensor subscription과 TF lookup.
- nuScenes/CARLA LiDAR 및 camera frame 변환.
- publish되는 bbox convention 변환.
- RViz, LiDAR BEV, camera overlay visualization.

따라서 CARLA smoke test는 이 fork의 unit test가 아니라
`carla_perception_mockup`의 integration test로 취급합니다.

## Quick Sanity Check

설치 후 다음 sanity check를 실행합니다.

```bash
source <workspace>/.venv/mmdet3d/bin/activate
python - <<'PY'
import torch
import mmcv
import mmengine
import mmdet
import mmdet3d

print("torch", torch.__version__, "cuda", torch.version.cuda)
print("cuda available", torch.cuda.is_available())
print("mmcv", mmcv.__version__)
print("mmengine", mmengine.__version__)
print("mmdet", mmdet.__version__)
print("mmdet3d", mmdet3d.__version__)
if torch.cuda.is_available():
    print("gpu", torch.cuda.get_device_name(0))
    print("capability", torch.cuda.get_device_capability(0))
PY
```

검증한 RTX 5090 환경에서는 CUDA 사용 가능 여부와 compute capability `(12, 0)`이
출력되어야 합니다.

## Known Pitfalls

- PyTorch 2.6+에서는 checkpoint load 시 기본값이 `weights_only=True`로 동작할
  수 있습니다. training metadata가 포함된 OpenMMLab checkpoint를 사용할 때는
  다음 환경변수를 설정하세요.

  ```bash
  export TORCH_FORCE_NO_WEIGHTS_ONLY_LOAD=1
  ```

- build isolation 환경에서 packaging module 누락으로 `mmcv` 빌드가 실패하면
  `--no-build-isolation` 옵션으로 다시 빌드하세요.
- CUDA toolkit 12.8보다 오래된 버전은 Blackwell `compute_120`을 인식하지
  못합니다. 전체 source에 `sm_120`을 commit하기보다 GPU별
  `TORCH_CUDA_ARCH_LIST`를 사용하세요.
- bbox frame 및 origin 보정은 ROS wrapper boundary에서 처리합니다. CARLA
  coordinate convention 때문에 MMDetection3D model code를 직접 patch하지
  않습니다.
