# Repo analysis: uhhhci/Gsplat-unity-virtual-human (Phase 1 + 1b)

Source analysed: `.scratch/Gsplat-unity-virtual-human` @ `8ed42ead77f2a3b0aba0d7bc4f927f74453d2718` ("v1.0 release", 2026-09-29).
Path shorthand used below (all relative to that clone's `Assets/`):

- `VH/` = `GsplatRig/` (virtual-human toolkit)
- `GS/` = `gsplat-unity/` (vendored renderer, package `de.uhh.hci.gsplat-unity` 1.0.0)
- Line numbers (`:Lnn`) refer to the files as cloned. Everything was traced from source, not from the README.

How this report was produced: full read of every runtime/editor/shader file on the deformation + sort + draw path, plus
numerical checks on the shipped binary assets with throw-away NumPy/Blender scripts (results in section 12; scripts
were deleted, all numbers are reproducible from the formulas in sections 5-7).

---

## 0. TL;DR (what a later step must know)

1. **Binding = 32 bytes per Gaussian**: `uint32 triIndex`, `float32 u,v` (unclamped in-plane barycentric), `float32 normalOffset`
   (signed, along the rest-pose triangle normal), `float32×4 restLocalRot` (xyzw, Gaussian orientation in the host triangle's
   tangent frame). File = 32-byte header (`"GSRG"`, v1, count) + four SoA sections. Size is exactly `32 + 32·N`
   (sample: N=40,000 → 1,280,032 bytes, verified). Nothing else per Gaussian (no scale, no weights, no bone ids).
2. **Deformation math** (`VH/Runtime/Shaders/GsplatSkinning.compute`): kernel 1 LBS-skins the proxy vertices (≤4 bones, 4×4 matrices);
   kernel 2 rebuilds each Gaussian from its deformed host triangle:
   `p = v0 + e1·u + e2·v + n·off`, `R = R_proxyToScan · [t×n, t, n] · R(restLocalRot)` with `n = norm(e1×e2)`, `t = norm(e1)` (re-orthogonalised).
   **Scale is never touched** (no triangle-area scaling). Output = position float3 + rotation float4 (**wxyz**) in scan-local space.
3. **Unity skinning data** does *not* come from `SkinnedMeshRenderer` (renderers are disabled). The project uses its *own* compute LBS; CPU builds
   28 bone matrices/frame as `skeletonRoot.worldToLocal · bone.localToWorld · bindPose[i]` from the Mecanim-driven Transform hierarchy and
   uploads them with `GraphicsBuffer.SetData`.
4. **Sorting** (from gsplat-unity, originally b0nes164/GPUSorting): per frame, gather deformed Gaussians to world space, key = view-space `z`
   (float, radix-transformed), **ascending** (far→near, Unity view space looks down −z), 8-bit LSD device radix sort, 4 passes, (key,payload) pairs,
   needs **wave/subgroup intrinsics**. Draw = instanced quads (128 splats/instance), premultiplied alpha `One, OneMinusSrcAlpha`, no depth write.
5. **Asset verdict: REAL ASSET OBTAINABLE.** The repo itself ships one complete rigged character (PLY + rig proxy FBX/OBJ + both baked blobs)
   and 12 MIT-licensed Rocketbox animation FBX clips, ~35 MB total, already in the local clone. License: avatar data **research/academic
   non-commercial only** (draft), clips MIT, code MIT. No releases/HF/Zenodo/Drive exist. Details in section 13.
6. **Three things in the Unity data/pipeline a faithful port must know about** (all measured, section 12):
   - The Unity-imported proxy mesh is an **x-mirrored copy** of the Mixamo FBX/OBJ (Unity FBX import flips x); the shipped bake binds the scan to that mirrored mesh.
     The unmirrored FBX ≡ the uploaded OBJ ≡ scan space × 100 exactly. Worst-case binding distance in asymmetric regions is ~2× worse with the mirrored mesh.
   - **1,132 of 40,000 Gaussians (2.83 %)** are baked degenerately (offset 0, identity frame) because Unity's `Vector3.normalized` returns the zero vector for
     `|v| ≤ 1e-5` and 12 % of the proxy triangles are slivers; at rest pose their runtime orientation is wrong and up to 3.7 cm off in position.
   - Unity animation is **Mecanim Humanoid retargeting** of Rocketbox `Bip01` clips onto the Mixamo humanoid avatar: there is no portable equivalent,
     so clips must be retargeted offline (Blender 4.1.1 is installed and can import both FBXs and export GLB; smoke-tested).

---

## 1. Provenance map: gsplat-unity vs virtual-human

| Piece | Origin (per file headers/README) | Key files |
|---|---|---|
| PLY importer | gsplat-unity, revised by Ke Li 2026 | `GS/Editor/GSplatImporter.cs` |
| `GsplatAsset` (CPU arrays) | gsplat-unity (Yize Wu) | `GS/Runtime/GsplatAsset.cs` |
| Buffer owner / draw submit | gsplat-unity (Yize Wu) | `GS/Runtime/GsplatRendererImpl.cs`, `GsplatRenderer.cs` |
| **`SetExternalBuffers` hook** (swap pos/rot buffers for deformed ones) | **added for virtual-human** | `GsplatRendererImpl.cs:L122-144` |
| Sort orchestration | gsplat-unity, derived from aras-p/UnityGaussianSplatting | `GS/Runtime/GsplatSorter.cs`, `GsplatSortPass.cs` |
| Radix sort kernels | b0nes164/GPUSorting (Thomas Smith 2024, MIT) | `GS/Runtime/Shaders/DeviceRadixSort.hlsl`, `SortCommon.hlsl` |
| `CalcDistance`/`InitPayload` kernels | aras-p `SplatUtilities.compute`, modified | `GS/Runtime/Shaders/Gsplat.compute` |
| Splat projection/covariance math | PlayCanvas engine chunks (MIT) + Yize Wu | `Gsplat.hlsl`, `GsplatCommon.hlsl` |
| Per-object draw shader | gsplat-unity, Ke Li 2025 | `Gsplat.shader` |
| **Unified cross-object gather + global sort + single draw** | **Ke Li 2026 (new in this fork)** | `GsplatCompositor.cs`, `GsplatGather.compute`, `GsplatComposite.shader` |
| URP render feature (sort pass) | aras-p/Yize Wu/Ke Li | `GS/Runtime/SRP/GsplatURPFeature.cs` |
| **Everything under `VH/`**: proxy ingestion, rig baker, skinning binding, skinning compute, skinning URP pass | **virtual-human specific (MIT, Ke Li)** | `VH/Runtime/*`, `VH/Runtime/Shaders/GsplatSkinning.compute`, `VH/Runtime/SRP/*`, `VH/Editor/*`, `VH/PythonTools/*` |
| Not in release | direct per-splat capsule skinning (`GsplatDirectSkinningBinding`), `RELEASE_NOTES.md` (referenced in README, absent) | README "How the rigging works", `IGsplatSkinningDispatcher.cs:L5-8` |

The vendored copy is authoritative (env.md). The wuyize25/gsplat-unity clone has a different layout and was not used.

---

## 2. How Gaussian data is represented

### 2.1 On disk (what the project consumes)

Binary little-endian PLY, **17 float32 properties / vertex = 68 bytes**, no SH rest coefficients. Header of the sample
(`VH/SampleFigures/LHM500M_woman-photoreal.ply`, 2,720,415 B = 415 B header + 40,000 × 68):

```
x y z  nx ny nz  f_dc_0 f_dc_1 f_dc_2  opacity  scale_0 scale_1 scale_2  rot_0 rot_1 rot_2 rot_3
```

- `nx,ny,nz` are all 0 in the shipped files (unused). All 25 `Dataset/ply/*.ply` have the identical layout and size (40,000 Gaussians each).
- Metric, Y-up, SMPL-X canonical A-pose, origin near chest height (Dataset/README). Measured sample bbox: x∈[−0.859,0.866], y∈[−1.348,0.427], z∈[−0.143,0.118] (metres);
  arms span ±0.86 m along x, height 1.775 m along y, body faces **+z**.
- Standard 3DGS conventions: `scale_*` = log-scale, `opacity` = logit, `f_dc_*` = raw SH-DC coefficients, `rot_0..3` = quaternion **(w,x,y,z)**.
  Sample stats: linear scale median 5.3 mm (min 0.08 mm, max 0.05 m — clamped), sigmoid(opacity) median 0.76, 65 % > 0.5, f_dc ∈ [−1.77,1.77].

### 2.2 Import (`GS/Editor/GSplatImporter.cs`, `[ScriptedImporter(1,"ply")]` :L14)

`OnImportAsset` :L37 parses the header into (name, offset, size) and reads in 4096-vertex batches, producing a `GsplatAsset`
(`GS/Runtime/GsplatAsset.cs`): `Positions Vector3[]`, `Scales Vector3[]`, `Rotations Vector4[]` (**wxyz**), `Colors Vector4[]`, `SHs Vector3[]`, `Bounds`, `SplatCount`, `SHBands`.

| PLY | → GsplatAsset | Line |
|---|---|---|
| x,y,z | `Positions` verbatim (**no axis flip, no handedness conversion**) | :L181-184 |
| scale_0..2 | `Scales = exp(scale_i)` | :L191-193 |
| rot_0..3 | `Rotations = normalize(Vector4(r0,r1,r2,r3))` (wxyz kept) | :L203 |
| f_dc_0..2 | `Colors.rgb` = **raw DC coefficient** (converted later) | :L205-210 |
| opacity | `Colors.a = sigmoid(opacity)` (`GsplatUtils.Sigmoid`, `GsplatUtils.cs:L15`) | :L209 |
| f_rest_* / sh_* | `SHs` + `SHBands = CalcSHBandsFromCoefficientCount(shCount)` | :L154-161 |

SH caveat (irrelevant for shipped assets, which have 0 `f_rest`): `CalcSHBandsFromCoefficientCount` (`GsplatUtils.cs:L25`) matches `(b+1)²−1` against the
number of `f_rest` *properties*, so a standard 45-property degree-3 file would resolve to band 0 as written (not executed). The sample scene also forces `SHDegree = 0`
(`VH/Editor/GsplatRigSampleScene.cs:L407`); README: "No per-Gaussian SH rotation under skinning".

Colour: `rgb = f_dc · SH_C0 + 0.5` with `SH_C0 = 0.28209479177387814` (`GsplatCommon.hlsl:L73`, applied at `Gsplat.shader:L127` and `GsplatGather.compute:L110`); project colour space is **Gamma**
(`ProjectSettings.asset: m_ActiveColorSpace: 0`), `GammaToLinear` defaults false, so colours are blended as-is (matches browser sRGB canvas).

### 2.3 GPU layout (`GsplatRendererImpl.CreateResources` `GS/Runtime/GsplatRendererImpl.cs:L85`)

All `GraphicsBuffer.Target.Structured`; **float3 buffers are tightly packed (stride 12)**:

| Buffer | Element | Content |
|---|---|---|
| `PositionBuffer` | float3 | rest position |
| `ScaleBuffer` | float3 | linear scale (post-exp) |
| `RotationBuffer` | float4 | quaternion **wxyz** |
| `ColorBuffer` | float4 | rgb = raw DC, a = opacity |
| `SHBuffer` | float3 × coeffs | only if bands > 0 |
| `OrderBuffer` | uint | sorted draw order (payload of the sort) |

`GsplatRenderer.SetBufferData` (`GsplatRenderer.cs:L29`) uploads the asset arrays. `SetExternalBuffers(pos, rot)` (`GsplatRendererImpl.cs:L122`) swaps the *active* position/rotation
buffers for externally owned ones (count must equal `SplatCount`), rebinds the property block, and recreates the sorter resource when the position buffer changes. Scale/colour/SH stay rest-pose.

**Minimal runtime data per Gaussian** (design input): scale (3), colour+opacity (4), binding (8 scalars = 32 B). Rest position and rotation are needed only at bake time; at runtime the deformation
kernel *replaces* both (even at rest pose).

---

## 3. How the proxy mesh / rig is generated and loaded

Three ingestion paths exist (README "How the rigging works"): (A) SMPL-X fit (source only, needs non-redistributable model, skipped), (B) **Poisson mesh + Mixamo auto-rig (the shipped sample's route)**, (C) direct per-splat capsule skinning (not released).

### 3.1 Path B, step by step

| Step | Where | What |
|---|---|---|
| 1. 3DGS human | external (LHM, single image → PLY) | sample PLY |
| 2. Poisson proxy | `VH/PythonTools/poisson_extract/extract_proxy.py` `main()` | load PLY (`load_gaussians`), per-Gaussian normals from the **smallest-scale axis** rotated by the quaternion (`gaussian_normals`), flip outward from centroid (`orient_normals_outward`), opacity filter ≥0.5, PyMeshLab screened Poisson (depth 9), coherent orientation, density trim, hole close, **keep largest component**, optional isotropic remesh (`--remesh-uniform 0.02`), then **vertices × `--output-scale 100` (m→cm, no translation)** → OBJ. Sample: `PythonTools/poisson_extract/mixamo_upload/LHM500M_woman-photoreal_proxy.obj` (4,201,976 B; **18,612 verts / 37,400 tris**). |
| 3. Rigging | **manual, external**: mixamo.com Auto-Rigger (Adobe login) | upload OBJ, place landmarks, download *FBX Binary, T-pose, no animation* → `VH/RigSources/LHM500M_woman-photoreal/LHM500M_woman-photoreal_proxy.fbx` (1,458,976 B, FBX 7.7). Not automatable. |
| 4. Unity FBX import | `.fbx.meta` | Humanoid avatar, `maxBonesPerVertex: 4`, `weldVertices: 1`, `useFileScale: 1` (cm→m ×0.01), `bakeAxisConversion: 0`, `skinWeightsMode: 0`, `meshCompression: 0`. Unity flips x on import (section 12.4). |
| 5. Rig proxy bake | `MixamoFbxToRigProxy.BakeInternal` `VH/Editor/MixamoFbxToRigProxy.cs:L150` | reads `smr.sharedMesh.vertices/triangles/GetBonesPerVertex/GetAllBoneWeights/bindposes` (:L160-168), `ExtractSkeleton` (:L188, L319), `RigProxyBlob.Pack` (:L198) → `.rigproxy.bytes` + `.rigproxy.asset`. Also stores Humanoid `Avatar`. |
| 6. proxy→scan alignment | `MixamoProxyAligner.TrySolve` `VH/Editor/MixamoProxyAligner.cs:L47` | fits the imported mesh back onto the uploaded OBJ (÷100): Umeyama similarity (:L123, det=+1 enforced :L224-226) and seeded ICP (:L96-97); stored as `ProxyToScanHint` in the asset. Fallback: PCA (`ComputeProxyToScanByPca` `GsplatRigBakerWindow.cs:L369`, documented as biased). |
| 7. Gaussian binding | `GsplatRigBakerWindow.BakeInternal` (`:L197`) | section 4 |

### 3.2 `.rigproxy.bytes` layout (`VH/Runtime/RigProxyAsset.cs`, `RigProxyBlob`, Pack :L137, Unpack :L236)

Little-endian; header 32 B: `u32 magic 0x50525347 ("GSRP")`, `i32 version` (1; 2 only when a facial expression basis exists, :L91-94), `i32 vertexCount`, `i32 triangleIndexCount (=3·tris)`, `i32 boneCount`, `i32 weightEntryCount`, `i32 reserved[2]`. Then, each section 4-byte aligned:

```
float32 vertices[V*3]            // proxy-mesh-local, bind pose
int32   triangleIndices[TI]      // 3 per triangle, Unity winding
uint8   bonesPerVertex[V]        // padded to multiple of 4
int32   weightBoneIndex[W]       // flat, per-vertex runs of length bonesPerVertex[v]
float32 weightBoneWeight[W]      // Unity sorts by weight desc; sums to 1
float32 bindPoses[B*16]          // Unity Matrix4x4, COLUMN-major: index = col*4+row (:L212-215)
```

Sample (verified by parsing): V=18,612, TI=112,200, B=28, W=42,870, total 1,035,540 B = sum of sections. `bonesPerVertex` histogram {1:4656, 2:5765, 3:6080, 4:2111} (max 4, 0 vertices >4), weight sums 1.0±2e-7, max bone index 27.
Only the **28 skinned bones** are stored (`BoneNames`, `ParentIndices` in `.rigproxy.asset` YAML; the FBX has 33 LimbNodes incl. leaf `*_End`/`HeadTop_End`/`*HandIndex4`). Skeleton = Mixamo "standard" rig; fingers: only `HandIndex1-3` per hand.
The proxy T-pose: hands at x≈±0.73 m, hips y≈−0.28.

Bone list (index: name, parent): 0 LeftUpLeg(18), 1 LeftLeg(0), 2 LeftFoot(1), 3 RightUpLeg(18), 4 RightLeg(3), 5 RightFoot(4), 6 RightToeBase(5), 7 RightHandIndex1(10), 8 RightHandIndex2(7), 9 RightHandIndex3(8),
10 RightHand(17), 11 LeftToeBase(2), 12 LeftHand(15), 13 LeftHandIndex1(12), 14 LeftHandIndex2(13), 15 LeftForeArm(20), 16 LeftHandIndex3(14), 17 RightForeArm(19), 18 Hips(−1), 19 RightArm(22), 20 LeftArm(27),
21 Head(26), 22 RightShoulder(24), 23 Spine1(25), 24 Spine2(23), 25 Spine(18), 26 Neck(24), 27 LeftShoulder(24). (all prefixed `mixamorig:`; order = `SkinnedMeshRenderer.bones` = bindpose order.)

### 3.3 Runtime loading

`GsplatSkinningBinding.ReloadIfNeeded` (`VH/Runtime/GsplatSkinningBinding.cs:L307`) unpacks both blobs, **checks `splatCount == rig.TriIndex.Length` (:L340)**, then `UploadStaticBuffers` (:L364): rest verts (float3), weights padded to exactly 4 per vertex into `uint idx4[V*4]` + `float w4[V*4]`
(:L377-398, unused slots index 0 / weight 0 — **influences beyond 4 would be dropped without renormalising**, not an issue for the sample), triangles (uint), binding arrays `_GTriIndex/_GBary/_GNormalOffset/_GRestLocalRot`, outputs `DeformedPositions/DeformedRotations/_deformedVerts`, 28 bone matrices.

---

## 4. How Gaussians are bound to proxy triangles (offline baker)

`GsplatRigBakerWindow.BakeInternal` (`VH/Editor/GsplatRigBakerWindow.cs:L197-L350`), menu `GsplatRig → Bake Rigged Gsplat` (:L30). Per Gaussian `i` (loop :L228):

1. `proxyToScan` = `RigProxy.ProxyToScanHint` if present, else manual, else PCA (:L205-210); `scanToProxy = proxyToScan.inverse` (:L211); `rotProxyFromScan = QuaternionFromMatrix(scanToProxy)` (:L212, `QuaternionFromMatrix` :L534 = `LookRotation(col2.normalized, col1.normalized)`).
2. `p_proxy = scanToProxy · p_scan` (:L244).
3. **Nearest triangle** by *clamped* closest-point distance (`ClosestPointOnTriangle` :L559, Ericson RTCD) over a uniform grid of triangle centroids (`BuildTriangleGrid` :L694; ~3 tris/cell, ≤200k cells :L722; ring search until ≥24 candidates and ring distance > best, :L640,681); brute-force fallback (`:L779`).
4. **Tangent frame** of the host triangle (`BuildTangentFrame` :L545-557):
   `n = norm(cross(e1,e2))`, `t = norm(e1)`, `b = norm(cross(n,t))`, `t = norm(cross(b,n))`, with `e1=v1−v0`, `e2=v2−v0` (vertex order = index order in the triangle list).
5. **Barycentric** = exact in-plane least-squares projection, *unclamped* (:L265-291): `d11=e1·e1, d22=e2·e2, d12=e1·e2, d1p=e1·(p−v0), d2p=e2·(p−v0)`,
   `det = d11·d22 − d12²`, `u = (d22·d1p − d12·d2p)/det`, `v = (d11·d2p − d12·d1p)/det` (float32; degenerate `|det|≤1e-20` → clamped closest-point bary). `w = 1−u−v` is implicit. Values outside [0,1] are intentional (hair, cloth, fingers).
6. **Signed normal offset**: `off = dot(p_proxy − (v0 + e1·u + e2·v), n)` (:L293-297). Proxy units (metres for the sample).
7. **Rest orientation**: `restLocalRot = inverse(LookRotation(n, t)) · (rotProxyFromScan · q_gaussian)` (:L305-311). PLY quaternion wxyz is reordered to Unity xyzw first (:L306). I.e. the Gaussian's orientation expressed in the host triangle's rest frame.
8. Written through `RiggedGsplatBlob.Pack` (:L324) as section 5.

Quality logging only (`dist` :L300) — the distance is not stored. **No scale handling** of any kind, and nothing is weighted by multiple triangles: one Gaussian ↔ exactly one triangle.

---

## 5. EXACT baked binding data (`.riggedgsplat.bytes`)

Writer `RiggedGsplatBlob.Pack` (`VH/Runtime/RiggedGsplatAsset.cs:L85`), reader `Unpack` (:L137), spec comment :L62-83, constants :L59-60.

```
offset  size      field
0       u32       magic = 0x47525347  (bytes 47 53 52 47 = "GSRG")
4       i32       version = 1
8       i32       gaussianCount N
12      i32[5]    reserved (0)
32      u32[N]    triIndex                 // triangle id (NOT vertex id). Triangle t = indices[3t..3t+2] of the rigproxy triangle list
32+4N   f32[2N]   bary  (u0,v0,u1,v1,...)  // w = 1-u-v implicit; UNCLAMPED (|u| up to 522 in sample)
32+12N  f32[N]    normalOffset             // signed, along rest normal n=normalize(cross(e1,e2))
32+16N  f32[4N]   restLocalRot (x,y,z,w)   // unit quaternion, Unity xyzw order, Hamilton product
total = 32 + 32N
```
Sample header bytes (verified): `47535247 01000000 409c0000 00000000 00000000 ...` (0x9c40 = 40000). File 1,280,032 B = 32 + 32·40000 ✓ (parsed to end, no trailing bytes).

Asset-level data **not in the blob** (`*.riggedgsplat.asset` YAML): `GaussianCount`, `ProxyToScanTransform` (4×4, YAML names `eRC` = row,col), references to source PLY asset and rig proxy. Sample:

```
ProxyToScan (row-major as e00..e33):
 [ 0.9979578     0.007437238  0.00469504  -0.0014446443 ]
 [-0.007431836   0.9979682   -0.0011647   -0.00051393156]
 [-0.004703587   0.0011296918 0.99798477  -0.00031708545]
 [ 0 0 0 1 ]          // uniform scale 0.99800, otherwise a rotation (orthonormality err 8e-8)
```
NB: it is a *similarity*, not rigid (comments say "rigid"); the kernel applies the full matrix to positions but only the rotation part (`_ProxyToScanRot`) to quaternions, and Gaussian scales are not scaled by 0.998 (0.2 % inconsistency, negligible, but note for exact parity). With the *unmirrored* FBX the exact transform is simply ×0.01 (section 12.4).

Reference fixtures (parsed from the shipped blobs; usable as unit-test vectors; `v0` is the first vertex of the host triangle in the rigproxy blob):

| G | triIdx | tri vertex idx | (u,v) | normalOffset | restLocalRot xyzw | PLY pos |
|---|---|---|---|---|---|---|
| 0 | 31310 | 15806,15805,15807 | (0.15601629, 0.8210371) | 0.00362059893 | (−0.8042465, 0.5710309, −0.16461223, −0.003792836) | (−0.12106422, −1.1540052, −0.070095465) |
| 1 | 25002 | 12584,12586,12551 | (0.75099415, 0.19041847) | −0.00446213875 | (0.079383336, −0.3432477, −0.082266934, 0.9322616) | (0.13312528, −0.5878538, 0.030038182) |
| 2 | 5627 | 2938,2937,2931 | (0.99370396, −0.10617616) | −0.0166462306 | (−0.8025271, −0.08661115, −0.05244378, 0.58796126) | (0.36174706, −0.011493387, −0.06813858) |

(The float64 reconstruction of G0-G2 reproduces the PLY positions to ≤2.4e-8 m.)

Distribution (sample): 15,339 of 37,400 triangles used; |u|,|v| up to 522 (36.7 % of Gaussians outside the simplex); normalOffset ∈ [−0.046, 0.061], |off| p50/p90/p99 = 6.3/18.4/31.7 mm; 1,132 have off == 0.0 exactly (section 14.2).

---

## 6. Runtime data flow into the compute pass

`GsplatSkinningBinding` (`VH/Runtime/GsplatSkinningBinding.cs`, `[ExecuteAlways]`, needs a `GsplatRenderer` on the same GameObject).

### 6.1 Buffers bound to the kernels (`DispatchImmediate` :L607, `EncodeDispatch` :L641)

| Kernel | Inputs | Outputs |
|---|---|---|
| `DeformProxyVertices` | `_RestVertices` float3[V], `_BoneIndices4` uint[4V], `_BoneWeights4` float[4V], `_BoneMatrices` float4x4[B], `_VertexCount` | `_DeformedVerts` float3[V] |
| `DeformGaussians` | `_Triangles` uint[3T], `_GTriIndex` uint[N], `_GBary` float2[N], `_GNormalOffset` float[N], `_GRestLocalRot` float4[N] (xyzw), `_DeformedVertsIn`, `_GaussianCount`, `_ProxyToScan` float4x4, `_ProxyToScanRot` float4 (xyzw, from `QuaternionFromMatrix` :L680 of the matrix) | `_DeformedPositions` float3[N], `_DeformedRotations` float4[N] (**wxyz**) |

Thread groups: 64 threads (`[numthreads(64,1,1)]` :L222, :L417); dispatch `⌈V/64⌉` and `⌈N/64⌉` (sample: 291 and 625 groups).

### 6.2 Where the "Unity skinning data" comes from

- **Not** `SkinnedMeshRenderer.GetVertexBuffer`/`BakeMesh`. The sample builder *disables every SMR* ("We want to see the Gaussians, not the mesh", `GsplatRigSampleScene.cs:L412-415`). The proxy is skinned by the project's own compute kernel from the static blob.
- Pose source: an `Animator` on the instantiated FBX prefab (`SkeletonRoot`), `applyRootMotion = false` (`GsplatRigSampleScene.cs:L427`), humanoid retargeting of Rocketbox clips (section 10).
- Per frame, `UpdateBoneMatrices` (`GsplatSkinningBinding.cs:L538-567`, called from `LateUpdate` :L203/L222):
  `m[i] = SkeletonRoot.worldToLocalMatrix · bone[i].localToWorldMatrix · bindPose[i]` (:L559-564), bones matched **by name** under `SkeletonRoot` (`BuildBoneTransformMap` :L569; missing → identity), uploaded with `m_boneMatrices.SetData` (:L566). 28×64 B = 1.8 KB/frame.
  `bindPose[i]` is Unity's `mesh.bindposes[i]` = `bone.worldToLocal · renderer.localToWorld` at bind. Result maps a bind-pose proxy-local vertex to its deformed position in skeleton-root-local space; assumes the SMR sits at identity under the skeleton root (true in the sample).
  Because `rootInv` divides out `SkeletonRoot`'s transform, **root motion is cancelled** (walks play on the spot; README says so).
- With `ForceIdentityBones` or missing skeleton, identity matrices are uploaded → rest pose (:L540-545).
- `ProxyToScan` is applied as the **last step of `DeformGaussians`**, so outputs are in PLY/scan-local space; the renderer's GameObject transform (yaw, ground offset) is applied later in gather (`_GObjectToWorld`).
- Skinning mode `SkinningMode.LBS` (default & sample, `GsplatRigSampleScene.cs:L452`) or `DQS` (`DQS_ENABLED` multi_compile, :L102-186, :L244-305). Expression blendshapes (`EXPRESSION_ENABLED`, :L206-240) exist only for the SMPL-X path; the Mixamo blob is version 1 with no expression basis. **Both DQS and expression are out of scope for the port.**
- Dispatch timing: `DispatchInLateUpdate=false` in the sample → the URP `GsplatSkinningURPFeature` pass at `RenderPassEvent.BeforeRendering` (`GsplatSkinningURPFeature.cs:L93-94`) calls `DispatchOnCommandBuffer` (:L252) and adds a GPU fence so the sort (AfterRenderingOpaques) reads finished data.

---

## 7. The deformation math, exactly (`VH/Runtime/Shaders/GsplatSkinning.compute`)

### 7.1 `DeformProxyVertices` (LBS path, :L222-L320 with `#else` branch :L305-319)

```
for v in vertices:                       // uses base = v*4
  acc = (0,0,0)
  for k in 0..3:                         // [unroll]
      w = BoneWeights4[4v+k];  if w == 0: continue
      acc += w * ( BoneMatrices[ BoneIndices4[4v+k] ] * float4(vRest,1) ).xyz   // column-vector mul, M[row][col]
  DeformedVerts[v] = acc                 // weights are NOT renormalised
```
`BoneMatrices` is `StructuredBuffer<float4x4>` fed from C# column-major `Matrix4x4[]` (Unity compiles with column-major packing, so `mul(M,v)` is the ordinary `M·v`). In WGSL (`mat4x4f` is column-major) upload the same 16 floats unchanged and use `M * vec4f(v,1)`.

### 7.2 `DeformGaussians` (:L417-L462), verbatim semantics

```
t  = GTriIndex[g]
i0,i1,i2 = Triangles[3t], Triangles[3t+1], Triangles[3t+2]
v0,v1,v2 = DeformedVerts[i0], [i1], [i2]
e1 = v1 - v0;  e2 = v2 - v0
n   = normalize(cross(e1, e2))            // :L437   (HLSL normalize: NO epsilon guard)
tan = normalize(e1)                       // :L438
bin = normalize(cross(n, tan))            // :L439
tan = normalize(cross(bin, n))            // :L440   re-orthogonalise
(u,v) = GBary[g];  off = GNormalOffset[g]
pProxy = v0 + e1*u + e2*v + n*off         // :L445-447
tfq    = LookRotationQuat(n, tan)         // :L450  (forward = n, up = tan)
rProxy = qmul(tfq, GRestLocalRot[g])      // :L452  xyzw, Hamilton product a*b
pScan  = (ProxyToScan * float4(pProxy,1)).xyz       // :L456
rScan  = qmul(ProxyToScanRot, rProxy)               // :L457
DeformedPositions[g] = pScan
DeformedRotations[g] = float4(rScan.w, rScan.x, rScan.y, rScan.z)   // :L460-461  WXYZ for the draw shader
```
`LookRotationQuat(f, up)` (:L368-414) = Unity `Quaternion.LookRotation` in matrix form: `f=norm(f); r=norm(cross(up,f)); u=cross(f,r)`; matrix columns `[r,u,f]`; then Shoemake matrix→quaternion with branch-by-largest-diagonal (xyzw).
Simplification that was verified numerically (section 12.2): since `n ⟂ tan`, the frame columns are exactly `[t×n, t, n]`, and

```
R_gaussian = R(ProxyToScanRot) · [t×n | t | n] · R(restLocalRot)        // quaternion product order == matrix product order
Σ_world    = R diag(s²) Rᵀ                                               // scale s untouched, straight from the PLY (exp'd)
```

What the kernel does **not** do: no scaling of Gaussians with triangle stretch, no normalisation of the output quaternion (inputs are unit so drift is ~1e-7), no SH rotation, no NaN/degenerate-triangle guard, no handling of `|bary|` blow-up beyond float32.

### 7.3 Quaternion conventions inside the pipeline (easy to get wrong)

| Where | Order | Note |
|---|---|---|
| PLY `rot_0..3`, `GsplatAsset.Rotations`, draw-shader `_RotationBuffer`, kernel output `_DeformedRotations` | **w,x,y,z** | `QuatToMat3` takes `(w,x,y,z)` in `.xyzw` slots (`GsplatCommon.hlsl:L21`) |
| `restLocalRot` blob + `_GRestLocalRot` + all kernel-internal math (`qmul`, `LookRotationQuat`, `_ProxyToScanRot`) | **x,y,z,w** | Unity `Quaternion` field order |
| glTF / three.js / Unity C# `Quaternion` | x,y,z,w | |

`QuatToMat3` + `CalcCovarianceMatrix` (`GsplatCommon.hlsl:L21-52`) are algebraically the textbook rotation matrix: `M = R·S`, `Σ = M Mᵀ` (verified).

---

## 8. Sorting and drawing (gsplat-unity part), per-frame flow

Default settings: `GsplatSettings.CompositeSort = true` (`GsplatSettings.cs:L90`), `SplatInstanceSize = 128` (:L95). The composite path is what the sample runs.

### 8.1 Order of operations in one frame

1. (CPU) `Animator` → `GsplatSkinningBinding.LateUpdate` → bone matrices `SetData`; `Impl.SetExternalBuffers(DeformedPositions, DeformedRotations)` (:L230).
2. (GPU, URP `BeforeRendering`) `DeformProxyVertices` → `DeformGaussians` → fence.
3. (CPU, `Update`) `GsplatRenderer.Update` (`GsplatRenderer.cs:L56`) → `Impl.Render` (`GsplatRendererImpl.cs:L179`) → `GsplatCompositor.SubmitFrame`; the first registered renderer issues the single draw in `Draw()` (`GsplatCompositor.cs:L200`, `Graphics.RenderMeshPrimitives` :L253; instance count `⌈total/128⌉`).
4. (GPU, URP `AfterRenderingOpaques`, `GsplatURPFeature.cs:L82`) `GsplatSorter.DispatchSort` (`GsplatSorter.cs:L183`) → `GsplatCompositor.Dispatch` (`:L140`):
   a. per renderer: **`GatherSplats`** (`GsplatGather.compute:L100`, 256 threads): reads the *active* (deformed) pos/rot + rest scale/colour; `color.rgb = dc*SH_C0+0.5` (:L110); `posWS = _GObjectToWorld · center`;
      world covariance `Σ = (M3·R·S)(M3·R·S)ᵀ` (:L130-131, `PackCovariance` → upper triangle `covA=(xx,xy,xz)`, `covB=(yy,yz,zz)`); writes `_DstPosition/_DstCovA/_DstCovB/_DstColor` at `_GDstOffset`.
   b. `InitPayload` only on first use / layout change (`:L184`, `m_payloadInitialised`), filling `order[i]=i`.
   c. **`GsplatSortPass.Dispatch`** (`GsplatSortPass.cs:L157`), `MatrixMv = camera.worldToCameraMatrix` (`GsplatCompositor.cs:L192`).
5. (GPU, transparent queue) draw with `GsplatComposite.shader`.

### 8.2 The sort (`GS/Runtime/GsplatSortPass.cs`, `Shaders/Gsplat.compute`, `DeviceRadixSort.hlsl`, `SortCommon.hlsl`)

- Keys: float. Kernel `CalcDistance` (`Gsplat.compute:L45-54`, 1024 threads): `pos = _PositionBuffer[ payload[idx] ]; key[idx] = (MatrixMV · float4(pos,1)).z`. The payload is the previous frame's order (it persists; only re-initialised when the layout changes), so each frame re-keys the previous permutation.
- Keys are mapped to orderable uints with Michael Herf's trick (`SortCommon.hlsl:L115` `FloatToUint`: `mask = -(asuint(f)>>31) | 0x80000000; return asuint(f) ^ mask`).
- Config: `KEY_FLOAT`, `PAYLOAD_UINT`, `SHOULD_ASCEND`, `SORT_PAIRS` keywords (`Gsplat.compute:L11-12`, `GsplatSortPass.cs:L138-139`). **Ascending on view-space z**: visible points have z<0 (Unity view space looks down −z; the vertex shader culls `centerView.z > 0`, `Gsplat.shader:L73`, `GsplatComposite.shader:L85`), so order is **farthest → nearest = back-to-front**, matching premultiplied blending.
- Algorithm: 8-bit LSD radix, 4 passes, partition size 3,840 keys (`PART_SIZE` `SortCommon.hlsl:L29`, `k_deviceRadixSortPartitionSize = 3840` :L29 of SortPass), `RADIX=256`. Dispatch sequence (`GsplatSortPass.cs:L175-218`): `CalcDistance ⌈N/1024⌉` → `InitDeviceRadixSort (1 group of 1024, clears 4×256 global histogram)` → 4× [`Upsweep ⌈N/3840⌉`, `Scan 256 groups`, `Downsweep ⌈N/3840⌉`], ping-pong Key/Alt and Payload/AltPayload, 4 swaps ⇒ result ends in the original buffers. Scratch: `AltBuffer`, `AltPayloadBuffer` (N uints each), `PassHistBuffer` (⌈N/3840⌉·256), `GlobalHistBuffer` (1024).
- **Requires wave intrinsics** (`#pragma require wavebasic/waveballot`, `Gsplat.compute:L25-26`; `WavePrefixSum`, `WaveReadLaneAt`, `WaveActiveBallot`, `WaveGetLaneCount`, atomics on groupshared). Hence README's "D3D12 or Vulkan only". For N=40,000: 11 thread blocks.

### 8.3 Draw path (per splat; `GsplatComposite.shader` + `Gsplat.hlsl`)

- Geometry: one mesh of 128 quads (`GsplatSettings.CreateMeshInstance` :L109-125): vertex `(±1,±1, asfloat(i))`; `order = instanceID*128 + asuint(vertex.z)`; `id = _GOrderBuffer[order]` (`GsplatComposite.shader:L67-80`, discard if `order >= _SplatCount`).
- `InitCenterWS` (:L85): `centerView = V·pos` (positions already world), cull `z > 0`, `proj = P·centerView`, `proj.z` clamped to `±w`.
- `InitCorner` (`Gsplat.hlsl:L39-110`): `focal = ScreenParams.x · P[0][0]` (:L49); `J = [[J1,0,J2.x],[0,J1,J2.y],[0,0,0]]`, `J1=focal/v.z`, `J2=−J1/v.z·v.xy`; `T=J·W` (W = 3×3 of modelView = view matrix); `cov2 = T Σ Tᵀ`;
  `d1 = cov00+0.3`, `d2 = cov11+0.3` (:L72-74); `mid, radius, λ1=mid+radius, λ2=max(mid−radius, 0.1)`; `vmin = min(1024, min(W,H))`; `l = 2·min(sqrt(2λ), vmin)`; cull if `l1<2 && l2<2`; frustum cull; axes `v1 = l1·normalize(off, λ1−d1)`, `v2 = l2·(dv.y,−dv.x)`; `offset_clip = (uv.x·v1 + uv.y·v2) · (proj.w / ScreenParams.xy)`.
  **Derived subtlety (my reading, not executed):** `focal` is W·P00 (twice the true pixel focal W/2·P00), so this "pixel" space is 2× scaled: the `+0.3` dilation is really 0.075 px², and the quad half-extent `l/2 = sqrt(2λ)` equals `2√2·σ_px`, which makes `exp(−4|uv|²)` equal to the standard `exp(−r²/2σ²)`. A port that uses true-pixel covariance must rescale (this math is PlayCanvas's; reusing PlayCanvas/Spark's renderer avoids re-deriving it).
- `ClipCorner` (:L112-118): `clip = min(1, sqrt(−ln(1/255/α))/2)` shrinks the quad and `uv` to the 1/255 alpha threshold.
- Vertex out: `proj + (offset.x, _ProjectionParams.x·offset.y, 0, 0)` (y flip for render-target convention). Fragment (`GsplatComposite.shader:L143-150`): `A=dot(uv,uv); if A>1 discard; α = exp(−4A)·color.a; if α<1/255 discard; return (rgb·α, α)`.
- State: `ZWrite Off`, `Blend One OneMinusSrcAlpha` (premultiplied), `Cull Off` (`GsplatComposite.shader:L24-26`).
- Non-composite fallback (`CompositeSort=false`): per object, sort uses the object's *deformed* position buffer with `MV = V·M` (`GsplatSorter.cs:L201-220`) and `Gsplat.shader` draws directly — same math, different packaging.

---

## 9. What is offline vs per-frame (concrete functions / dispatches)

### 9.1 Offline / editor-side (never per frame)

| # | Operation | Function | File |
|---|---|---|---|
| 1 | 3DGS human reconstruction | external LHM | (Dataset README) |
| 2 | PLY → `GsplatAsset` | `GsplatImporter.OnImportAsset` | `GS/Editor/GSplatImporter.cs:L37` |
| 3 | Poisson proxy OBJ | `extract_proxy.main`, `poisson_reconstruct`, `remesh_uniform` | `VH/PythonTools/poisson_extract/extract_proxy.py` |
| 4 | Auto-rig | Mixamo web (manual) | — |
| 5 | FBX import (Humanoid, 4 bones, weld, x-flip, cm→m) | Unity `ModelImporter` | `*.fbx.meta` |
| 6 | Rig proxy blob | `MixamoFbxToRigProxy.BakeInternal` | `VH/Editor/MixamoFbxToRigProxy.cs:L150` |
| 7 | proxy→scan solve | `MixamoProxyAligner.TrySolve` (Umeyama+ICP) | `VH/Editor/MixamoProxyAligner.cs:L47` |
| 8 | **Gaussian↔triangle binding** | `GsplatRigBakerWindow.BakeInternal` | `VH/Editor/GsplatRigBakerWindow.cs:L197` |
| 9 | Animation import (Humanoid, loopTime+loopPose) | `RocketboxAnimations` editor + `fetch_rocketbox_animations.py` | `VH/Editor/RocketboxAnimations.cs` |
| 10 | Sample scene + AnimatorControllers | `GsplatRigSampleScene.BuildScene` | `VH/Editor/GsplatRigSampleScene.cs` |

### 9.2 Load time (once)

`GsplatRenderer.OnEnable/SetBufferData` (`GsplatRenderer.cs:L39-47`), `GsplatSkinningBinding.ReloadIfNeeded/UploadStaticBuffers/ResolveKernels` (`:L307/L364/L263`).

### 9.3 Every frame

1. Mecanim evaluation (CPU) — Unity-only.
2. `UpdateBoneMatrices` (28 matrices, CPU) + `SetData`.
3. GPU `DeformProxyVertices` (⌈18,612/64⌉ = 291 groups) → `DeformGaussians` (⌈40,000/64⌉ = 625 groups).
4. GPU `GatherSplats` (157 groups) → `CalcDistance` (40) → `InitDeviceRadixSort` (1) → 4×(`Upsweep` 11, `Scan` 256, `Downsweep` 11).
5. Instanced draw: ⌈40,000/128⌉ = 313 instances × 128 quads.

No CPU readback of Gaussian data anywhere; only the 28 bone matrices cross CPU→GPU.

---

## 10. Animation source (relevant to the web port)

- Unity plays **Rocketbox** clips (`VH/RocketboxAnimations/*.fbx`, 12 clips) on the Mixamo humanoid via Mecanim (muscle-space retargeting). `GsplatRigSampleScene.k_cycle` (:L64-76): `m_wave_01, f_idle_neutral_02, m_claphands_01, m_idle_look_around_02, f_gestic_laugh_loud, m_gestic_shrug_02, f_dancing_cool, f_gestic_listen_accept_03, m_gestic_talk_relaxed_02, f_cheer_03` (+ two walk clips not in the cycle). 6 s per state, 0.5 s cross-fades, `loopTime+loopPose`.
- Measured properties of the clips (FBX 7.7 parse): each has 124 `Bip01*` nodes (3ds Max Biped incl. ~40 face/finger bones), 247 curve-nodes, **dense keys at 30 fps**; key counts: cheer 96, dancing 708, laugh 311, listen 133, idle 338, clap 211, shrug 104, talk 823, look-around 136, wave 86, walk 33/37 (≈ 2.9 s wave … 27 s talk, 23.6 s dance). Unit cm, Y-up in file (original Z-up), body faces +z, left at +x (same RH convention as the Mixamo FBX).
- **Rest poses differ**: Rocketbox biped rest = relaxed, arms down (L hand ≈ (0.27, −0.03, 0.88) in Blender Z-up) whereas the Mixamo proxy is a **T-pose** (hands at x≈±0.73). Any offline retargeting must account for this (per-bone rest-relative world rotation transfer). Body bone mapping is straightforward: Pelvis→Hips, Spine/Spine1/Spine2→same, Neck, Head, L/R Clavicle→Shoulder, UpperArm→Arm, Forearm→ForeArm, Hand, Thigh→UpLeg, Calf→Leg, Foot, Toe0→ToeBase.
- Unity cancels root motion (`rootInv`), so hip height/translation inside the body hierarchy still animates; only the SkeletonRoot transform is removed. `_static` clips are in place; `_xy` walks rely on root-motion extraction.
- No other animation source is bundled; Mixamo clips are not redistributable and need an Adobe login.

---

## 11. Unity vs glTF/WebGPU conventions (for a faithful port)

| Topic | Unity (this repo) | glTF / three.js / WebGPU | Porting consequence |
|---|---|---|---|
| Handedness | Left-handed, Y-up, +Z forward | **Right-handed**, Y-up, asset front = +Z, camera looks −Z | Cross products are numerically identical but geometrically flip under a mirror. **Never mirror only part of the data**: proxy, bones, animation, Gaussians, baked binding must share one frame. |
| FBX→engine | Unity importer **negates x** (measured; FBX left bones +x → Unity blob −x) and reverses winding | FBX/Blender/glTF keep RH, no flip | Author in RH (FBX-native) and re-bake binding there (section 12.4). |
| PLY (3DGS) | imported verbatim (no flip) → displayed mirrored vs. an RH viewer | RH viewers show it un-mirrored (SMPL-X: left = +x, facing +z) | Web shows the anatomically correct person; Unity shows the mirror image. Expected, document it. |
| Quaternion storage | `Quaternion` (x,y,z,w); PLY/GPU render buffers (w,x,y,z) | (x,y,z,w) | See 7.3. Quaternion→matrix formula is the standard one in both. |
| Quaternion mirror rule (if ever needed) | — | mirror in x: `(x,y,z,w)→(x,−y,−z,w)`, position `(−x,y,z)` | only for a deliberate Unity-space reproduction. |
| Matrices | `Matrix4x4` column-major memory, `m[row,col]`; HLSL `mul(M,v)`, column vectors | glTF column-major, WGSL `mat4x4f` column-major, `M*v` | Bone matrices and bindposes: same 16 floats, no transpose. |
| Buffer packing | `StructuredBuffer<float3>` stride **12 B** | WGSL `array<vec3f>` stride **16 B** | Use flat `array<f32>` (×3) or pad to `vec4f`. Affects positions, scales, bary (vec2 stride 8 is OK). |
| Units | FBX cm → Unity ×0.01 → m; PLY metres | glTF metres | Blender's GLB export kept a 0.01 scale on the armature/mesh nodes (observed) → apply scale in the bake tool so IBMs, joint TRS and vertices are in metres. |
| 3DGS fields | `scale = exp(scale_i)`, `α = sigmoid(opacity)`, `rgb = f_dc·0.28209479+0.5`, quaternion normalised at import, `nx..nz` ignored | same | — |
| Clip space / depth | reversed-Z on D3D12/Vulkan, `_ProjectionParams.x` flips y for RT | NDC z∈[0,1], y up in NDC, framebuffer origin top-left | Re-derive splat projection from your own camera matrices. |
| Sort key | view-space z ascending (negative, far first) | depends on camera convention | For RH cameras looking −z the same expression works; otherwise sort by depth descending. |

---

## 12. Verification performed (numerical facts on the shipped data)

Independent float64 NumPy port of `DeformGaussians` with **identity bone matrices** (rest pose), fed only from the shipped blobs/PLY/asset YAML:

### 12.1 Blob integrity
Both blobs parse to the last byte (`rigproxy` 1,035,540 B; `riggedgsplat` 1,280,032 B = 32+32·40000). Mesh: 18,612 verts / 37,400 tris, no index out of range, 0 zero-area triangles (min |cross| = 2.3e-11), 0 vertices with >4 influences.

### 12.2 Rest-pose round trip (position + orientation + covariance)
Reconstruct `pProxy → pScan = ProxyToScan·pProxy` and `R = R_p2s · [t×n|t|n] · R(restLocalRot)`:

| Subset | count | max pos error | max rot-matrix error | max covariance error |
|---|---|---|---|---|
| hosted by triangles with `|e1×e2| > 1e-5` | 38,868 | **3.0e-7 m** (mean 1.7e-8) | **9.2e-7** | **2.6e-9** |
| hosted by slivers (`|e1×e2| ≤ 1e-5`) | 1,132 | 3.7e-2 m | ≈2 (orientation wrong) | 1.6e-3 |

This independently confirms the frame construction, the quaternion composition order (quaternion product `a*b` ↔ matrix `R(a)R(b)`), the `[t×n,t,n]` form of `LookRotation(n,t)`, xyzw storage, and that PLY rotation is wxyz. It is the strongest available ground truth in the absence of a Unity run.

### 12.3 Sliver-triangle defect (root cause proven)
`Vector3.normalized` returns the **zero vector when magnitude ≤ 1e-5** (Unity `kEpsilon`). `BuildTangentFrame` (`GsplatRigBakerWindow.cs:L550`) therefore yields `n = 0` for triangles with `|e1×e2| ≤ 1e-5` (= area ≤ 5e-6 m², about 5 mm²; 4,471 of 37,400 triangles = 12 %, hosting 1,132 Gaussians = 2.83 %). For those the baker writes `normalOffset = 0` **exactly** (1,132 exact zeros, all on sliver hosts) and `restLocalRot = R_scanToProxy · q_ply` (identity frame; verified to 3.6e-7).
At runtime `DeformGaussians` uses HLSL `normalize` (no epsilon) on the tiny-but-nonzero cross product, so these Gaussians get a real frame multiplied onto a pre-rotated quaternion ⇒ wrong orientation (mean matrix error 1.46) and missing normal offset (position errors 1 mm-3.7 cm; 991 >1 mm, 201 >1 cm). Also large |bary| (up to 522) from float32 `det` cancellation. This is an existing artifact of the Unity sample, not a port bug.

### 12.4 The Unity proxy mesh is x-mirrored relative to the Mixamo FBX / OBJ
- FBX geometry parse (binary FBX 7.7, GlobalSettings Y-up, UnitScaleFactor 1 = cm): 18,612 vertices, **identical to the uploaded OBJ to 0.0** (index-wise). OBJ bbox x[−85.42,85.35] cm.
- `rigproxy.bytes` bbox x[−0.8535,0.8542] m = `−x_FBX·0.01` (same y,z). Mixamo `LeftArm` is at **+x in the FBX (and in Blender's import)** but at **x=−0.1867 in the Unity blob**. Vertex *order* also differs between FBX and blob (same count). Winding: a crude "normal points away from the body axis" test gives the same fraction (0.682 over 31,391 central triangles) for the FBX and for the mirrored blob; a mirror without an index flip would invert that fraction, so Unity **reversed the triangle winding** on import, which keeps `cross(e1,e2)` pointing outward in both meshes.
- Consequence: `ProxyToScanHint ≈ 0.998·I` (Umeyama with forced det=+1, `MixamoProxyAligner.cs:L224-226`) succeeded only because the body is nearly left/right symmetric; it is not the true relation (the true one is ×0.01 with identity rotation and zero translation, since `extract_proxy.py` only scales).
- Measured effect (vertex→nearest opaque-Gaussian-centre distance): all vertices mean 10.9 mm (Unity mirrored) vs 9.7 mm (unmirrored); in the 99 strongly asymmetric Gaussian regions (mirror NN > 2.5 cm) **19.7 mm vs 11.7 mm**. The scan itself has mirror-NN mean 7.4 mm.
- Blender 4.1.1 import of the FBX (axis `-Z/Y`): 1 armature (33 bones), mesh 18,612 v / 37,400 f / 28 vertex groups, `LeftArm` head x=+0.1867, `LeftToeBase − LeftFoot` points toward FBX +z; mesh carries a 0.01 object scale. Blender GLB export (`export_scene.gltf`, skins on) produced a 1,053,360 B GLB with 18,612 upload vertices.

### 12.5 Other measurements
Proxy T-pose joints (bind, mesh space): Hips (0.0003,−0.2845,−0.0214), Head (0.0003, 0.2063,−0.0260), feet y=−1.2329, toes y=−1.3171; `LeftToeBase−LeftFoot = (−0.027,−0.084,+0.119)` ⇒ character faces +z. Bindposes are rigid (|det−1| ≤ 3.6e-7). Dataset-wide: the first 12 PLYs share the sample's frame/size (y∈[−1.42,0.47], x≈±0.9), so the sample proxy *approximately* fits other figures (possible "swap character" stretch feature after re-binding).

---

## 13. PHASE 1b: Asset availability

### Verdict: **REAL ASSET OBTAINABLE** (no synthetic character required)

All needed assets are files inside the upstream repository; nothing hosted elsewhere.

**Source**: https://github.com/uhhhci/Gsplat-unity-virtual-human (branch `master`, HEAD `8ed42ea`, "v1.0 release"). GitHub API: repo `size` 118,913 KB, `releases` = `[]`, no tags (`git ls-remote --tags` empty), license field "Other/NOASSERTION" (custom files). Searched README/Dataset README/tool docs for URLs: **no Hugging Face / Zenodo / Google Drive / release-asset links exist.** `RELEASE_NOTES.md` referenced by README is absent from the repo.
Local copy: `/Users/mbp16_2022_m1max/MyTests/GSplatPeople-web/.scratch/Gsplat-unity-virtual-human` (shallow clone, 300 tracked files, 100 MiB pack).

### Needed files (all local; sizes exact)

| Purpose | Path (under `Assets/GsplatRig/` unless noted) | Size |
|---|---|---|
| Gaussian cloud (40,000) | `SampleFigures/LHM500M_woman-photoreal.ply` | 2,720,415 B |
| Rig proxy (mesh+weights+bindposes) | `MixamoRigs/LHM500M_woman-photoreal_mixamo.rigproxy.bytes` + `.rigproxy.asset` (YAML: names, parents, hint) | 1,035,540 B |
| Binding (fixture + reference) | `MixamoRigs/LHM500M_woman-photoreal_mixamo.riggedgsplat.bytes` + `.asset` | 1,280,032 B |
| Unmirrored rigged proxy (recommended authoring source) | `RigSources/LHM500M_woman-photoreal/LHM500M_woman-photoreal_proxy.fbx` | 1,458,976 B |
| Proxy OBJ (Poisson output, cm) | `PythonTools/poisson_extract/mixamo_upload/LHM500M_woman-photoreal_proxy.obj` | 4,201,976 B |
| Animations (12 clips) | `RocketboxAnimations/*.fbx` (`f_cheer_03, f_dancing_cool, f_gestic_laugh_loud, f_gestic_listen_accept_03, f_idle_neutral_02, f_walk_neutral_01, m_claphands_01, m_gestic_shrug_02, m_gestic_talk_relaxed_02, m_idle_look_around_02, m_walk_neutral, m_wave_01`) | 24.9 MB total |
| Source image (provenance) | `SampleFigures/SourceImages/LHM500M_woman-photoreal.png` | 1,643,396 B |
| Extra figures (PLY only, **no rigs**) | `Dataset/ply/*.ply` (25 × 2,720,415 B = 65 MB) + `Dataset/images/*.png` | ~94 MB dir |

Total for the MVP (one character + all clips): **≈ 35 MB** (≪ 500 MB). Upstream full Rocketbox library (471 clips, ~1.5 GB) is at https://github.com/microsoft/Microsoft-Rocketbox (`Assets/Animations/all_animations_max_motextr_{static,xy,xyz}/<name>.max.fbx`; raw URL pattern in `fetch_rocketbox_animations.py`); the 12 shipped clips suffice.

### License / terms (exact sources)

| Item | Terms | Source |
|---|---|---|
| Toolkit code (VH + vendored renderer) | MIT (Ke Li); upstream MIT notices (Yize Wu 2025, Aras Pranckevičius 2023) must travel with any copy | `LICENSE.md`, `Assets/gsplat-unity/LICENSE.md` |
| **Avatar data** (sample PLY, its rig bake + RigSources, source image, all `Dataset/`) | **Research/academic, non-commercial only** ("draft, pending legal review"). Permits use/copy/modify/redistribute for non-commercial research & teaching. Forbids: any commercial use, depicting/identifying real people, removing image provenance (C2PA), redistribution without the licence file + attribution. Requires: cite the paper, state data used under this licence, **mark modified data as modified**; honour removal requests within 30 days. | `Dataset/LICENSE.md` §2-5 |
| Rocketbox clips | MIT © Microsoft 2020; authors ask for citation (Gonzalez-Franco et al., *The Rocketbox Library and the Utility of Freely Available Rigged Avatars*, Frontiers in VR 2020) | `RocketboxAnimations/LICENSE-Microsoft-Rocketbox.md` |
| Mixamo | rig output is covered by the data licence above; Mixamo *clips* are not redistributable and are not shipped | README, Rocketbox LICENSE |
| SMPL-X / AMASS | not included, not needed | README |
| LHM (generator) | Apache-2.0 code, weights own terms; irrelevant to shipped outputs | `SourceImages/ATTRIBUTION.md` |

Practical conclusions: fine for a local, non-commercial research web demo. If assets are committed/redistributed in our repo, ship `Dataset/LICENSE.md` + attribution + the Rocketbox LICENSE, label converted files (`.splat`/GLB/binding) "modified", and keep the demo non-commercial. Only the **sample figure has a rig**; the other 25 are PLY-only (rigging them needs Mixamo manual upload or the restricted SMPL-X path).

---

## 14. Risks, quirks, and recommendations for the design/plan steps

### 14.1 Keep the algorithm, change the authoring frame
- Runtime kernel: port `DeformProxyVertices` (LBS) + `DeformGaussians` exactly (section 7). It is mirror-agnostic linear algebra; verified.
- Authoring: build the proxy GLB from the **unmirrored FBX** (RH, FBX-native, apply the 0.01 scale), bake bindings from that GLB (single source of truth for triangle order; Unity's blob has a different vertex order and mirrored winding), and set `ProxyToScan = identity` (proxy in metres already equals scan space). Re-run the sample's binding with the float64 baker; the rest-pose reconstruction must be exact (acceptance: max position error ≲1e-6 m, max covariance error ≲1e-6 on non-degenerate hosts).
- Keep the **Unity blobs as parity fixtures** (kernel tests, section 14.5).

### 14.2 Fix the sliver defect in *our* baker, replicate it only in parity tests
- Exclude triangles with tiny area from the nearest-triangle candidate set (or weight the choice by conditioning), use float64 and no 1e-5 epsilon; add a bake report of max/mean reconstruction error and sliver count.
- Runtime WGSL must guard `normalize` of a zero/near-zero cross product (NaN would poison the sort key and the quad). The Unity kernel has no guard.
- For kernel-parity tests against the Unity blob, compare the 38,868 well-conditioned Gaussians at tight tolerance (≈1e-5 m / 1e-4 covariance in float32) and treat the 1,132 sliver Gaussians separately (ill-conditioned normals; expect larger float32 divergence).

### 14.3 Animation (the biggest unknown)
Mecanim retargeting has no browser equivalent. Plan an **offline Blender-headless retargeter** (Blender 4.1.1 at `/Applications/Blender.app`, verified to import both FBX files and export GLB): sample each Rocketbox clip per frame, transfer per-bone rest-relative world rotations onto the Mixamo T-pose skeleton, convert to local, scale hip translation by the hip-height ratio, drop the root-motion/`Footsteps` channels (Unity cancels them), resample to 30 fps, export animations into the proxy GLB (28 skinned joints at minimum). Keep clips in place. Fallback if quality is poor: procedurally authored clips (still validates the pipeline). `gltf-transform` CLI is available for GLB inspection.

### 14.4 WebGPU-specific porting traps
- `array<vec3f>` stride 16 vs Unity's packed 12 (section 11). Prefer `array<f32>` ×3 or `vec4f` padding; keep bary as `vec2f`.
- Sort uses wave intrinsics; WebGPU `subgroups` shipped in Chrome 134 (https://developer.chrome.com/blog/new-in-webgpu-134) but is not universal, and subgroup sizes vary. For 40k Gaussians a non-subgroup GPU sort (e.g. bitonic or shared-memory radix) is viable; the design step should pick one and keep a path that avoids CPU readback.
- `StructuredBuffer<uint>` payload order persists across frames in Unity; harmless to replicate.
- Decide whether to reuse an existing web renderer (PlayCanvas/Spark/other) — **not investigated in this phase**. The draw math in section 8.3 is PlayCanvas-derived, so an adapter that feeds (position, rotation, scale, colour) buffers is conceptually compatible; buffer stride/format must be checked against the chosen renderer.

### 14.5 Numerical validation plan (what is actually available)
- **Available now**: the shipped PLY + both blobs + the float64 reference formulas (section 7) reproduce the scan exactly at rest for well-conditioned hosts (section 12.2). This is non-circular (PLY is independent input) and validates all convention choices. A WGSL kernel can be checked (a) at rest pose against the PLY, (b) under random/analytic bone transforms against a float64 CPU port of the HLSL, with G0-G2 in section 5 as spot vectors.
- **Not available**: a run of real Unity. Project requires `6000.4.2f1` (`ProjectSettings/ProjectVersion.txt`); installed editors include 6000.3.x and 6000.4.0a5 only (unverified compatibility, licence state unknown); the renderer's sort needs D3D12/Vulkan (macOS Metal unsupported), though `GsplatSkinning.compute` itself uses no wave ops. A batch-mode dump of `_DeformedPositions/_DeformedRotations` for fixed bone matrices would be an optional stretch golden; without it, "validated against the Unity math" means validated against the HLSL-equivalent float64 port plus the rest-pose identity.

### 14.6 Known limitations inherited from the original (document, don't fix in MVP)
No SH rotation (shipped data has no SH anyway); Gaussian scale does not follow triangle stretch; shoulders/armpits weakest; hair rigid to head; no facial animation on the Mixamo path; ≤4 bones/vertex; root motion cancelled.

### 14.7 Suggested runtime asset contents (input to the format design)
Per Gaussian: scale (3×f32 or f16), colour rgb + opacity, binding (u32 tri, 2×f32 bary, f32 offset, 4×f32 rest quat; quantisation is possible but not needed at 40k). Plus proxy: triangle indices, (optional) rest vertices, joints/weights (4×u8/f32), inverse-bind matrices; skeleton hierarchy + animation clips (GLB). The PLY's rest position/rotation are needed only at bake time.

---

## 15. Not verified / not covered

- No Unity run; Unity-side behaviours (`Vector3.normalized` epsilon, `Quaternion.LookRotation` zero-input result, importer x-flip) are inferred from documented API behaviour plus proof on the data (exact match of all 1,132 zero-offset Gaussians to `|cross|≤1e-5` hosts; 38,868 exact round-trips; blob = x-negated FBX). Importer internals were not read.
- `GsplatRigSample.unity` and generated `.controller` files were not parsed. `SmplxFitToRigProxy.cs`, `SmplxPoseDriver.cs`, `PythonTools/smplx_fit/*`, `RigProxyDebugViewer.cs`, `FlyCameraController.cs`, `lhm_prompts` were skimmed only (SMPL-X path is out of scope).
- Web renderer survey (PlayCanvas / Spark / others) is not part of Phase 1 and was not done.
- Draw-shader derivation in 8.3 (focal doubling, kernel scale) is from reading code, not from running it.
- Sorting kernels were read structurally (dispatch sequence, key transform, direction); the wave-level internals of `Upsweep/Scan/Downsweep` were not re-derived.
