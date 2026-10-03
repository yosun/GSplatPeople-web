# Environment

- Platform: macOS Apple Silicon (Apple M3 Max per `sysctl machdep.cpu.brand_string`; an earlier note said M1 Max), zsh
- node v26.5.0, npm 11.17.0, git 2.53.0, Python 3.14.7
- npx at /opt/homebrew/bin/npx. Playwright/puppeteer can be installed locally later via npm (nothing installed now, nothing global).

# Source repo
- .scratch/Gsplat-unity-virtual-human (shallow clone of https://github.com/uhhhci/Gsplat-unity-virtual-human)
- HEAD: 8ed42ead77f2a3b0aba0d7bc4f927f74453d2718

# gsplat-unity dependency
- VENDORED in the source repo at Assets/gsplat-unity (package de.uhh.hci.gsplat-unity 1.0.0, author Ke Li, MIT).
  No Packages/manifest.json entry and no .gitmodules. This vendored copy is the authoritative dependency.
  Shaders: Gsplat.compute, GsplatGather.compute, Gsplat.shader, GsplatComposite.shader, DeviceRadixSort.hlsl, SortCommon.hlsl, etc.
- The vendored package does not name an upstream URL. A likely ancestor, https://github.com/wuyize25/gsplat-unity
  (v1.4.0, commit a2bf458d6b16395e6570e9345f9f4408f92684b8), was cloned to
  .scratch/Gsplat-unity-virtual-human-deps/gsplat-unity for REFERENCE ONLY. Its layout differs (Runtime/Shaders/CalcDepth.compute etc.),
  so the ancestry is unverified. Trace behavior from the vendored copy, not this clone.
