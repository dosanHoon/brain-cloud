# Three.js로 3D 에셋을 웹에서 렌더링하기

FBX·GLB·BVH 같은 3D 에셋을 웹에서 재생하고, 나중에 프레임 단위로 캡처까지 이어가려면 **로더 선택**, **축·스케일 보정**, **카메라 추적**, **리소스 해제**를 어떻게 할지가 중요합니다.  
실전에서 썼던 패턴을 정리한 글입니다.

---

## 1. 포맷별 로더와 좌표계

| 포맷 | 로더 | 좌표계 | 비고 |
|------|------|--------|------|
| **FBX** | `FBXLoader` (three/examples/jsm) | 보통 Z-up (cm 단위) | 스케일 0.01 + 루트에 Z-up→Y-up 회전 필요할 수 있음 |
| **GLB** | `GLTFLoader` | glTF는 Y-up | 특정 리그(metarig 등)에만 -90° X 보정하는 경우 있음 |
| **BVH** | BVHLoader2 등 커스텀 | 툴마다 다름 | pelvis X 90°, 그 외 본 quaternion 스왑으로 Y-up 맞춤 |

- **스케일:** FBX는 cm 단위가 많아서 `object.scale.setScalar(0.01)` (cm→m) 적용이 일반적.
- **Z-up → Y-up:** Three.js는 Y-up이므로, FBX 루트에 `setRotationFromEuler(new THREE.Euler(-Math.PI/2, 0, 0))` 를 넣는 파이프라인이 많음.  
  
---

## 2. 씬 구성 — 그리드, 바닥, 그림자

- **GridHelper** + **PlaneGeometry** 바닥으로 땅과 그리드 라인을 두고, `receiveShadow = true` 로 그림자를 받게 하면 미리보기·캡처 모두 자연스럽습니다.
- **AxesHelper**는 디버깅용으로만 쓰고, 실제 서비스/캡처용 뷰에는 넣지 않는 편이 좋습니다. (축 끝이 픽셀처럼 보이는 이슈는 `FBX 미리보기 XYZ 축 점 이슈와 해결.md` 참고.)
- **Shadow:** `renderer.shadowMap.enabled`, `DirectionalLight.castShadow = true`, `shadowMapSize`(예: 8192) 설정. 캡처 품질을 올리려면 해상도를 키웁니다.

---

## 3. 카메라·pelvis 추적

- **pelvis** 노드를 찾을 때는 `name.toLowerCase().includes('pelvis')` 또는 `spine` 등 fallback을 두는 방식이 많이 쓰입니다.
- 매 프레임 pelvis의 월드 위치를 구해 **OrbitControls.target**과 **카메라 위치**, 그리고 **ground_group.position**을 같이 옮기면, 캐릭터가 걸어가도 항상 화면 중앙에 오게 할 수 있습니다.
- GLB 캐릭터(예: manny, quinn)에 BVH/모션을 붙일 때는 **SkeletonUtils.clone**으로 메쉬를 복제한 뒤, 원본 스켈레톤에 애니메이션을 적용하는 패턴을 씁니다.

---

## 4. 애니메이션 루프와 캡처 타이밍

- **AnimationMixer** + **Clock.getDelta()** 로 매 프레임 `mixer.update(delta)` 호출.
- **캡처 시:** “한 프레임만 진행 → 렌더 → toDataURL” 하려면, `advanceMotionFrame(delta, { capture: true })` 처럼 **캡처 플래그**로 그 프레임만 진행하고, 그 다음에 `renderer.render(scene, camera)` 한 번 호출한 뒤 캔버스를 읽는 방식이 안전합니다.  
  requestAnimationFrame 루프와 분리해서 “한 스텝만 진행 후 캡처”가 되도록 하면, 배치 스크립트에서 호출할 때 타이밍이 꼬이지 않습니다.

---

## 5. 리소스 해제 (dispose)

- 모델을 바꾸거나 페이지를 나갈 때 **geometry, material, texture, skeleton** 을 반드시 dispose 해야 메모리 누수가 줄어듭니다.
- **정책:**  
  - **full:** 메쉬·머티리얼·텍스처·스켈레톤 전부 dispose.  
  - **skeletonOnly:** 스켈레톤만 dispose (캐릭터 템플릿은 유지하고 모션만 바꿀 때).
- 머티리얼의 `map`, `normalMap`, `roughnessMap` 등 **텍스처 키**를 순회하면서 texture.dispose() 한 뒤 material.dispose() 호출.
- SkinnedMesh의 **skeleton**은 별도로 traverse해서 한 번만 dispose (여러 메쉬가 같은 skeleton을 참조할 수 있음).

---

## 6. 캡처용 뷰 크기·환경 변수

- 뷰 크기(VIEW_WIDTH, VIEW_HEIGHT), FPS, shadow map 크기, 에셋 로드 타임아웃 등을 **환경 변수**로 두면, 로컬 미리보기와 배치 캡처(고정 해상도)를 같은 앱으로 유연하게 맞출 수 있습니다.

---

## 요약

| 항목 | 요약 |
|------|------|
| 로더 | FBX/GLB/BVH 각각 축·스케일 보정 규칙이 다름. FBX는 0.01 스케일 + Z-up→Y-up 자주 필요. |
| 씬 | Grid + 바닥 Plane, shadow, AxesHelper는 선택(또는 제거). |
| 추적 | pelvis 월드 위치로 카메라·target·ground_group 동기화. |
| 캡처 | “한 프레임 진행 → render → toDataURL” 를 동기적으로 한 번씩 호출하는 API가 배치에 유리. |
| 메모리 | geometry/material/texture/skeleton dispose 정책을 두고, 모델 전환 시 꼭 호출. |

BVH·FBX·GLB 포맷 차이와 패키지/클립 데이터는 `BVH-FBX-GLB 포맷과 패키지·클립 데이터 정리.md`를, Varco와의 캘리브레이션 비교는 `Varco frontend vs 3d-to-gif XYZ 캘리브레이션 비교.md`를 참고하면 됩니다.
