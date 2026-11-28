# pixtreme-transport

`pixtreme-transport` provides `pixtreme.transport`, the core of pixtreme's companion for connecting external video streams with `px.core.Frame`. Protocol and codec modules provide stream I/O, decoding and encoding through separately installed distributions.

One core wheel includes the C++ shared library, public C headers and header-only C++ wrappers, CMake package configuration, nanobind Python bindings and the `pixtreme.transport.testing` package. The core supplies the common contracts and SDK that modules use.

## Install

The following commands install the Linux wheels currently published on PyPI.

```bash
pip install pixtreme-transport
```

For a uv project:

```bash
uv add pixtreme-transport
```

The core depends on `pixtreme>=2.34.0`, which the package installer resolves automatically. Module developers can install the conformance suite's pytest dependency with the `testing` extra:

```bash
pip install 'pixtreme-transport[testing]'
# Or, in a uv project:
uv add 'pixtreme-transport[testing]'
```

The testing package is included in the core wheel and shares its version. Ordinary `import pixtreme.transport` defers native loading and does not import the testing package or pytest.

## Supported environments

The wheels published on PyPI support Linux x86_64 with glibc 2.28 or later (`manylinux_2_28_x86_64`) and, from version 0.35.1, native Windows x86_64 (`win_amd64`), with separate wheels for CPython 3.12 and 3.13 on each platform. Runtime use requires an NVIDIA GPU and an NVIDIA driver that supports the CUDA 13 runtime.

When using a Windows wheel, install the [Microsoft Visual C++ v14 Redistributable (x64)](https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist/) for toolset 14.44 or later. It supplies `MSVCP140.dll`, `MSVCP140_ATOMIC_WAIT.dll`, `VCRUNTIME140.dll` and `VCRUNTIME140_1.dll`. The package registers its own DLL directory during import, after pixtreme initializes its directory, so runtime use needs no compiler, CUDA toolkit or DLL directories added to `PATH`.

PyPI releases contain wheels only; no source distribution is uploaded. Installation from PyPI requires a published wheel for the selected operating system, CPU architecture and Python version; otherwise it fails with no matching distribution. The package does not provide a CPU-only runtime mode.

## Protocol and codec modules

Install only the module distributions needed by your application. Modules use distribution names of the form `pixtreme-transport-<module>` and import names of the form `pixtreme.transport.<module>`. Each module documents its supported protocols, codecs and additional dependencies. Installing the core alone does not install protocol or codec modules.

## Create sessions by module name

C and C++ programs can create an entrance or exit session from a module's name. `<pixtreme/transport/c/module.h>` declares `pixtreme_transport_module_create_entrance` and `pixtreme_transport_module_create_exit`. Each takes the module name (`[a-z][a-z0-9_]*`, the same `<module>` as the import name), a sized request record (the device and the caller's frame buffer, plus the core exit settings for an exit) and an array of name/value string settings. The core selects a registered builtin entry when its descriptor declares that name. For other names, the shared core opens only `<module>/lib/libpixtreme_transport_<module>_entry.so` on Linux or `<module>/bin/pixtreme_transport_<module>_entry.dll` on Windows under the installed `pixtreme/transport` directory, located from the core's own shared library, and calls its `pixtreme_transport_module_entry` function. Both routes check the descriptor version and the core ABI. A core statically embedded from a source build returns `module-not-found` for names absent from its builtin list, including when no list was registered. It does not search or load shared module libraries. The core does not scan directories or read environment variables. On success the call returns an unopened session; start it with the existing open operation.

Failures carry stable reasons: `invalid-module-name`, `module-not-found` (its guidance names the distribution `pixtreme-transport-<module>` to install), `module-load-failed`, `module-entry-missing`, `module-abi-mismatch`, `module-direction-unavailable`, `invalid-module-option` and `module-create-failed`. Each call creates an independent instance that can be used on another thread; a loaded module library stays loaded until the process ends. Module discovery is implemented for Linux and native Windows x86_64.

Setting names use `.` for nesting, for example `connection.channel`. Values are text: decimal integers with an optional leading `-` (no `+`, spaces, `0x` or fractions), `true` or `false`, and rationals `<numerator>/<denominator>` with a positive denominator. Each module documents its keys. Errors name the key and never include its value. Module authors and C++ callers can use the header-only `<pixtreme/transport/module.h>`, which provides `pixtreme::transport::module::Options` for parsing and validating settings, `create_entrance` and `create_exit` returning an owning `Handle`, and `creation_boundary` for converting setting errors into the C error contract.

Decoding inputs use `decode.library` (`nvdec` or `ffmpeg`); encoding outputs use `encode.library` (`nvenc` or `ffmpeg`). Module factories add the relevant core key constant to `Options`' `known_keys` and call the corresponding helper, rather than defining their own spellings or defaults. For example, an input factory can read its borrowed setting array this way, inside its `creation_boundary`:

```cpp
namespace pm = pixtreme::transport::module;
const pm::Options options(data, count, stride, {"connection.channel", pm::decode_library_key});
const pm::DecodeLibrary decoder = pm::decode_library(options);
```

An output factory uses `pm::encode_library_key` and `pm::encode_library(options)` to obtain an `EncodeLibrary`. Omission selects `DecodeLibrary::nvdec` or `EncodeLibrary::nvenc`; an empty string, a spelling from the opposite direction or any unrecognized value produces `OptionError` with the key path and allowed names, without the supplied value. `creation_boundary` maps it to `invalid-module-option`. `pm::library_name` converts either enum to the library's distribution suffix. Because omission always selects the same library and a selection is never replaced, the selected library follows from the settings the caller passed; the core does not add a session field for it. These helpers only select a library; they do not load one, inspect capabilities or fall back to another. Modules retain their other schema, range and combination checks, and call the selected library on their own receive or send thread. An unsupported codec, format or setting must fail with a reason before the first picture.

Module authors declare their entry with `PIXTREME_TRANSPORT_MODULE_EXPORT`
from `<pixtreme/transport/c/module.h>` and `extern "C"` in C++. This exports the
shared entry on Windows independently of the core's import declarations. When
compiling an entry into an executable, define
`PIXTREME_TRANSPORT_MODULE_STATIC_DEFINE` and use its suffixed entry name.
Windows source builds install the core DLL `pixtreme_transport-17.dll` in `bin/`
and its import library `pixtreme_transport.lib` in `lib/` under `pixtreme/transport`.
Applications supply the DLL search directories as described in their own
runtime configuration; module discovery does not change global search paths.

Applications that link modules into their executable pass their entry functions
to `pixtreme_transport_module_register_builtin_entries(entries, count, &error)`
once, before any name-based creation call. Each linked module supplies
`pixtreme_transport_module_entry_<module>` with the same descriptor and factories
as its shared entry. Call the C++ wrapper during initialization, then use the
usual module name and settings when creating sessions:

```cpp
#include <pixtreme/transport/abi.h>
#include <pixtreme/transport/module.h>
#include <array>

extern "C" const pixtreme_transport_module_descriptor*
pixtreme_transport_module_entry_my_module(); // Its descriptor names "my_module".

void initialize_modules() { // Call once before starting session-creation threads.
    pixtreme::transport::require_abi();
    const std::array<pixtreme_transport_module_entry_fn, 1> entries{
        pixtreme_transport_module_entry_my_module};
    pixtreme::transport::module::register_builtin_entries(entries);
}

pixtreme::transport::Handle make_input(
    const pixtreme_transport_module_entrance_request& request,
    std::span<const pixtreme_transport_module_option> settings) {
    return pixtreme::transport::module::create_entrance("my_module", request, settings);
}
```

