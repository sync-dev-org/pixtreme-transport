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

The core depends on `pixtreme>=2.0`, which the package installer resolves automatically. Module developers can install the conformance suite's pytest dependency with the `testing` extra:

```bash
pip install 'pixtreme-transport[testing]'
# Or, in a uv project:
uv add 'pixtreme-transport[testing]'
```

The testing package is included in the core wheel and shares its version. Ordinary `import pixtreme.transport` defers native loading and does not import the testing package or pytest.

## Supported environments

PyPI wheels support Linux x86_64 with glibc 2.28 or later (`manylinux_2_28_x86_64`), with separate wheels for CPython 3.12 and 3.13. Runtime use requires an NVIDIA GPU and an NVIDIA driver that supports the CUDA 13 runtime. Native GPU code targets sm 86, sm 89 and sm 120.

PyPI releases contain wheels only; no source distribution is uploaded. Installation on another operating system, CPU architecture or Python version fails with no matching distribution. The package does not provide a CPU-only runtime mode.

## Protocol and codec modules

Install only the module distributions needed by your application. Modules use distribution names of the form `pixtreme-transport-<module>` and import names of the form `pixtreme.transport.<module>`. Each module documents its supported protocols, codecs and additional dependencies. Installing the core alone does not install protocol or codec modules.

## Build a C or C++ module

Modules that use the core's native API build against the SDK in the installed wheel. Find its CMake package directory with:

```python
import pixtreme.transport

print(pixtreme.transport.get_cmake_dir())
```

Pass that directory to CMake as `pixtreme_transport_DIR`. In the module's `CMakeLists.txt`, discover the package and link its public target:

```cmake
find_package(pixtreme_transport CONFIG REQUIRED)
target_link_libraries(your_module PRIVATE pixtreme::transport)
```

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

C++ buffer and input-session wrappers prepare their cached return values before removing a queued frame. If preparation fails, the queue and visible input snapshot stay unchanged. The prepared C operations document their synchronous callback and non-reentrancy rules. A frame left queued after a failed preparation remains subject to ordinary overflow.
