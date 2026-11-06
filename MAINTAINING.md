# Maintaining XDL-browser-packages

How to release, test, and update the pinned libraries and the toolchain. [README.md](README.md) describes what the library and the releases contain.

## Workflows

| Workflow | Runs | Does |
|---|---|---|
| [Build](.github/workflows/build.yml) | On every pull request to `main`, and inside Release | Checks out the pinned libraries, applies the XDL patches, installs the .NET 11 preview SDK and its `wasm-tools` workload, builds the release assets with `tools/build-release.sh`, links them into the .NET browser app in `test/dotnet`, and uploads them as the `release-assets` artifact |
| [Release](.github/workflows/release.yml) | By hand | Computes the tag, stops if it exists, runs Build, and publishes the artifact as a GitHub release. Only its last job has a token that can write |

A pull request's `release-assets` artifact is a preview: the archives are the ones a release of the merged commit would publish, so they can be downloaded and tried before merging. Its tag is the placeholder `pr-<number>`, and its `release-notes.md` links to that tag, so its release notes are not final.

## Release

1. Merge the pull request to `main`.
2. In GitHub: Actions → Release → Run workflow, on `main`.
3. The tag is the released commit's UTC commit time, `vYYYYMMDD.HHMMSS`. A commit can be released once; a second run stops at the existing tag. To release again, commit something.
4. The release notes list the Emscripten version, the flags, and the archives in link order with each library's version and commit. `tools/native-references.sh` prints a release's archives as MSBuild items. The archives download without credentials only when the repository is public.

A failed run publishes nothing. Fix the cause on a branch and run Release again after merging.

## Run the checks locally

Needs the .NET 11 `wasm-tools` workload and Node.js.

```
git submodule update --init external/XDL external/SDL_image external/SDL_mixer external/SDL_ttf
git -C external/XDL submodule update --init external/SDL
git -C external/SDL_ttf submodule update --init external/freetype external/harfbuzz external/plutosvg external/plutovg
external/XDL/tools/apply.sh

# Release assets, into build/release-assets
source external/XDL/tools/emsdk-env.sh
tools/build-release.sh vTEST owner/repo
```

Then, in a new shell without `emsdk-env.sh`, the .NET link test:

```
dotnet publish test/dotnet -c Release -o build/dotnet-link -p:XdlArchiveDirectory="$PWD/build/release-assets" -p:WasmCachePath="$PWD/build/dotnet-em-cache"
node test/dotnet/run.mjs build/dotnet-link/wwwroot
```

Expected last line: `XDL link test PASSED`. Keep `WasmCachePath` apart from the cache that `emsdk-env.sh` sets: a .NET link into that cache emptied it once, emdawnwebgpu port included, and the next build failed.

## Update SDL_image, SDL_mixer or SDL_ttf

1. Before the update, record the current notices and license texts:

   ```
   tools/build-release.sh vTEST owner/repo
   cp build/release-assets/THIRD-PARTY-NOTICES.txt build/notices-before.txt
   xargs grep -l -i -E "licen[cs]e|copyright|permission to use|permission is hereby|freely granted|public domain|unicode\.org" \
     < build/release/compiled-sources.txt | sed "s|^$(pwd -P)/||" | sort > build/license-files-before.txt
   ```

2. Move the submodule to the new release tag:

   ```
   git -C external/SDL_image fetch --tags
   git -C external/SDL_image checkout release-X.Y.Z
   ```

   For SDL_ttf, also update its vendored libraries: `git -C external/SDL_ttf submodule update --init external/freetype external/harfbuzz external/plutosvg external/plutovg`.
3. Run `tools/build-release.sh vTEST owner/repo` again.
   - If a CMake option changed name, the configure fails or the codec summary (`SDL3_image backends`, `SDL3_mixer backends`) changes. Compare it with the codec lists in README, Releases.
   - If a copied notice moved or changed, the script fails with "the notice of … is missing or empty". Fix its start and end patterns in `tools/build-release.sh`.
4. Look for new third-party licenses. Run the `xargs grep` of step 1 into `build/license-files-after.txt`, then compare:

   ```
   diff build/notices-before.txt build/release-assets/THIRD-PARTY-NOTICES.txt
   diff build/license-files-before.txt build/license-files-after.txt
   ```

   The copyright lines in `THIRD-PARTY-NOTICES.txt` follow by themselves, and every compiled file's copyright holders are listed there. A new holder or a new file with a license text points to code to read. If its license asks to keep its notice or permission text in copies or documentation, and the library's own license does not cover it, add a `notice` or `notice_file` entry to `tools/build-release.sh`. Public-domain code and zlib-licensed code need no entry; their licenses ask for no notice in a binary. Code derived from Unicode data (tables generated from the Unicode Character Database or emoji data) is covered by `LICENSES/Unicode-3.0.txt`; extend that entry's title if a new library has such tables.
5. Run the .NET link test.
6. Commit the submodule and open a pull request. Build runs on it.

## Update XDL

Follow the steps of the section above. Replace only its step 2 with: move the submodule (`git -C external/XDL fetch origin`, `git -C external/XDL checkout <tag or commit>`, `git -C external/XDL submodule update --init external/SDL`, `git add external/XDL`), then run `external/XDL/tools/apply.sh`.

If XDL moved to a new SDL version, a copied notice can move, and step 4's license check matters most.

## Change the codecs

The codecs of SDL_image and SDL_mixer are the `set(SDLIMAGE_…)` and `set(SDLMIXER_…)` lines in `CMakeLists.txt`. The build uses only codecs that need no extra library, so each release has four archives. A codec that needs a library (libwebp, libopus and others) needs the library's submodule; its objects in `SDL3_image.a` or `SDL3_mixer.a`, which SDL_image and SDL_mixer do not do by themselves (unlike SDL_ttf, whose archive holds its vendored libraries), so `tools/build-release.sh` has to merge them; its row in the release notes; and its license and copyright lines in `THIRD-PARTY-NOTICES.txt`. Update the codec lists in README, Releases.

## .NET and Emscripten

- Build installs the newest .NET 11 preview SDK, so its Emscripten can change between two releases without a commit. The release notes of each release record the Emscripten version and the flags.
- The archives link only into a .NET runtime built with the same Emscripten. A new .NET major version needs a new `dotnet-version` in `build.yml` and a new `TargetFramework` in `test/dotnet`.
- Emscripten pins the Dawn version of the emdawnwebgpu port, so a new Emscripten can change the WebGPU header the backend compiles against. A compile error in XDL's `SDL_gpu_webgpu.c` after a .NET update is the first sign.
- The release flags (`-fwasm-exceptions -sWASM_LEGACY_EXCEPTIONS=0`) match the .NET runtime's. If .NET changes its exception handling, change `flags` in `tools/build-release.sh`.

## Where the pinned versions are written

| What | Where |
|---|---|
| XDL, SDL_image, SDL_mixer, SDL_ttf | Submodules under `external/` |
| SDL | XDL's `external/SDL` submodule |
| FreeType, HarfBuzz, plutosvg, plutovg | SDL_ttf's submodules; the release notes of each release |
| Emscripten, Dawn | The .NET workload |