The core accepts or rejects the complete list. Rejection leaves the previous
state intact; an accepted list stays fixed until process exit. The caller may
release its array after registration, but the entry functions, descriptors and
module code must remain available. A second registration or one after any
creation attempt fails. Concurrent registration and creation have one of the
two serial outcomes. An empty list is valid. Errors from a selected builtin
(ABI, direction or factory) never cause a fallback to the shared module with the
same name. Programs using the shared core retain shared module discovery for
names absent from the list, including when they do not register a list.

Registration adds these stable reasons: `module-entries-invalid`,
`module-entry-null`, `module-descriptor-invalid`, `module-name-duplicate`,
`module-entries-already-registered` and `module-registration-closed`. Invalid
descriptor names and versions use `invalid-module-name` and
`module-abi-mismatch`. The C++ wrapper throws the existing SDK exception types.

## Build a C or C++ module

The [public core Releases](https://github.com/sync-dev-org/pixtreme-transport/releases)
provide a prebuilt SDK and `release.json`. The SDK contains the same shared
library, headers and CMake configuration as the release wheel under
`pixtreme/transport/{include,lib,cmake}` on Linux and
`pixtreme/transport/{include,bin,lib,cmake}` on Windows. Linux SDKs support
x86_64 with glibc 2.28 or later. Use the Windows procedure below only for a
core Release with a `win_amd64` SDK asset and a shared recipe Release containing
that version's Windows entry. Neither core nor pixtreme is compiled when obtaining these
SDKs, and no Python wheel or C++ source checkout is needed.

The core and pixtreme Conan recipes are maintained together in the private
[sync-dev-org/conan-recipes](https://github.com/sync-dev-org/conan-recipes)
repository. Recipe acquisition needs Conan 2.34.0, Python 3, `sha256sum`, `tar`,
and a host `gh` login that can read that repository. C++ consumers also need a
C++20 compiler, CMake 3.31 or later and Ninja. CMake does not search for Python,
nanobind or a CUDA compiler when building the consumer.

Choose a core release and read its checked provenance record:

```bash
core_version=X.Y.Z
base="https://github.com/sync-dev-org/pixtreme-transport/releases/download/v${core_version}"
curl --fail --location --remote-name "${base}/release.json"
curl --fail --location --remote-name "${base}/SHA256SUMS"
sha256sum --check --ignore-missing SHA256SUMS
pixtreme_version=$(python3 -c 'import json; print(json.load(open("release.json"))["pixtreme_sdk_version"])')
export CONAN_HOME="$PWD/deps/conan-home"
mkdir -p "$CONAN_HOME"
```

`release.json` records the core version, pixtreme SDK version used to build it,
wheel runtime lower bound, wheel hash, and SDK archive's public URL and SHA256.
The core recipe's default `from_source=False` downloads and verifies the public
SDK without compiling it. Both core and pixtreme use `shared=True`; core requires
pixtreme at the recorded build SDK version or later. The default SDK package
identity depends only on OS and architecture. The consumer's dependency graph
contains the two shared SDKs and excludes CUDA build components and Catch2.

Create `conanfile.py` in your consumer directory, replacing `X.Y.Z` with your
chosen core minimum:

```python
from conan import ConanFile


class Consumer(ConanFile):
    settings = "os", "arch", "compiler", "build_type"
    requires = "pixtreme-transport/[>=X.Y.Z]"
    default_options = {"pixtreme-transport/*:shared": True}
    generators = "CMakeDeps", "CMakeToolchain"
```

Use the [shared recipe acquisition instructions](https://github.com/sync-dev-org/conan-recipes#using-a-recipe)
with a published shared Release tag that contains your core version. In that
example, export only the two required recipes, replacing its export commands
with these commands inside the same verified extraction directory:

```bash
conan export recipes/pixtreme --version="$pixtreme_version"
conan export recipes/pixtreme-transport --version="$core_version"
```

The shared acquisition procedure passes the host's `gh` token directly into a
temporary Conan `source_credentials.json`, downloads the archive and
`SHA256SUMS` through GitHub API asset URLs, deletes the credential home, verifies
the archive, and exports into your selected `CONAN_HOME`. Keep shell tracing and
debug logs disabled. Commit the consumer lock generated by that procedure.
Recipes stay in this shared repository; core Releases contain no recipe archive.

Create your `CMakeLists.txt`, then install with the profiles appropriate for your
compiler and build your consumer. The shared acquisition procedure detects a default profile for the
host; you can supply your own host and build profiles to `conan install`.

```cmake
cmake_minimum_required(VERSION 3.31)
project(example LANGUAGES CXX)
find_package(pixtreme_transport CONFIG REQUIRED)
add_executable(example main.cpp)
target_link_libraries(example PRIVATE pixtreme::transport)
```

```bash
conan lock create . --no-remote --lockfile="" --lockfile-out conan.lock
conan install . --lockfile conan.lock --no-remote --build=missing --output-folder build
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
cmake --build build
```

Retain your `conan.lock` and pass `--lockfile conan.lock` on later installs. Its
recipe revisions identify the selected SDKs. Keep dependency declarations as
lower bounds without an upper bound or exact version pin. CMake uses the SDK's
original `pixtreme_transport` package and `pixtreme::transport` target, including
the transitive `pixtreme::pixtreme` dependency. The CMake package version is the
core distribution's `X.Y.Z`. Call `pixtreme::transport::require_abi()` before other
core operations; it checks both core's native ABI and core's build header against
the loaded pixtreme library. ABI compatibility, rather than an exact distribution
version pin, determines whether runtime libraries can be used together.

Arrange runtime lookup for the chosen installation, for example with Conan's
runtime environment during standalone development, and load one core library
and one pixtreme library in each process. Module wheels retain relative
`$ORIGIN` paths to their installed dependencies and do not copy those libraries.

### Windows C and C++ consumers

Use native Windows x86_64 with MSVC 194 (Visual Studio 2022 toolset 14.44),
the Windows SDK, Python 3, Conan 2.34.0 and `gh`. Runtime machines need the
Microsoft Visual C++ v14 Redistributable described above. The consumer uses
the same `conanfile.py`, CMake package and `pixtreme::transport` target as Linux.
No Python wheel, CUDA toolkit or CUDA compiler is needed for SDK acquisition
or for compiling a consumer.

Select a core version whose Release contains
`pixtreme_transport-X.Y.Z-sdk-win_amd64.tar.gz` and
`release-win_amd64.json`. Download the record and `SHA256SUMS` with `curl.exe`
and verify the record before using its pixtreme version:

```bat
set "core_version=X.Y.Z"
set "base=https://github.com/sync-dev-org/pixtreme-transport/releases/download/v%core_version%"
curl.exe --fail --location --output release-win_amd64.json "%base%/release-win_amd64.json"
if errorlevel 1 exit /b 1
curl.exe --fail --location --output SHA256SUMS "%base%/SHA256SUMS"
if errorlevel 1 exit /b 1
python -c "import hashlib,pathlib; p=pathlib.Path('release-win_amd64.json'); expected=[line.split()[0] for line in pathlib.Path('SHA256SUMS').read_text().splitlines() if line.split()[-1]==p.name]; assert expected==[hashlib.sha256(p.read_bytes()).hexdigest()]"
if errorlevel 1 exit /b 1
python -c "import json; print(json.load(open('release-win_amd64.json'))['pixtreme_sdk_version'])"
```

Set `CONAN_HOME` to a writable consumer directory. Acquire and verify the
shared recipe archive using the shared acquisition procedure linked above;
export into this Windows Conan home. From the verified extraction directory,
run these cmd commands, replacing `pixtreme_version` with the checked record's
value:

```bat
set "CONAN_HOME=C:\path\to\consumer\deps\conan-home"
set "pixtreme_version=X.Y.Z"
conan export recipes\pixtreme --version=%pixtreme_version%
if errorlevel 1 exit /b 1
conan export recipes\pixtreme-transport --version=%core_version%
if errorlevel 1 exit /b 1
```

In the consumer directory, create `windows-msvc194` with these settings.
Replace `<parallel-jobs>` in the profile and commands with a job count chosen
from the host's CPU configuration and the shared resource allocation received
through `nf resource`:

```ini
[settings]
os=Windows
arch=x86_64
compiler=msvc
compiler.version=194
compiler.runtime=dynamic
compiler.runtime_type=Release
compiler.cppstd=20
build_type=Release
[conf]
tools.build:jobs=<parallel-jobs>
tools.cmake.cmaketoolchain:generator=Ninja
```

Create `tools` to obtain CMake and Ninja through Conan:

```ini
[tool_requires]
!(cmake/*|ninja/*): cmake/3.31.12, ninja/1.13.2
```

Add `VirtualBuildEnv` and `VirtualRunEnv` to the consumer's `generators`, next
to `CMakeDeps` and `CMakeToolchain`. Run the following from the consumer
directory in one cmd process. Keep the generated lock and use it for later
installs. `--build=missing` packages the downloaded SDKs without compiling core
or pixtreme:

```bat
conan lock create . -pr:h windows-msvc194 -pr:h tools -pr:b windows-msvc194 -pr:b tools --lockfile="" --lockfile-out=conan.lock
if errorlevel 1 exit /b 1
conan install . -pr:h windows-msvc194 -pr:h tools -pr:b windows-msvc194 -pr:b tools --lockfile=conan.lock --build=missing --output-folder=build
if errorlevel 1 exit /b 1
call build\conanbuild.bat
if errorlevel 1 exit /b 1
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
if errorlevel 1 exit /b 1
cmake --build build --parallel <parallel-jobs>
if errorlevel 1 exit /b 1
call build\conanrun.bat
if errorlevel 1 exit /b 1
build\example.exe
```

The SDK installs `bin/pixtreme_transport-17.dll` and
`lib/pixtreme_transport.lib` below `pixtreme/transport`; pixtreme supplies its
own `bin/pixtreme-2.dll`. Conan's run environment supplies both DLL directories
for this standalone example. A deployed application chooses its own DLL
layout and search setup and keeps one core and one pixtreme per process.
Call `pixtreme::transport::require_abi()` before other core operations, or
`pixtreme_transport_require_abi(PIXTREME_TRANSPORT_ABI_VERSION, &error)` from C.
The same CMake target propagates the pixtreme dependency to C and C++ consumers.

### Embed a source-built static core

From core 0.32.0, C and C++ consumers can embed a core built from source.
The released SDKs contain shared libraries only: requesting `shared=False`
with `from_source=False` stops with the reason and the remedy to build from
source with `from_source=True` on 0.32.0 or later. Choose the options together:

| Library | Static core and static pixtreme | Static core and prebuilt shared pixtreme |
|---|---|---|
| `pixtreme-transport/*:from_source` | `True` | `True` |
| `pixtreme-transport/*:shared` | `False` | `False` |
| `pixtreme/*:from_source` | `True` | `False` |
| `pixtreme/*:shared` | `False` | `True` |

Each process must contain exactly one core and one pixtreme. The artifact
embedding the static core must be the process's only core: do not also load
the core shared library or another artifact embedding a core. The same rule
applies to an embedded pixtreme, including copies installed through Python
wheels or Conan.

Use a fresh `CONAN_HOME` and the shared recipe acquisition procedure above,
selecting conan-recipes `v1.18.0` for this example. In its verified extraction
directory, export these recipes into that home:

```bash
conan export recipes/pixtreme-transport --version=0.32.0
conan export recipes/pixtreme --version=2.21.0
conan export recipes/cuda-toolkit --version=13.2.2
conan export recipes/libjpeg-turbo --version=3.2.1
```

Set the consumer's requirement to `pixtreme-transport/[>=0.32.0]`; the recipe
resolves pixtreme 2.21.0 from these exports. The core keeps its pixtreme lower
bound; this example's selected version is recorded in the consumer lock.
Building pixtreme from source also takes `libdeflate`, `libwebp` and `zlib`
from ConanCenter, so keep that remote enabled and omit `--no-remote` below.
`libjpeg-turbo` comes from the same verified conan-recipes archive.

Run the remaining commands from your consumer directory, outside the core
checkout. Use the CMake project above and link only `pixtreme::transport`.
For a C source file, enable `project(example LANGUAGES C CXX)` so CMake can
link the core's C++ implementation. Do not add archive paths, CUDA roots or
individual dependency libraries to the consumer.

Detect the native compiler profile and create `static-profile` to select
C++20 and your installed CMake and Ninja. Replace `<parallel-jobs>` in the
profile and build command with a job count chosen from the host's CPU
configuration and the shared resource allocation received through `nf resource`:

```bash
conan profile detect --exist-ok
cat > static-profile <<'PROFILE'
include(default)

[settings]
compiler.cppstd=20

[conf]
tools.build:jobs=<parallel-jobs>
tools.cmake.cmaketoolchain:generator=Ninja

[replace_tool_requires]
cmake/*: cmake/system

[platform_tool_requires]
cmake/system
PROFILE

static_options=(
    -o 'pixtreme-transport/*:from_source=True'
    -o 'pixtreme-transport/*:shared=False'
    -o 'pixtreme/*:from_source=True'
    -o 'pixtreme/*:shared=False'
)
conan lock create . -pr:a ./static-profile "${static_options[@]}" \
    --lockfile="" --lockfile-out conan.lock
```

Source builds fetch private development tags through the GitHub API, from the
`url` that each recipe's `conandata.yml` records for the selected version. The
host's `gh` login must be able to read both source repositories. Export
`recipes_dir` as the `recipes` directory of the verified extraction above, then
place the token in a temporary, mode-0600 `source_credentials.json` only during
install. The credential prefixes come from those recorded URLs:

```bash
(
    set -eu
    set +x
    umask 077
    trap 'rm -f "$CONAN_HOME/source_credentials.json"' EXIT
    python3 - <<'PY'
import json
import os
from pathlib import Path
import re
import subprocess

token = subprocess.run(
    ["gh", "auth", "token"], check=True, capture_output=True, text=True
).stdout.strip()
if not token:
    raise SystemExit("gh could not supply a token")
recipes = Path(os.environ["recipes_dir"])
prefixes = sorted({
    match.group(0)
    for name in ("pixtreme-transport", "pixtreme")
    for match in re.finditer(
        r"https://api\.github\.com/repos/[^/\s\"']+/[^/\s\"']+/",
        (recipes / name / "conandata.yml").read_text(encoding="utf-8"),
    )
})
if len(prefixes) != 2:
    raise SystemExit(f"expected two source repositories, found {prefixes}")
with (Path(os.environ["CONAN_HOME"]) / "source_credentials.json").open("x") as output:
    json.dump({"credentials": [
        {"url": prefix, "token": token} for prefix in prefixes
    ]}, output)
PY
    conan install . -pr:a ./static-profile "${static_options[@]}" \
        --lockfile conan.lock --build=missing --output-folder build-static
)
cmake -S . -B build-static -G Ninja -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_TOOLCHAIN_FILE=build-static/conan_toolchain.cmake
cmake --build build-static --parallel <parallel-jobs>
```

The exit trap removes credentials immediately after install, including on
failure. Keep shell tracing and Conan debug/trace logging disabled. The source
URL entries omit the release asset's `Accept: application/octet-stream` header;
GitHub tarball requests require different headers. Retain the consumer lock
and reuse it for later installs. To select a static core with prebuilt shared
pixtreme, change the last two options to `pixtreme/*:from_source=False` and
`pixtreme/*:shared=True`, then repeat with a separate lock and build directory.

The public `pixtreme::transport` target supplies
`PIXTREME_TRANSPORT_STATIC_DEFINE`, the chosen pixtreme, CUDA's static runtime
components and system link dependencies. The archives contain each library's
own objects; Conan supplies dependencies to the final link. Catch2 is absent
from this consumer graph. Building the core and the C or C++ consumer needs
neither nvcc nor an OS CUDA toolkit. Building pixtreme itself from source uses
nvcc supplied by the build-context `cuda-toolkit` package; the consumer receives
the separate runtime-only package with `compiler=False`. No stage uses an OS
CUDA toolkit or requires a manually set `CUDAToolkit_ROOT`.

Call the ABI check before other core operations, as in the examples below.
A static core creates sessions only from its builtin module entries; see
[Create sessions by module name](#create-sessions-by-module-name) for registering
the list and the `module-not-found` result for names absent from it.

### Build and test from source

Provide Python 3.12 or later, a C++20 compiler, GitHub CLI authentication for the
shared recipes, and the pinned Conan CLI. CMake and Ninja come from Conan's
`cpp/conan/profiles/tools`; do not install them on the host or through pip.
Replace `<parallel-jobs>` in the commands below with a job count chosen from
the host's CPU configuration and the shared resource allocation received
through `nf resource`.

```bash
uv tool install 'conan==2.34.0'
cd python
uv sync --no-install-project
uv pip install --no-deps --reinstall-package pixtreme-transport .
cd ..
python3 cpp/conan/prepare-tests.py
cd python
uv run --no-sync pytest -n <parallel-jobs> --dist=worksteal
uv run --no-sync ruff check
uv run --no-sync ruff format --check
uv run --no-sync mypy
```

The parent `pixtreme` package belongs to its dependency wheel. Development uses
an installed core wheel. After `uv sync` or any source change, repeat both the
`uv sync --no-install-project` and `uv pip install` commands above.

The default pytest selection excludes `slow` distribution build/install tests
and `performance` tests. Run the slow tests with
`uv run --no-sync pytest -n <parallel-jobs> --dist=worksteal -m slow`.
Run affected slow tests when changing distribution builds or packaging, and
include them in integration checks after build, packaging or dependency changes
and before a release. Performance tests remain opt-in with `-m performance`.

For a standalone C++ build, start at the repository root:

```bash
python3 cpp/conan/prepare.py --output build/cpp-build
. ./build/cpp-build/conanbuild.sh
preset=$(python3 -c 'import json; print(json.load(open("build/cpp-build/CMakePresets.json"))["configurePresets"][0]["name"])')
cd cpp
cmake --preset "$preset"
cmake --build --preset "$preset" --parallel <parallel-jobs>
```

For C++ tests, run `python3 cpp/conan/prepare-tests.py`, then prepare a separate
output with `python3 cpp/conan/prepare.py --output build/cpp-tests --tests --offline`.
Activate that output's `conanbuild.sh`, read its preset name as above, and configure
with `cmake --preset "$preset" -DTRANSPORT_BUILD_TESTS=ON`. Build with
`cmake --build --preset "$preset" --parallel <parallel-jobs>`, and run
`CUDA_VISIBLE_DEVICES="" ctest --preset "$preset" --output-on-failure --parallel <parallel-jobs>`.

For Windows source builds, create the checkout with `git clone --config core.autocrlf=false --config core.symlinks=true <repository-url>`. This preserves the license file bytes and keeps `python/LICENSE` and `python/README.md` as symbolic links to the root `LICENSE` and `README.md`, which the wheel ships as its license and description. Creating symbolic links on Windows needs permission to do so (for example, Developer Mode). Before building, check that `python/LICENSE` is a symbolic link; a plain file containing `../LICENSE` produces a wheel with the wrong license text.

On Windows use cmd, run `python cpp\conan\prepare.py --output build\cpp-build`,
then `call build\cpp-build\conanbuild.bat` in that same cmd process. Change to
`cpp` and use the generated preset's configure and build commands. Stop the batch
on any failure. When MSVC is selected, the generated environment includes its
compiler settings. Windows builds require an appropriate host/build profile.

Use `TRANSPORT_CONAN_CACHE` for package storage and `CONAN_HOME` (or
`TRANSPORT_CONAN_HOME`) for Conan configuration; pass the same values to
preparation and builds. Use a fresh build directory after changing them.
Configuration itself never runs Conan. An embedding consumer supplies the single
toolchain and all required dependencies in its own consumer recipe and profile.

### Use the SDK installed with a wheel

The installed wheel also retains its native SDK. Existing C and C++ module
builds can find its CMake package directory with:

```python
import pixtreme.transport

print(pixtreme.transport.get_cmake_dir())
```

Importing `pixtreme.transport` also imports pixtreme and CuPy. If your build environment cannot load them, for example a container user without a writable home directory, read the same directory of the installed wheel from the package metadata instead:

```python
import importlib.metadata

print(importlib.metadata.distribution("pixtreme-transport").locate_file("pixtreme/transport/cmake"))
```

Pass that directory to CMake as `pixtreme_transport_DIR`. In the module's `CMakeLists.txt`, discover the package and link its public target:

```cmake
find_package(pixtreme_transport CONFIG REQUIRED)
target_link_libraries(your_module PRIVATE pixtreme::transport)
```

Source builds obtain the CUDA components from the locked `cuda-toolkit` recipe
in the private `sync-dev-org/conan-recipes` Release. Install the host GitHub CLI
with `sudo apt-get install gh` and authenticate with `gh auth login` before the
first checkout build. The preparation script passes the host token through a temporary
Conan `source_credentials.json` and removes it immediately after downloading;
prepared builds reuse the locked recipes. Creating an sdist copies the locked
CUDA recipe into that archive, so rebuilding it needs no GitHub token. The
[wheel build instructions](wheel-build/README.md) describe how to supply the
verified archive to a container without credentials.

Build-time pixtreme headers and libraries come from its public release SDK through Conan. Python runtime uses the pixtreme wheel. Core checks its own build's pixtreme ABI after its native ABI check.

The CMake package version is the distribution's `X.Y.Z`; an optional requested version sets a lower bound. Keep the core library's location in the module's installed RPATH using a path relative to `$ORIGIN`. Do not copy the core library into the module's wheel.

Include `<pixtreme/transport/abi.h>` and call `pixtreme::transport::require_abi()` after the core shared library loads and before any other core operation. The default argument records the module's build header ABI version. A mismatch with the loaded library throws `std::runtime_error` with both versions.

C programs can use `<pixtreme/transport/c/abi.h>`, `c/error.h` and `c/owner.h`. Enable `LANGUAGES C` in the consumer project and link the same `pixtreme::transport` target. The compatibility query and check do not initialize CUDA or probe a device:

```c
#include <pixtreme/transport/c/abi.h>
#include <pixtreme/transport/c/error.h>

int main(void) {
    pixtreme_transport_error* error = NULL;
    pixtreme_transport_status status =
        pixtreme_transport_require_abi(PIXTREME_TRANSPORT_ABI_VERSION, &error);
    /* On failure, error_get_info exposes the category, reason, why, what and how. */
    pixtreme_transport_handle_release(error, NULL);
    return status == PIXTREME_TRANSPORT_OK ? 0 : 1;
}
```

The shared library exports only C functions. C++ classes, exceptions, virtual hooks and automatic resource release are defined in the installed headers; no C++ object layout crosses the library boundary. The current native ABI is 17. Build C and C++ consumers and modules with ABI-17 headers and check compatibility before other operations. The runtime rejects earlier ABI expectations, including ABI 16. Windows consumers load `pixtreme_transport-17.dll`.

Every extensible C record begins with `struct_size`. Set it to the accessible byte capacity, normally `sizeof(record)`, and pass the record by pointer. Each operation documents its mandatory input prefix and the defaults for omitted trailing fields. Output operations write only complete fields that fit, preserve the capacity, and leave unknown trailing bytes untouched. Status, category and boolean values use fixed-width integer storage. The common C header defines these rules and the immutable string view.

Error outputs belong to each call. Release a previous error before reusing its output variable; errors remain valid across subsequent calls and threads while retained. Input text is borrowed during the call and copied when constructing an error; returned error text is borrowed until the handle's final release. `<pixtreme/transport/errors/check.h>` supplies `check_status(status, error)`, which consumes that error reference and throws the corresponding C++ exception with its original details.

An owner holds a caller's context and release callback. Successful construction transfers one context reference; failed construction leaves it with the caller. Handle copies retain the context, and the last release invokes the callback once on the releasing thread. The caller supplies any synchronization the callback needs. `<pixtreme/transport/owner.h>` provides the header-only `pixtreme::transport::Owner` wrapper; `Handle` supplies shared reference management. Callers that need to inspect a cleanup failure can detach a handle and explicitly release it with an error output; C++ destructors discard cleanup failures.

The GPU C API is in `<pixtreme/transport/c/gpu.h>`. Arrays retain a DLPack `DLTensor` description and an owner handle; creation copies descriptors, not GPU pixels. Image descriptions and typed side data are immutable handles. Frame construction shares already owned arrays; capture copies reusable source arrays into core-owned device memory. Replacement preserves timing and omitted values, shares replacement allocations, and distinguishes omitted PCM from explicitly cleared PCM. Within timing, only an explicitly selected `received_at_ns` changes during replacement.

`FrameTiming.received_at_ns` is optional signed 64-bit nanoseconds recording when an input module received the picture's access unit or capture frame. The input observes it once with C's `pixtreme_transport_now_ns`, C++'s `pixtreme::transport::now_ns()` or Python's `pixtreme.transport.values.now_ns()`. These operations read the same monotonic source as core sender pacing. Synthetic pictures and inputs without a receipt observation leave it absent. This value is separate from the input-number triple, media time and audio positions. Frame copies, capture, replacement, caller buffers and C/Python transfer preserve it. Python `frame.replace(received_at_ns=value)` sets it, `received_at_ns=None` clears it, and omission preserves it. C uses `PIXTREME_TRANSPORT_REPLACE_RECEIVED_AT_NS`; C++ uses the final optional replacement argument, whose outer optional selects replacement and whose inner optional carries or clears the value. The C timing descriptor and inspection record append a `const int64_t* received_at_ns`: NULL means absent, creation copies the value, and inspection borrows it until timing's final release. Older `struct_size` prefixes remain valid, including output records; zero is a present value.

The entrance C API is in `<pixtreme/transport/c/entrance.h>`. Validation borrows a pixtreme PixelBuffer; conversion consumes one source reference on success, rejection and failure. The core waits for recorded source uses, records its reads and releases its source reference on the supplied stream, following pixtreme's storage ownership contract independently of later frame consumers. Colour resolution exposes observed declarations, resolved names and the axes that used defaults separately. Conversion returns the existing GPU frame handle with independent core-owned float32 HWC pixels and a colour resolution handle. Channels retain the input's native model: Y, Cb, Cr for YCbCr, R, G, B for RGB, or Y for luma. A is appended only when the input carries alpha, giving 3/4, 3/4 or 1/2 channels respectively. Conversion delegates to pixtreme's `from_pixel_buffer`, preserving its sample values, bilinear chroma reconstruction and full-range alpha independently of the colour range. RGB has no matrix: the resolved matrix is an empty C string view or an empty C++/Python string, without a matrix fallback flag; the Frame's matrix remains absent. The C++ entrance types wrap these C operations in their headers.

The entrance accepts the following combinations from the pixtreme 2.30.0 PixelBuffer contract (`docs/tokens.md`, "pixel format"). Depth means effective sample bits; alignment describes their position in the storage word. PixelBuffer construction validates depth and alignment before entrance validation. Words use little-endian byte order except the explicit `be` tokens.

| PixelFormat tokens | Effective bit depths | Alignment |
|---|---|---|
| `nv12`, `nv16`, `yv12`, `uyvy422`, `yuyv422`, `vuya`, `yuv410p`, `yuv411p` | 8 | lsb |
| `p010`, `p210`, `yuv444p10msb` | 10 | msb |
| `p012`, `p212`, `yuv444p12msb` | 12 | msb |
| `p016`, `p216` | 16 | lsb |
| `v210` | 10 (three components per 32-bit word) | lsb |
| `yuv420p`, `yuv422p`, `yuv444p`, `yuv440p`, `yuva420p`, `yuva422p`, `yuva444p` | 8–16 | lsb |
| `rgba`, `bgra`, `rgb0`, `bgr0`, `rgb24`, `bgr24`, `argb`, `abgr`, `0rgb`, `0bgr`, `gray8` | 8 | lsb |
| `x2rgb10`, `x2bgr10` | 10 | lsb |
| `gbrp`, `gbrap`, `gray`, `ya` | 8–16 | lsb |
| `rgb48`, `bgr48`, `rgba64`, `bgra64` | 16 | lsb |
| `gray16be`, `ya16be`, `rgb48be`, `rgba64be`, `gbrp16be`, `gbrap16be` | 16 (big-endian words) | lsb |
| `gray16` | 16 | lsb |
| `gray10msb` | 10 | msb |
| `gray12msb` | 12 | msb |

`yuva420p`, `yuva422p`, `yuva444p`, `vuya`, `rgba`, `bgra`, `argb`, `abgr`, `rgba64`, `bgra64`, `rgba64be`, `gbrap`, `gbrap16be`, `ya` and `ya16be` carry straight alpha, normalized over its full effective range without multiplying the colour channels. `rgb0` and `bgr0` ignore their fourth byte; `0rgb` and `0bgr` ignore their first byte. These padding bytes produce no alpha channel. The big-endian tokens change only byte order, preserving the component values and scale of their little-endian counterparts. The continuous 8–16-bit families use 8-bit bytes or 16-bit LSB words for depths above 8. A token/depth/alignment combination absent from pixtreme's contract cannot be constructed as a PixelBuffer; for example, 10-bit `rgb24` and MSB-aligned `yuv420p` are invalid. Callers must handle that rejection at the PixelBuffer construction boundary.

Chroma subsampling factors are `(4, 4)` for `yuv410p`, `(4, 1)` for `yuv411p`, `(1, 2)` for `yuv440p`, `(2, 2)` for 4:2:0 and `(2, 1)` for 4:2:2. Chroma dimensions round up independently. Siting offsets stay in luma-grid units: a chroma sample at `(k, l)` lies at `(sx*k + ox, sy*l + oy)`, with offset zero and no interpolation on an axis whose factor is 1. Thus `center` stays `(0.5, 0.5)` on subsampled axes even when a factor is 4. Conversion uses pixtreme's bilinear reconstruction and clamps at the stored chroma edges.

The input-session C API is in `<pixtreme/transport/c/session_entrance.h>`. A sized callback table registers source open/start/receive/stop/close hooks and a retained context owner. Core owns the reception thread, producer serialization, audio slicing and shutdown. Modules publish GPU frames and PCM through C operations into the caller's drop or blocking buffer. A full blocking buffer waits for a consumer to make room, both for core reception and module callback publication. Session stop or close cancels the pending publication without accepting or counting its picture and leaves the caller's buffer open. PCM writes and consumers can progress while publication waits. Immutable stream-info and snapshot handles carry numbering, PCM and colour observations with each returned frame. C stop registrations let native module waits observe the same stop state as buffer waits. The C++ `session::Entrance` base builds the table in its header; its constructor taking a C buffer handle depends only on C symbols. Pass `buffer.native_handle()` from the C++ `session::FrameBuffer` wrapper. The session retains the caller's C buffer; its lifetime is independent of that wrapper object.

`open_source` prepares reception and returns without waiting for a sender. It
returns no stream information. A thread source returns `StreamInfo` with its
first picture through `receive(stop, info)`; an SDK callback source supplies it
to its first `publish(frame, info)`. Missing information or missing numbering on
that picture is fatal with reason `input-number-unavailable`. `snapshot()` is
empty after each open until `next()` successfully returns a picture. Publication,
call cancellation and failed return preparation leave it empty.

An input module retains PCM received before its first picture. After that
picture initializes the core's PCM store, a thread source submits the retained
PCM at the start of the next `receive`, before handling stop or waiting for more
video. An SDK callback source submits it immediately after the first `publish`.
Use `push_pcm` with source-derived sample positions; the first picture's audio
reference can then resolve those samples. The core refuses PCM before that first
stream observation. Subsequent cadence/audio reanchors use the existing
`reanchor_audio` and `reanchor_audio_at` operations.


Readiness records producer completion and queues repeatable waits on consumer CUDA streams without a host wait. Queue reads after a wait, and retain caller-owned streams and contexts until those reads have retired. An array exported from a frame also retains that frame's completion. DLPack 1.x export accepts both a writer stream and a consumer stream, includes external writes ordered on the writer stream, and keeps descriptors and pixels alive until its deleter with no stream argument is called. Queue consumer reads before calling that deleter; the core records their completion and retires owners without synchronizing the host. Import transfers a managed tensor only on success, accepts byte offsets and scalar byte-aligned contiguous CUDA arrays, and rejects unsupported versions, flags and layouts without calling the producer's deleter.

`pixtreme_transport_collect_completed` releases completed ordinary owners without waiting. Owners with `PIXTREME_TRANSPORT_OWNER_DEFER_COLLECTION` retain their callback and context until `pixtreme_transport_collect_completed_explicit`, wherever their last reference is released. Native collection and ordinary handle release never invoke these callbacks; the explicit collector satisfies their cleanup requirements and receives any callback failure through its status and error outputs. A failed callback is never invoked again. Older owner descriptors default to flags zero, whose final release still invokes the callback on that thread.

The serialization C API is in `<pixtreme/transport/c/serialization.h>`. Immutable target, decision and inspection handles describe wire layouts without consuming pixels or waiting for readiness. Inspection entries expose typed scalar, string, array and plane-shape views; retaining an inspection keeps its views alive independently of the decision. Decisions bind source metadata and the lender's layout, and can be reused with other frame handles and allocations that match them.

Luma-only float32 frames with Y or YA channels are accepted as gray sources.
The effective view reports `source_representation="gray"` and starts its operations
with `neutral-chroma-expand`. The serializer uses pixtreme channel routing to
expand Y to neutral RGB or YCbCr before applying the requested colour conversion,
sampling and quantization. Straight alpha is preserved for RGBA/BGRA, filled to
opaque when absent, or discarded for formats without alpha. Matrix, colorspace
and gamma changes retain their independent authorization requirements.

`pixtreme_transport_serialization_execute` accepts the GPU API's frame handle, a decision, a borrowed destination description and the caller's selected-device stream. Device output writes into that exact lender allocation without a host wait. Host output stages on the device and completes its download before returning. Result inspection exposes the carrier region, lender owner, the GPU API's completion handle and the decision's same requested/effective inspection handles. Keep the result and lender alive through downstream reads; serializer completion does not track an external SDK's read completion. Low-level region preflight and pixel enqueue operations instead leave readiness and lifetime management to the caller. The C++ serialization headers own the corresponding C target, decision and result handles and provide cached C++ inspection values.

The SDK bundles the unchanged [DLPack v1.2 header](https://github.com/dmlc/dlpack/blob/v1.2/include/dlpack/dlpack.h) at `pixtreme/transport/dlpack/dlpack.h` with its Apache-2.0 license at `pixtreme/transport/dlpack/LICENSE`. The header uses upstream's include guard so including that same upstream header before or after the transport headers is supported.

`<pixtreme/transport/c/buffer.h>` provides caller-owned bounded frame buffers. Drop mode evicts the oldest waiting frame and returns an owning metadata record with its coordinate, input number and discontinuity flag; blocking mode waits for room without accepting early. Both modes preserve accepted frames after close so callers can drain them. Buffers own no thread and do not synchronize the GPU. An item retains its frame and optional snapshot owner, while a drop record retains no payload. `<pixtreme/transport/c/stop.h>` provides a shared stop state for consumer and prefill waits: stopping leaves queued input and the buffer's open state intact. The C++ wrappers `session::FrameBuffer` (also named `NativeFrameBuffer`), `StopSource` and `StopToken` are in `<pixtreme/transport/buffer/frame_buffer.h>` and `<pixtreme/transport/buffer/stop.h>`.

Both buffer modes also offer `try_select(classify)` in C++ and Python (`pixtreme_transport_frame_buffer_try_select` in C). The caller supplies the schedule: the classifier returns `expired`, `current` or `future` for each borrowed candidate. Input must have an ordered expired prefix followed by a current or future picture. Under the same storage lock, the operation removes that prefix, transfers at most one current picture and leaves future pictures queued. It does not inspect past the first current or future picture. Callbacks must not reenter or wait on the buffer, process pixels or synchronize the GPU. A failed classifier or return preparation leaves queued frames and counters unchanged.

Selection returns the selected optional frame and an immutable metadata record for each expired frame; it releases each expired picture together with its side data, ancillary data, audio and snapshot. Overflow and expiry records both expose `reason` (`overflow` or `expired`) and `removed_at_ns` from the buffer's monotonic clock, alongside the coordinate, input number and discontinuity flag. `dropped_count` continues to count overflow only; `expired_count` counts explicit selection expiry. The caller collects returned records; the buffer retains no unbounded history. C records use `struct_size`, so older callers can still read their existing fields. The native C++ selection result owns the immutable C record list; indexing, iteration or `expired.values()` creates independent record values and can be retried after a failed read. Python allocates its result object before selecting; reading its `frame` or `expired` properties creates values without consuming that result. Python `FrameBuffer(depth, mode="drop")` also accepts `mode="blocking"`; its selection result exposes `frame` and `expired`.

The sending thread selects one picture according to its selection mode for each output period, performs serialization and codec encoding, then sends it. It prepares no future picture or repeat ahead of that period.

The exit C API is in `<pixtreme/transport/c/exit.h>`. Modules register size-prefixed callback tables for target lifecycle, encode and send; core owns one sending thread, pacing, caller-buffer acquisition and completion tracking. Native frame transforms register `run`, serialization transforms register a non-consuming `prepare`, and native send clocks register both monotonic-time operations. All tables retain their optional context owner. Immutable settings and inspection handles expose audio policy and separate video/audio completion coordinates.

Encode and transform results carry owner references and opaque module contexts; serialization results use the existing typed C result handle. These references survive core GPU work and known encoder/SDK reads, including failures. A successful synchronous send reports both completions; asynchronous senders report video and audio separately using the ticket. Cancellation first stops admission and records a fatal failure, then requires the module's explicit proof that all admitted external reads and callbacks have ended. A stop request or failure alone is no such proof. Close finishes core workers, joins module reads and callbacks, finishes GPU work and returns retained loans before closing the target. It leaves the caller's buffer open. Derived C++ modules keep the same hook interface through the header-only `session::Exit`, `FrameTransform`, `SerializationTransform` and `SendClock` bases.

Native output modules can override C++ `session::Exit::finish_sending(SendFinishReason)`
or append the optional `finish_sending` callback in C's
`pixtreme_transport_exit_callbacks`. The core calls it once per started sending
worker, after the last ordinary `encode`/`send` and existing completion or
cancellation waits, before publishing sender completion. It runs on the sending
thread in the same CUDA context as `encode`/`send`, without the callback mutex or
GIL, so a module can flush its encoder and finalize its output there. An unopened
session or failed open does not call it; reopening starts a new call count.

C++ `SendFinishReason::drained`, `stopped` and `failed` correspond to C's
`PIXTREME_TRANSPORT_SEND_DRAINED`, `PIXTREME_TRANSPORT_SEND_STOPPED` and
`PIXTREME_TRANSPORT_SEND_FAILED`. Acquisition fixes `drained` when it observes
closed, empty input, or `stopped` when it observes a normal stop request. A later
ordinary `close()` does not rewrite a completed drain. A recorded fatal or
explicit cancellation takes precedence as `failed` at hook entry; recoverable
diagnostics alone do not. Receipt scheduling can drain by sending or expiring
pictures, so `drained` does not promise every received picture was sent. The
reason describes that point in time, before the hook and later lifecycle work
have succeeded. Every hook error, including a recoverable C status, becomes
fatal while preserving an earlier fatal.

The hook and its return supply no external read-completion proof. Its SDK reads
belong to the module's existing `cancel_pending`/`finish_cancel` coordination:
it can be invoked after cancellation proof and must start no new external reads
then. Lifecycle `stop_target` notices can arrive during the hook; drain/failure
notices must keep sends and callbacks operational and return promptly without
waiting for the sending worker or every loan return. Existing unconfirmed-stop
errors, close retries and owner retention still apply. The C++ default does no
work. C tables omitting the whole trailing field, and NULL callbacks, retain the
same behavior; the native ABI is 17. Modules relying on finalization here
must raise their core dependency's lower bound to the release providing it.

### Select pictures at the output

`ExitSettings.frame_selection` selects how the sending thread obtains the current
picture from the caller's buffer. `FrameSelectionMode.FIFO` is the default and
takes pictures in queue order. It works with either `Pacing.CORE_CLOCK` or
`Pacing.SDK_CLOCK`; `prefill_frames` controls its initial wait for queued pictures.
`Pacing.NONE` selects periodless FIFO output and requires `prefill_frames=1`.
It waits while the caller's buffer is empty and open, sends each picture once,
and ends after the buffer closes and drains. It neither repeats pictures nor
reads a clock or waits for a pacing deadline. Stop and cancellation end the
input wait without closing the caller's buffer. Nominal frame periods still
determine media time and audio sample counts; missing-PCM deadlines still
start at picture selection. C++ uses `session::Pacing::none`; C uses
`PIXTREME_TRANSPORT_PACING_NONE` (value 2). Core and SDK pacing retain values
0 and 1 and their existing behavior; the native ABI is 17.
`FrameSelectionMode.RECEIPT_SCHEDULE` uses input receipt times and requires
`Pacing.CORE_CLOCK` and `prefill_frames=1`. An SDK-paced output or another prefill
count rejects receipt scheduling as a configuration error (`ValueError` in Python).

C++ uses `session::FrameSelectionMode::fifo` and
`session::FrameSelectionMode::receipt_schedule`. C uses
`PIXTREME_TRANSPORT_FRAME_SELECTION_FIFO` and
`PIXTREME_TRANSPORT_FRAME_SELECTION_RECEIPT_SCHEDULE` in
`pixtreme_transport_exit_settings_options.frame_selection`; the same options
record also carries `max_phase_step_ms` and the optional `has_delay_frames` /
`delay_frames` pair. An explicit presence flag requires the complete value field.

For receipt scheduling, choose the delay `delay_frames` N, caller buffer capacity
M and nominal frame period T. M counts waiting pictures and excludes the retained
picture used for repeats. N must be a positive integer no greater than M;
omitting it uses M. Other selection modes reject an explicit delay. The
scheduling delay is D = N × T: N = 2 at 25 fps gives D = 80 ms, including when
M = 3. Extra storage retains early pictures without increasing the delay.
For input number k, define media time `x_k = (k - k0) * T`. The exit estimates
a local monotonic mapping F from input numbers and receipt observations, then
selects on `s_k = F(x_k) + D`. This estimates input progression and does not
reconstruct an unobserved sender clock. `received_at_ns` stores nanoseconds. Selection starts the
current picture's serialization and encoding immediately. D ends at selection;
the SDK receives the picture after encoding finishes.

```python
from fractions import Fraction
from pixtreme.transport.session import ExitSettings, FrameBuffer, FrameSelectionMode, Pacing

post_buffer = FrameBuffer(2)
settings = ExitSettings(Fraction(1, 25))
settings.pacing = Pacing.CORE_CLOCK
settings.frame_selection = FrameSelectionMode.RECEIPT_SCHEDULE
settings.prefill_frames = 1
settings.max_phase_step_ms = 5.0
# Pass post_buffer and settings to the output module's public factory.
```

The input module stamps each picture with `FrameTiming.received_at_ns`, as
described above. Preserve that value through processing and buffer handoff.
A receipt-scheduled output reports fatal `receipt-time-unavailable` for a picture
without that observation, before accepting or sending it. Synthetic pictures
without receipt observations can use FIFO.

Each picture k is eligible on `[s_k, s_(k+1))`, including the start and excluding
its end. Missing input numbers still have nominal scheduled times. Each cycle
selects at most one unsent eligible picture, leaves future pictures queued and
expires pictures whose intervals have ended. An empty interval repeats the last
picture with silence; expired pictures lose their audio and ancillary data
together without advancing accepted-frame counts or completion coordinates.
A late sender skips elapsed intervals and continues at the current interval.
Input receipt + D is not a second deadline. There is no later-slot rescue.

F smooths receipt jitter and follows input progression with normal corrections
bounded by `max_phase_step_ms` per cycle (default 5 ms). A normal correction
applies to subsequent cycles after the current cycle's wait has begun. Processing
completion, buffer admission, occupancy and audio waiting do not feed F. Missing
input and sender lateness retain the last estimated progression. Persistent
receipt steps, input restarts, numbering-scope changes and cadence changes replace
the one mapping for unsent work, without retaining predecessor mappings or
rewinding consumed input numbers. A reset does not receive normal correction a
second time in the same cycle. D remains N*T throughout.

`Exit.inspection` exposes `frame_selection`, `schedule_scope`,
`phase_correction_ms`, optional `receipt_error_ms` and `schedule_reanchor_count`.
The error is the last observed receipt minus F's expected receipt at observation.
The reanchor counter counts persistent receipt steps since open, excluding input
range, discontinuity and cadence changes. The same fields are available through
`pixtreme_transport_exit_inspection_info` in C. FIFO keeps its four-period catch-up
rule. Output media time and audio sample counts always count sent nominal periods,
independently of F and the playout times.


Call `Exit.take_drops()` to drain the sender's expiry records into immutable
`FrameDropRecords` (`pixtreme_transport_exit_take_drops` in C). Each record exposes
`reason`, `removed_at_ns`, `coordinate`, `input_number` and `discontinuity`.
Sender expiry has reason `expired`; buffer overflow has reason `overflow` and is
reported by the buffer's put result. `dropped_count` counts overflow and
`expired_count` counts expiry. Neither record retains the picture payload.
Indexing or `values()` can be retried on the returned immutable records; a later
`take_drops()` drains only newly accumulated sender records.

### Update output consumers

Use the following settings when moving output consumers to native ABI 17:

| Removed setting | Current configuration |
| --- | --- |
| `follow_input` | Select `frame_selection`: `FIFO` for queue order, or `RECEIPT_SCHEDULE` for a receipt grid on a core-paced output. |
| `target_buffer_wait_ms` | Set `delay_frames` N and buffer capacity M independently; D = N × T, with 1 ≤ N ≤ M. Omission uses M. |
| `max_follow_step_ms` | Set `max_phase_step_ms`, the maximum receipt-phase change per period. |

Read `phase_correction_ms` and `receipt_error_ms` for receipt-grid observations.
The removed `pacing_correction_ms`, `buffer_wait_ms` and C `has_buffer_wait` fields
have no field aliases. Rebuild native consumers and modules with ABI-17 headers,
then use `require_abi()` before other operations. ABI 15 modules are rejected
because their settings conversions can discard an explicit receipt delay.

The installed conformance suite has `CONFORMANCE_REVISION == 26`, exported by
`pixtreme.transport`. Consumers check their expected revision during collection.
`EntranceConformance` checks opening without a first picture, stopping its wait,
absent snapshots until successful delivery, and fatal missing stream observations
or numbering on the first picture. It also preserves decoder scalar metadata
from the source fixture.
Every decoded picture includes `codec`, copied unchanged from the decoder as a string.
The core supplies no codec-name enumeration and does not restrict its spelling; `faulty_reference` is present only when
the decoder supplies it. The suite checks its exact boolean type and value when
present and rejects adding or dropping the key relative to the fixture.
The `slice_type` and `nal_unit_type` fields remain paired when supplied.
`ExitConformance` adds the `frame-selection` and `receipt-time` scenes to check
settings acceptance, configuration errors and fatal receipt-time absence.
The `periodless` scene checks FIFO waits without repeats, input EOF, stopping
empty input, settings validation and media/audio counts for `Pacing.NONE`.
It uses the existing clock injection and observation controls to verify that
periodless sending performs no pacing clock reads or waits. Exclude this scene
through `ConformanceCapabilities.scenes` when the output mode is unsupported.
An adapter declares unsupported output clocks through the existing `core-clock`
and `sdk-clock` scene exclusions in `ConformanceCapabilities`; undeclared scenes
run. These checks use the existing factory and failure-observation controls.
`ConformanceCapabilities.exit_frame_selections` optionally declares a frozenset
of supported `"fifo"` and `"receipt_schedule"` selections; a declared set must
include `"fifo"`. Omitting the field or leaving it `None` keeps all existing
cases. A FIFO-only declaration excludes receipt-success parameters and the
missing-receipt-time case, with the declaring field and mode in each reason.
`exit.unsupported_frame_selection_is_rejected` then checks that a receipt
selection fails at creation or open with a `ValueError` naming `frame_selection`.
It rejects silent acceptance, unrelated setting errors and cleanup-only failures,
and checks that the caller's buffer is unchanged. This rejection case is
inapplicable when selections are undeclared or include receipt scheduling.

## Record session boundaries

Create a `pixtreme.trace.Trace(capacity, 8)` before opening a session and pass it
with `Entrance.set_trace(trace)` or `Exit.set_trace(trace)`. Pass `None` while
closed to detach it. The session retains its own reference; a different value
count raises an argument error and changing the sink while open raises a state
error. Stop all recording sessions before calling `trace.read()` or exporting
its arrays. Recording without a sink does not read the measurement clock.

C callers use `pixtreme_transport_entrance_set_trace` and
`pixtreme_transport_exit_set_trace`; C++ callers use `set_trace` with a
`pixtreme::trace::Trace` or a borrowed `pixtreme_trace_t*`. The public constants
in `<pixtreme/transport/c/trace.h>` describe each row: boundary kind, session
identity, input number, epoch, sequence, work ID, send ticket and flags. The
host measurement timestamps do not change media timing. Input put rows record
both the attempt and whether the buffer accepted it; output rows distinguish
selection, encode and send calls, including failures and asynchronous returns.
A send return does not mean its asynchronous reads completed.

Output modules receive borrowed `trace`, `work_id` and `trace_session_id`
fields in the encode input and output stamp. C++ encode hooks read the same
values with `encoding_trace()`, `encoding_work_id()` and
`encoding_trace_session_id()`; these return null/zero outside the hook. Input
modules borrow the sink and session identity with the entrance's `get_trace`
and `get_trace_session_id` C operations, or `Entrance::trace()` and
`Entrance::trace_session_id()`. Modules can record their own kinds from 1000
upward. Match rows on `(trace_session_id, work_id)`: repeats have distinct work
IDs, reanchor can reset period indices, and reopen restarts work IDs and tickets
with a new session identity. The sink owns no picture, GPU storage or SDK loan.

## Exchange handles between Python extensions

Import `pixtreme.transport.native` explicitly when an extension needs `export_handle(value)` or `import_handle(capsule)`. Ordinary `import pixtreme.transport` loads neither this bridge nor the shared library. Extensions use the capsule names in `<pixtreme/transport/c/python.h>` and the C API instead of registering, casting or inheriting the core's C++ types in a binding framework.

Each exported capsule owns one C handle reference. `import_handle` checks the name, native ABI and handle kind, retains independently, and leaves the capsule usable. Its returned value survives destruction of the capsule. A module that borrows a capsule pointer and uses it later must retain its own reference and release it after its final use. Keep the GIL while creating or using Python objects and capsules; native callbacks use retained C handles. Owners requiring Python cleanup use `PIXTREME_TRANSPORT_OWNER_DEFER_COLLECTION` and GIL-held explicit collection.

The bridge supports frames, completion signals, frame buffers, exit settings, entrance and exit sessions, number scopes, timing, ancillary items/data, frame audio, stream information/snapshots and frame items. Buffers share the same queue; frames share their allocation and completion. Exporting mutable exit settings captures their current values; importing creates independent mutable settings. Small record-only values travel in their enclosing C handle information.

An input read can be cancelled by its caller without stopping the session: `pixtreme_transport_entrance_next_with_options` in C, `Entrance::next(StopToken)` and `Entrance::next(std::stop_token)` in C++, and `Entrance.next(stop=...)` with a `StopSource` in Python. Cancellation affects only that call's wait and take; reception, PCM, the session and the caller's buffer keep running. A recorded fatal error and the first unread recoverable error are still returned before cancellation. A read cancelled before its return preparation starts leaves the queue and the visible snapshot unchanged and returns no item, like the end of the stream; once preparation has started, a successful preparation delivers the picture. Pass a new stop state to read again from the same session.

C++ buffer and input-session wrappers prepare their cached return values before removing a queued frame. If preparation fails, the queue and visible input snapshot stay unchanged. The prepared C operations document their synchronous callback and non-reentrancy rules. A frame left queued after a failed preparation remains subject to ordinary overflow.
