# XDL-browser-packages

Prebuilt Emscripten archives of [XDL](https://github.com/stanoddly/XDL), SDL_image, SDL_mixer and SDL_ttf, for the .NET browser runtime. XDL is an altered version of SDL, and its archive includes the WebGPU backend for the SDL GPU API.

This repo holds no library of its own. It pins the libraries as submodules, builds them, tests the result and publishes releases. This repo replaces the packaging in [XDL_wgpu](https://github.com/stanoddly/XDL_wgpu), whose existing releases stay available.

## Using from .NET (browser-wasm)

Reference the archives with `NativeFileReference`. Keep the file names: the .NET WASM toolchain derives the P/Invoke module name from the file name, so `XDL3.a` registers the `"XDL3"` module your `DllImport("XDL3")` calls bind to.

```xml
<ItemGroup>
  <NativeFileReference Include="native/XDL3.a" />
  <NativeFileReference Include="native/SDL3_image.a" />
  <NativeFileReference Include="native/SDL3_mixer.a" />
  <NativeFileReference Include="native/SDL3_ttf.a" />
</ItemGroup>
<PropertyGroup>
  <EmccExtraLDFlags>--use-port=emdawnwebgpu</EmccExtraLDFlags>
</PropertyGroup>
```

`tools/native-references.sh [--file <dir> | --url] [tag]` prints the archives of a release, the latest without a tag, as MSBuild items in link order. `--file <dir>` (the default, with `<dir>` = `native`) downloads them into `<dir>`, checks their SHA-256 and prints these `NativeFileReference` items with absolute paths. `--url` prints `NativeUrlReference` items with each archive's URL and SHA-256 instead.

The .NET runtime does not use asyncify, so an app must create the GPU device without blocking. [XDL's WebGPU README](https://github.com/stanoddly/XDL/blob/main/patches/0007-webgpu-backend/README.md) describes how, along with the shader format and property names.

## Releases

The [Build workflow](.github/workflows/build.yml) runs on every pull request. It builds the archives with the Emscripten of the .NET 11 `wasm-tools` workload and the runtime's flags (`-fwasm-exceptions -sWASM_LEGACY_EXCEPTIONS=0`), and links them into a .NET browser app (`test/dotnet`). The [Release workflow](.github/workflows/release.yml) is started by hand. It runs Build and publishes its artifact.

- The version is the released commit's UTC commit time, `vYYYYMMDD.HHMMSS`. A run stops when the tag exists, so a second run for the same commit publishes nothing.
- Assets: `XDL3.a`, `SDL3_image.a`, `SDL3_mixer.a`, `SDL3_ttf.a` and `THIRD-PARTY-NOTICES.txt`. `SDL3_ttf.a` also contains the libraries SDL_ttf vendors: FreeType, HarfBuzz, plutosvg and plutovg. GitHub shows the SHA-256 of each asset.
- SDL_image loads ANI, BMP, GIF, JPEG (stb_image), LBM, PCX, PNG (SDL's codec), PNM, QOI, SVG, TGA, XCF, XPM and XV. AVIF, JXL, TIFF and WebP are off. SDL_mixer plays WAVE, AIFF, VOC, AU, FLAC (dr_flac), MP3 (dr_mp3), Ogg Vorbis (stb_vorbis) and MIDI (TiMidity). Opus, MOD, GME and WavPack are off.
- The release notes list the Emscripten version, the flags, and the archives with the version and commit of each library's source.

[MAINTAINING.md](MAINTAINING.md) covers building locally, releasing and updating the pinned libraries.

## License

zlib (see `LICENSE`). The libraries keep their own licenses, listed in each release's `THIRD-PARTY-NOTICES.txt`. `LICENSES/Unicode-3.0.txt` is the Unicode license, for HarfBuzz's tables generated from Unicode data.

## AI disclosure

Except for the code derived from SDL and the original WebGPU backend, the work in this repository was produced with the help of LLM-based coding tools, under human direction and review.
