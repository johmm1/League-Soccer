# FloodlightFC Asset Upgrade Foundation

## Goal

Upgrade the existing C++/SDL2/OpenGL football game with a curated, legally usable asset collection while preserving a playable low-spec PC and Android path. This is an integration plan and asset register; it does not claim that third-party files have already been downloaded or imported.

## Current pipeline (repository inspection)

- Rendering: OpenGL.
- Existing geometry path: ASE models and the game's `.object` bundles.
- Animation: the game's own keyframe format and node-name-based `Animation` system.
- Runtime assets live under `data/media/` and are copied beside the executable by CMake / `scripts/setup_assets.sh`.

A GLB/FBX model is not plug-and-play. Before importing a rigged player, we must verify skeleton hierarchy, bone names, scale, forward axis, animation compatibility, material/texture support, and license. Keep source assets separate from converted game-ready outputs.

## Asset register

| Category | Candidate / source | Status | Integration gate |
|---|---|---|---|
| Stadium | Innerscene football stadium model listing: https://www.innerscene.com/tools/library/3d-parts/football-stadium-f7b93967 | Candidate only; verify current download terms and actual file | Test triangle count, material count, UVs, scale, and conversion to current geometry pipeline |
| Player / goalkeeper | Quaternius character assets referenced by openFIFA: https://github.com/tshrjn/openFIFA/blob/main/docs/ASSETS.md | Candidate only; verify exact asset, license, rig, and included animations | Import one player first; goalkeeper can share base rig if proportions/animations work |
| Alternate soccer character | https://github.com/TechLabID/techlabid | Candidate only; review repository license and asset-specific terms before use | Check redistribution restrictions, file formats, skeleton and texture sizes |

No candidate is approved for redistribution or bundled into a release until its exact license and source are recorded. Prefer CC0 / permissive assets with explicit attribution terms. Do not copy assets from proprietary games or repositories without a clear license.

## Target asset layout

```
data/media/
  source_assets/       # optional, unmodified sources; keep out of releases when terms require
  objects/             # converted stadium, players, boots, goal frames
  textures/            # optimized textures and material maps
  animations/          # game-native clips / mapping notes
  sounds/              # licensed crowd, whistle, boot and ball sounds
  asset_licenses/      # per-asset source, author, license, date checked, attribution
```

## Import and optimization sequence

1. Inspect the current ASE loader, `.object` loader, humanoid node map, and animation collection.
2. Establish one repeatable conversion/import path. Prefer a small standalone converter/test first; do not replace the runtime loader until a sample asset passes.
3. Import one stadium and one rigged player. Confirm materials, normals, UVs, scale, skeleton mapping, and match camera framing.
4. Build lightweight gameplay versions: reduce geometry, use sensible texture sizes, and add LODs where supported. Keep detailed assets for close-ups only.
5. Add goalkeeper-specific animation coverage, then kits/boots, pitch details, goals, crowd/audio and presentation assets.
6. Run build and match smoke tests after each batch. Keep existing assets as fallback until replacements pass.

## Acceptance checklist

- [ ] Exact source URL, author, license, and required attribution recorded.
- [ ] File opens and converts without losing normals, UVs, materials, or skeleton.
- [ ] Player rig maps to the game's node-based animation system.
- [ ] Stadium does not block camera, goals, ball, or players.
- [ ] No severe frame-time or memory regression on the target PC.
- [ ] Mobile-friendly asset variant exists before Android packaging.
- [ ] Build and kickoff smoke test pass; rollback path remains available.
