# pixtreme-transport

`pixtreme-transport` provides `pixtreme.transport`, the core of pixtreme's companion for connecting external video streams with `px.core.Frame`. Protocol and codec modules provide stream I/O, decoding and encoding through separately installed distributions.

One core wheel includes the C++ shared library, public C headers and header-only C++ wrappers, CMake package configuration, nanobind Python bindings and the `pixtreme.transport.testing` package. The core supplies the common contracts and SDK that modules use.

## Install

```bash
pip install pixtreme-transport
```

For a uv project:

```bash
uv add pixtreme-transport
```

The core depends on `pixtreme>=2.22.1`, which the package installer resolves automatically. Module developers can install the conformance suite's pytest dependency with the `testing` extra:

```bash
pip install 'pixtreme-transport[testing]'
# Or, in a uv project:
uv add 'pixtreme-transport[testing]'
```

The testing package is included in the core wheel and shares its version. Ordinary `import pixtreme.transport` defers native loading and does not import the testing package or pytest.

## Supported environments

PyPI wheels support Linux x86_64 with glibc 2.28 or later (`manylinux_2_28_x86_64`), with separate wheels for CPython 3.12 and 3.13. Runtime use requires an NVIDIA GPU and an NVIDIA driver that supports the CUDA 13 runtime.

PyPI releases contain wheels only; no source distribution is uploaded. Installation on another operating system, CPU architecture or Python version fails with no matching distribution. The package does not provide a CPU-only runtime mode.

## Protocol and codec modules

Install only the module distributions needed by your application. Modules use distribution names of the form `pixtreme-transport-<module>` and import names of the form `pixtreme.transport.<module>`. Each module documents its supported protocols, codecs and additional dependencies. Installing the core alone does not install protocol or codec modules.

## Create sessions by module name

C and C++ programs can create an entrance or exit session from a module's name. `<pixtreme/transport/c/module.h>` declares `pixtreme_transport_module_create_entrance` and `pixtreme_transport_module_create_exit`. Each takes the module name (`[a-z][a-z0-9_]*`, the same `<module>` as the import name), a sized request record (the device and the caller's frame buffer, plus the core exit settings for an exit) and an array of name/value string settings. The core selects a registered builtin entry when its descriptor declares that name. For other names, the shared core opens only `<module>/lib/libpixtreme_transport_<module>_entry.so` under the installed `pixtreme/transport` directory, located from the core's own shared library, and calls its `pixtreme_transport_module_entry` function. Both routes check the descriptor version and the core ABI. A core statically embedded from a source build returns `module-not-found` for names absent from its builtin list, including when no list was registered. It does not search or load shared module libraries. The core does not scan directories or read environment variables. On success the call returns an unopened session; start it with the existing open operation.

Failures carry stable reasons: `invalid-module-name`, `module-not-found` (its guidance names the distribution `pixtreme-transport-<module>` to install), `module-load-failed`, `module-entry-missing`, `module-abi-mismatch`, `module-direction-unavailable`, `invalid-module-option` and `module-create-failed`. Each call creates an independent instance that can be used on another thread; a loaded module library stays loaded until the process ends. Module discovery is implemented for Linux.

Setting names use `.` for nesting, for example `connection.channel`. Values are text: decimal integers with an optional leading `-` (no `+`, spaces, `0x` or fractions), `true` or `false`, and rationals `<numerator>/<denominator>` with a positive denominator. Each module documents its keys. Errors name the key and never include its value. Module authors and C++ callers can use the header-only `<pixtreme/transport/module.h>`, which provides `pixtreme::transport::module::Options` for parsing and validating settings, `create_entrance` and `create_exit` returning an owning `Handle`, and `creation_boundary` for converting setting errors into the C error contract.

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
`pixtreme/transport/{include,lib,cmake}`. Current SDKs support Linux x86_64 with
glibc 2.28 or later. Neither core nor pixtreme is compiled when obtaining these
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
C++20, at most four build jobs and your installed CMake and Ninja:

```bash
conan profile detect --exist-ok
cat > static-profile <<'PROFILE'
include(default)

[settings]
compiler.cppstd=20

[conf]
tools.build:jobs=4
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
cmake --build build-static --parallel 4
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
first checkout build. The bootstrap passes the host token through a temporary
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

The shared library exports only C functions. C++ classes, exceptions, virtual hooks and automatic resource release are defined in the installed headers; no C++ object layout crosses the library boundary. Native ABI 6 starts the C compatibility set. Modules built for native ABI 5 must be rebuilt once for this set; the former SONAME and C++ symbols are unavailable.

Every extensible C record begins with `struct_size`. Set it to the accessible byte capacity, normally `sizeof(record)`, and pass the record by pointer. Each operation documents its mandatory input prefix and the defaults for omitted trailing fields. Output operations write only complete fields that fit, preserve the capacity, and leave unknown trailing bytes untouched. Status, category and boolean values use fixed-width integer storage. The common C header defines these rules and the immutable string view.

Error outputs belong to each call. Release a previous error before reusing its output variable; errors remain valid across subsequent calls and threads while retained. Input text is borrowed during the call and copied when constructing an error; returned error text is borrowed until the handle's final release. `<pixtreme/transport/errors/check.h>` supplies `check_status(status, error)`, which consumes that error reference and throws the corresponding C++ exception with its original details.

An owner holds a caller's context and release callback. Successful construction transfers one context reference; failed construction leaves it with the caller. Handle copies retain the context, and the last release invokes the callback once on the releasing thread. The caller supplies any synchronization the callback needs. `<pixtreme/transport/owner.h>` provides the header-only `pixtreme::transport::Owner` wrapper; `Handle` supplies shared reference management. Callers that need to inspect a cleanup failure can detach a handle and explicitly release it with an error output; C++ destructors discard cleanup failures.

The GPU C API is in `<pixtreme/transport/c/gpu.h>`. Arrays retain a DLPack `DLTensor` description and an owner handle; creation copies descriptors, not GPU pixels. Image descriptions and typed side data are immutable handles. Frame construction shares already owned arrays; capture copies reusable source arrays into core-owned device memory. Replacement preserves timing and omitted values, shares replacement allocations, and distinguishes omitted PCM from explicitly cleared PCM.

The entrance C API is in `<pixtreme/transport/c/entrance.h>`. Surface handles copy explicit source geometry and colour observations and retain allocation owners and producer readiness. Colour resolution exposes observed codes, resolved names and the axes that used defaults separately. A surface loan retains its return callback context and processing stream owner until the last source read finishes; `try_release` never waits on the host. Conversion returns the existing GPU frame handle with independent core-owned float32 Y/Cb/Cr pixels and a colour resolution handle. The protocol module returns its source loan separately from later frame consumers. The C++ entrance types wrap these C operations in their headers.

The input-session C API is in `<pixtreme/transport/c/session_entrance.h>`. A sized callback table registers source open/start/receive/stop/close hooks and a retained context owner. Core owns the reception thread, producer serialization, audio slicing and shutdown. Modules publish GPU frames and PCM through C operations into the caller's drop buffer. Immutable stream-info and snapshot handles carry numbering, PCM and colour observations with each returned frame. C stop registrations let native module waits observe the same stop state as buffer waits. The C++ `session::Entrance` base builds the table in its header; its constructor taking a C buffer handle depends only on C symbols. Pass `buffer.native_handle()` from the C++ `session::FrameBuffer` wrapper. The session retains the caller's C buffer; its lifetime is independent of that wrapper object.

Readiness records producer completion and queues repeatable waits on consumer CUDA streams without a host wait. Queue reads after a wait, and retain caller-owned streams and contexts until those reads have retired. An array exported from a frame also retains that frame's completion. DLPack 1.x export accepts both a writer stream and a consumer stream, includes external writes ordered on the writer stream, and keeps descriptors and pixels alive until its deleter with no stream argument is called. Queue consumer reads before calling that deleter; the core records their completion and retires owners without synchronizing the host. Import transfers a managed tensor only on success, accepts byte offsets and scalar byte-aligned contiguous CUDA arrays, and rejects unsupported versions, flags and layouts without calling the producer's deleter.

`pixtreme_transport_collect_completed` releases completed ordinary owners without waiting. Owners with `PIXTREME_TRANSPORT_OWNER_DEFER_COLLECTION` retain their callback and context until `pixtreme_transport_collect_completed_explicit`, wherever their last reference is released. Native collection and ordinary handle release never invoke these callbacks; the explicit collector satisfies their cleanup requirements and receives any callback failure through its status and error outputs. A failed callback is never invoked again. Older owner descriptors default to flags zero, whose final release still invokes the callback on that thread.

The serialization C API is in `<pixtreme/transport/c/serialization.h>`. Immutable target, decision and inspection handles describe wire layouts without consuming pixels or waiting for readiness. Inspection entries expose typed scalar, string, array and plane-shape views; retaining an inspection keeps its views alive independently of the decision. Decisions bind source metadata and the lender's layout, and can be reused with other frame handles and allocations that match them.

`pixtreme_transport_serialization_execute` accepts the GPU API's frame handle, a decision, a borrowed destination description and the caller's selected-device stream. Device output writes into that exact lender allocation without a host wait. Host output stages on the device and completes its download before returning. Result inspection exposes the carrier region, lender owner, the GPU API's completion handle and the decision's same requested/effective inspection handles. Keep the result and lender alive through downstream reads; serializer completion does not track an external SDK's read completion. Low-level region preflight and pixel enqueue operations instead leave readiness and lifetime management to the caller. The C++ serialization headers own the corresponding C target, decision and result handles and provide cached C++ inspection values.

The SDK bundles the unchanged [DLPack v1.2 header](https://github.com/dmlc/dlpack/blob/v1.2/include/dlpack/dlpack.h) at `pixtreme/transport/dlpack/dlpack.h` with its Apache-2.0 license at `pixtreme/transport/dlpack/LICENSE`. The header uses upstream's include guard so including that same upstream header before or after the transport headers is supported.

`<pixtreme/transport/c/buffer.h>` provides caller-owned bounded frame buffers. Drop mode evicts the oldest waiting frame and returns an owning metadata record with its coordinate, input number and discontinuity flag; blocking mode waits for room without accepting early. Both modes preserve accepted frames after close so callers can drain them. Buffers own no thread and do not synchronize the GPU. An item retains its frame and optional snapshot owner, while a drop record retains no payload. `<pixtreme/transport/c/stop.h>` provides a shared stop state for consumer and prefill waits: stopping leaves queued input and the buffer's open state intact. The C++ wrappers `session::FrameBuffer` (also named `NativeFrameBuffer`), `StopSource` and `StopToken` are in `<pixtreme/transport/buffer/frame_buffer.h>` and `<pixtreme/transport/buffer/stop.h>`.

The exit C API is in `<pixtreme/transport/c/exit.h>`. Modules register size-prefixed callback tables for target lifecycle, encode and send; core owns the worker threads, pacing, caller-buffer acquisition and completion tracking. Native frame transforms register `run`, serialization transforms register a non-consuming `prepare`, and native send clocks register both monotonic-time operations. All tables retain their optional context owner. Immutable settings and inspection handles expose audio policy and separate video/audio completion coordinates.

Encode and transform results carry owner references and opaque module contexts; serialization results use the existing typed C result handle. These references survive core GPU work and known encoder/SDK reads, including failures. A successful synchronous send reports both completions; asynchronous senders report video and audio separately using the ticket. Cancellation first stops admission and records a fatal failure, then requires the module's explicit proof that all admitted external reads and callbacks have ended. A stop request or failure alone is no such proof. Close finishes core workers, joins module reads and callbacks, finishes GPU work and returns retained loans before closing the target. It leaves the caller's buffer open. Derived C++ modules keep the same hook interface through the header-only `session::Exit`, `FrameTransform`, `SerializationTransform` and `SendClock` bases.


## Exchange handles between Python extensions

Import `pixtreme.transport.native` explicitly when an extension needs `export_handle(value)` or `import_handle(capsule)`. Ordinary `import pixtreme.transport` loads neither this bridge nor the shared library. Extensions use the capsule names in `<pixtreme/transport/c/python.h>` and the C API instead of registering, casting or inheriting the core's C++ types in a binding framework.

Each exported capsule owns one C handle reference. `import_handle` checks the name, native ABI and handle kind, retains independently, and leaves the capsule usable. Its returned value survives destruction of the capsule. A module that borrows a capsule pointer and uses it later must retain its own reference and release it after its final use. Keep the GIL while creating or using Python objects and capsules; native callbacks use retained C handles. Owners requiring Python cleanup use `PIXTREME_TRANSPORT_OWNER_DEFER_COLLECTION` and GIL-held explicit collection.

The bridge supports frames, completion signals, frame buffers, exit settings, entrance and exit sessions, number scopes, timing, ancillary items/data, frame audio, stream information/snapshots and frame items. Buffers share the same queue; frames share their allocation and completion. Exporting mutable exit settings captures their current values; importing creates independent mutable settings. Small record-only values travel in their enclosing C handle information.

An input read can be cancelled by its caller without stopping the session: `pixtreme_transport_entrance_next_with_options` in C, `Entrance::next(StopToken)` and `Entrance::next(std::stop_token)` in C++, and `Entrance.next(stop=...)` with a `StopSource` in Python. Cancellation affects only that call's wait and take; reception, PCM, the session and the caller's buffer keep running. A recorded fatal error and the first unread recoverable error are still returned before cancellation. A read cancelled before its return preparation starts leaves the queue and the visible snapshot unchanged and returns no item, like the end of the stream; once preparation has started, a successful preparation delivers the picture. Pass a new stop state to read again from the same session.

C++ buffer and input-session wrappers prepare their cached return values before removing a queued frame. If preparation fails, the queue and visible input snapshot stay unchanged. The prepared C operations document their synchronous callback and non-reentrancy rules. A frame left queued after a failed preparation remains subject to ordinary overflow.
