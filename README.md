# U8g2 – Middleware (Azure DevOps staging)

Internal Azure DevOps mirror of **U8g2** (upstream: [olikraus/u8g2](https://github.com/olikraus/u8g2)), the monochrome graphics library for embedded displays, imported here for testing before being exported as a public middleware module to GitHub.

This is the **import** side only — see `U8g2_AzureImport.sh` in `Middlewares_Importer`. The export step (Azure DevOps → GitHub, with its own testing/approval logic) is separate and not yet built.

This file itself is regenerated (copied verbatim) by every import run — don't hand-edit it here, edit `README_U8g2Default.md` next to `U8g2_AzureImport.sh` instead.

---

## Branch & repository layout

Everything lives on a single anchor branch (`master`). One real commit per U8g2 version, tree replaced wholesale each time, commit message naming its version (e.g. `U8g2 2.36.5`). No git tags are created here — this is the Azure DevOps staging side only; tagging happens later, during the separate export step that promotes a tested/approved version to the public GitHub middleware repo.

```text
<repo root>
├── CMakeLists.txt   auto-generated every version (see below) — do not hand-edit
├── README.md        this file
└── U8g2/            the release as upstream tags it, submodules vendored in place,
                     no .git / .gitmodules / .gitignore anywhere
    ├── CMakeLists.txt     upstream's own — defines the `u8g2` library target
    ├── csrc/              the C library: u8g2, u8x8, u8log, MUI — this is what gets built
    ├── cppsrc/            Arduino C++ wrapper (U8g2lib / U8x8lib) — headers only are exposed
    ├── sys/               upstream's per-platform examples and HAL samples
    ├── tools/             upstream's font/code generators
    └── ...
```

## Source

`U8g2_AzureImport.sh` discovers versions via the GitHub **Tags** API (u8g2 publishes no GitHub Releases) and only takes plain `x.y.z` tags. Each version is fetched with a shallow `git clone --branch <tag>`, the checked-out tag is verified to be exactly that version, and the resulting tree is copied into `U8g2/` as-is.

## Submodules

Upstream references a submodule — `sys/arm-linux/c-periphery` (from [vsergeev/c-periphery](https://github.com/vsergeev/c-periphery), used by the Linux example ports). `U8g2_AzureImport.sh` resolves every submodule recursively at the exact commit the tag pins (`git submodule update --init --recursive`), then strips **every** `.git` entry, `.gitmodules` and `.gitignore` file before committing. What lands in `U8g2/` is fully flattened, ordinary source — no submodules, no gitlinks, no dependency on any other repository. A version whose submodules can't be fetched is skipped with a warning rather than imported incomplete.

## CMake support requirement — versions without a usable target are not imported at all

The generated root `CMakeLists.txt` only wraps upstream's own `U8g2/CMakeLists.txt`, so a version is imported **only** if that file exists and defines the plain `add_library(u8g2 ...)` target. Anything else is **skipped entirely — no commit, nothing published for it in this repo.** In practice (checked against every upstream `x.y.z` tag) this excludes only `2.29.11`, whose `CMakeLists.txt` is an ESP-IDF-only `register_component()` file with no regular CMake target.

---

## Building against it — the generated `CMakeLists.txt`

The root `CMakeLists.txt` in this repo does **not** invent any library — it just `add_subdirectory()`s `U8g2/`, whose upstream `CMakeLists.txt` defines the `u8g2` target:

- sources: every `csrc/*.c` (u8g2, u8x8, u8log and MUI — including all display drivers and fonts; unused ones are dropped by the linker with `-ffunction-sections -fdata-sections` + `--gc-sections`),
- `PUBLIC` include directories: `csrc/` and (in newer releases) `cppsrc/`.

Vendor this repo (e.g. as a submodule at `Middlewares/U8g2`), then from **your own** top-level `CMakeLists.txt`:

```cmake
add_subdirectory(Middlewares/U8g2)
target_link_libraries(${PROJECT_NAME}
    PRIVATE
        u8g2
)
```

```c
#include "u8g2.h"
```

U8g2 itself has no hardware access — you provide the two `u8x8` callbacks for your MCU (`u8x8_msg_cb` for the byte transport — SPI / I²C / parallel — and for GPIO & delay) and pass them to the matching `u8g2_Setup_<display>()` call. See upstream's [porting guide](https://github.com/olikraus/u8g2/wiki/Porting-to-new-MCU-platform).

### Notes on the upstream target

- **C++ wrapper is not compiled.** `cppsrc/U8g2lib.cpp` / `U8x8lib.cpp` depend on the Arduino framework and are not part of upstream's `u8g2` target — only the `csrc/` C API is built.
- **ESP-IDF** is handled by upstream's file itself: under ESP-IDF it calls `idf_component_register()` and returns before `add_library()`; use `U8g2/` as an IDF component there instead of going through the root wrapper.
- Upstream's file also declares `install()` rules (library, headers, `u8g2-config.cmake`) — harmless unless you run `cmake --install` on your own project.

---

## Notes

- A brand-new repo (or an existing repo that's just missing the `master` branch) gets a real `Initial commit` containing just this `README.md`, pushed immediately when the branch is created — so it's never left with zero commits even if the first version afterwards fails or every available version is skipped. The first version actually imported then wholesale-replaces that with the full tree, same as every later version.
- No git tags are created by this importer — tagging happens later, during the separate export step.
- Re-running the import is idempotent — a version already present as a commit on `master` (matched by its commit message, `U8g2 <version>`) is skipped.
- Versions come straight from upstream git tags (`x.y.z`, no `v` prefix) — no normalization needed, unlike FreeRTOS.

---

## Authors

- **Mr.Nobody** — [embedbits.com](https://embedbits.com)
