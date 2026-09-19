---
title: "Running llama.cpp on Android"
date: 2026-09-19 00:00:00 -0700
categories: [AI, Local LLMs]
tags: [android, llama.cpp, on-device-ai, ndk, jni, kotlin]
lang: en
---

I've been experimenting with on-device LLMs, and [llama.cpp](https://github.com/ggml-org/llama.cpp)
is the most practical way to run one on a phone: fast on CPU, GGUF quantization keeps memory
usage sane, and it builds cleanly with the NDK once you know the right flags.

Here's the setup, step by step, with the gotchas that aren't in the official docs.

> Every step below is boilerplate I've actually built and run, not theory — safe to hand this
> whole post to a coding agent and ask it to set it all up in your project. Skim the recap at
> the end afterward so you know what to check if generation comes out slow or broken.
{: .prompt-tip }

You need: Android Studio, NDK, CMake, and a 64-bit device or emulator (`arm64-v8a` or `x86_64`).

## 1. Add llama.cpp as a submodule

```bash
git submodule add https://github.com/ggml-org/llama.cpp.git
git submodule update --init --recursive
```

Pin the version. llama.cpp's C API changes often enough that tracking `main` will break your
build eventually. Clone with `--recurse-submodules` or the folder is empty.

## 2. Gradle config

In `app/build.gradle.kts`:

```kotlin
ndkVersion = "29.0.13113456"

defaultConfig {
    minSdk = 24
    ndk { abiFilters += listOf("arm64-v8a", "x86_64") }

    externalNativeBuild {
        cmake {
            arguments += "-DCMAKE_BUILD_TYPE=Release"
            arguments += "-DBUILD_SHARED_LIBS=ON"
            arguments += "-DLLAMA_BUILD_COMMON=ON"
            arguments += "-DLLAMA_OPENSSL=OFF"
            arguments += "-DGGML_NATIVE=OFF"
            arguments += "-DGGML_BACKEND_DL=ON"
            arguments += "-DGGML_CPU_ALL_VARIANTS=ON"
            arguments += "-DGGML_LLAMAFILE=OFF"
        }
    }
}

externalNativeBuild {
    cmake {
        path("src/main/cpp/CMakeLists.txt")
        version = "3.31.6"
    }
}

packaging {
    jniLibs { useLegacyPackaging = true }
}
```

| Flag | Why |
|---|---|
| `BUILD_SHARED_LIBS=ON` | Ship `.so` files, not a static blob |
| `LLAMA_BUILD_COMMON=ON` | Need `common` for chat templates, tokenizer, sampler |
| `LLAMA_OPENSSL=OFF` | No OpenSSL in the NDK |
| `GGML_NATIVE=OFF` | `-march=native` targets the build machine, not the phone |
| `GGML_BACKEND_DL=ON` | CPU backends built as separate `.so`, loaded at runtime |
| `GGML_CPU_ALL_VARIANTS=ON` | Builds multiple CPU variants, picks the best one on-device |
| `GGML_LLAMAFILE=OFF` | Not needed |

> `useLegacyPackaging = true` is required. With `GGML_BACKEND_DL=ON`, backends are `dlopen`'d
> by file path at runtime. Without legacy packaging they stay compressed inside the APK with
> no path to open — zero backends load, silently, and the model never runs.
{: .prompt-warning }

Pin `ndkVersion` and CMake `version` explicitly so builds are reproducible across machines.

### This is where the tokens/sec actually come from

My first working build was painfully slow — a couple tokens a second, borderline unusable.
Two flags fixed almost all of it:

- **`GGML_CPU_ALL_VARIANTS=ON` + `GGML_BACKEND_DL=ON`.** `GGML_NATIVE=OFF` is required to
  cross-compile, but on its own it means one generic build runs on every phone — the slowest
  possible kernel, since it can't assume any CPU feature exists. `ALL_VARIANTS` builds a
  separate kernel per feature set (NEON dotprod, i8mm, SVE, ...) and `BACKEND_DL` picks the
  right one for the actual device at load time. This pair alone was most of the win.
- **`GGML_CPU_KLEIDIAI=ON` (ARM only, see step 3).** Arm's own int4/int8 matmul kernels,
  tuned for quantized models. Basically free speed if you're already running a quantized GGUF.

`GGML_OPENMP=ON` on ARM (also step 3) helped further — it's a faster threading backend than
plain pthreads for the matmul-heavy inner loop on Android.

## 3. CMakeLists

`app/src/main/cpp/CMakeLists.txt` pulls in llama.cpp and builds your bridge library:

```cmake
cmake_minimum_required(VERSION 3.31.6)
project("llama-bridge" VERSION 1.0.0 LANGUAGES C CXX)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED true)

if(DEFINED ANDROID_ABI)
    if(ANDROID_ABI STREQUAL "arm64-v8a")
        set(GGML_CPU_KLEIDIAI ON)
        set(GGML_OPENMP ON)
    elseif(ANDROID_ABI STREQUAL "x86_64")
        set(GGML_CPU_KLEIDIAI OFF)
        set(GGML_OPENMP OFF)
    endif()
endif()

set(LLAMA_SRC ${CMAKE_CURRENT_LIST_DIR}/../../../../llama.cpp)
add_subdirectory(${LLAMA_SRC} build-llama)

add_library(${CMAKE_PROJECT_NAME} SHARED llm_bridge.cpp)

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
        ${LLAMA_SRC} ${LLAMA_SRC}/common ${LLAMA_SRC}/include
        ${LLAMA_SRC}/ggml/include ${LLAMA_SRC}/ggml/src)

target_link_libraries(${CMAKE_PROJECT_NAME} llama llama-common android log)
```

Notes:
- **KleidiAI** (Arm's optimized matmul kernels) and **OpenMP** only apply to ARM — gate them
  on `ANDROID_ABI`.
- `common` isn't part of llama.cpp's public headers — include it explicitly or you'll get
  `fatal error: common.h: No such file`.
- First build compiles ggml per ABI, so it's slow. Trim `abiFilters` to one ABI while
  iterating.

## 4. The JNI bridge

You need about eight functions. Keep model, context, batch, and sampler as file-level statics
— there's only one model loaded at a time:

```cpp
static llama_model              * g_model;
static llama_context            * g_context;
static llama_batch                g_batch;
static common_chat_templates_ptr  g_chat_templates;
static common_sampler           * g_sampler;
```

**Startup** — load backends and init:

```cpp
Java_com_example_llmdemo_llm_LlamaEngineImpl_init(JNIEnv *env, jobject, jstring nativeLibDir) {
    llama_log_set(aichat_android_log_callback, nullptr);

    const auto *path = env->GetStringUTFChars(nativeLibDir, nullptr);
    ggml_backend_load_all_from_path(path);   // finds libggml-cpu-*.so
    env->ReleaseStringUTFChars(nativeLibDir, path);

    llama_backend_init();
}
```

Pass `context.applicationInfo.nativeLibraryDir` for `nativeLibDir` — that's where
`GGML_BACKEND_DL` put the backend `.so` files. Route llama.cpp's logs into logcat with
`llama_log_set`; debugging native inference without this is painful.

**Loading a model:**

```cpp
Java_..._load(JNIEnv *env, jobject, jstring jmodel_path) {
    const auto *model_path = env->GetStringUTFChars(jmodel_path, nullptr);
    auto *model = llama_model_load_from_file(model_path, llama_model_default_params());
    env->ReleaseStringUTFChars(jmodel_path, model_path);
    if (!model) return 1;
    g_model = model;
    return 0;
}
```

Then build the context, capping thread count:

```cpp
const int n_threads = std::max(2, std::min(4, (int) sysconf(_SC_NPROCESSORS_ONLN) - 2));

llama_context_params ctx_params = llama_context_default_params();
ctx_params.n_ctx           = 8192;
ctx_params.n_batch         = 512;
ctx_params.n_threads       = n_threads;
ctx_params.n_threads_batch = n_threads;

g_context        = llama_init_from_model(g_model, ctx_params);
g_batch          = llama_batch_init(512, 0, 1);
g_chat_templates = common_chat_templates_init(g_model, "");
```

> Don't grab every core. llama.cpp's thread pool is synchronous — every step waits for the
> slowest thread — so throwing in a phone's slow efficiency cores can make generation *slower*
> than using fewer, faster ones, on top of heating up the phone and starving the UI thread.
> 2–4 threads (skip the small cores) was the sweet spot on every device I tried.
{: .prompt-tip }

Return int error codes, not exceptions — simpler across JNI.

**Generating** — one token per call, `nullptr` means done:

```cpp
Java_..._generateNextToken(JNIEnv *env, jobject) {
    const auto new_token_id = common_sampler_sample(g_sampler, g_context, -1);
    common_sampler_accept(g_sampler, new_token_id, true);

    common_batch_clear(g_batch);
    common_batch_add(g_batch, new_token_id, current_position, {0}, true);
    if (llama_decode(g_context, g_batch)) return nullptr;
    current_position++;

    if (llama_vocab_is_eog(llama_model_get_vocab(g_model), new_token_id)) return nullptr;

    cached_token_chars += common_token_to_piece(g_context, new_token_id);
    // buffer until UTF-8 is complete, then emit — see below
}
```

A token is bytes, not a character — multi-byte characters split across tokens. Calling
`NewStringUTF` on a half sequence gives garbage or crashes. Buffer and check validity first:

```cpp
const auto *bytes = (const unsigned char *) cached_token_chars.c_str();
bool valid = true;
while (*bytes && valid) {
    int len;
    if      ((*bytes & 0x80) == 0x00) len = 1;
    else if ((*bytes & 0xE0) == 0xC0) len = 2;
    else if ((*bytes & 0xF0) == 0xE0) len = 3;
    else if ((*bytes & 0xF8) == 0xF0) len = 4;
    else { valid = false; break; }
    for (int i = 1; i < len && valid; i++)
        if ((bytes[i] & 0xC0) != 0x80) valid = false;
    bytes += len;
}

if (valid) {
    jstring result = env->NewStringUTF(cached_token_chars.c_str());
    cached_token_chars.clear();
    return result;
}
return env->NewStringUTF("");   // incomplete — wait for next token
```

Empty string = "nothing yet"; `nullptr` = "done". Keep them distinct.

**Context full?** Discard the oldest half and shift the rest down, but never touch the system
prompt:

```cpp
static void shift_context() {
    const int n_discard = (current_position - system_prompt_position) / 2;
    llama_memory_seq_rm (llama_get_memory(g_context), 0,
                         system_prompt_position, system_prompt_position + n_discard);
    llama_memory_seq_add(llama_get_memory(g_context), 0,
                         system_prompt_position + n_discard, current_position, -n_discard);
    current_position -= n_discard;
}
```

## 5. Kotlin side

Declare the natives and load the library:

```kotlin
internal class LlamaEngineImpl private constructor(
    private val nativeLibDir: String
) : LlamaEngine {

    private external fun init(nativeLibDir: String)
    private external fun load(modelPath: String): Int
    private external fun prepare(): Int
    private external fun processUserPrompt(userPrompt: String): Int
    @FastNative private external fun generateNextToken(): String?
    private external fun unload()
}
```

> Names must match exactly. `Java_com_example_llmdemo_llm_LlamaEngineImpl_load` has to match
> `com.example.llmdemo.llm.LlamaEngineImpl.load` — package, class, method. Get it wrong and
> it's `UnsatisfiedLinkError` at runtime, not build time.
{: .prompt-warning }

**`@FastNative` only on genuinely fast calls.** It skips a thread-state transition but blocks
GC suspension for the call's duration. `generateNextToken` (one sampling step) qualifies.
`load()` (reads a GB off disk) does not — mark it `@FastNative` and you freeze the UI thread.

**Serialize everything onto one thread.** The llama.cpp context isn't thread-safe:

```kotlin
private val llamaDispatcher = Dispatchers.IO.limitedParallelism(1)
private val llamaScope = CoroutineScope(llamaDispatcher + SupervisorJob())

init {
    llamaScope.launch {
        System.loadLibrary("llama-bridge")
        init(nativeLibDir)   // context.applicationInfo.nativeLibraryDir
    }
}
```

Callers can be on any coroutine context — `withContext`/`flowOn` handle the hop safely.

**Stream tokens as a Flow:**

```kotlin
override fun sendUserPrompt(message: String, predictLength: Int): Flow<String> = flow {
    processUserPrompt(message)
    val accumulator = StringBuilder()
    var tokenCount = 0
    while (!_cancelGeneration && tokenCount < predictLength) {
        val token = generateNextToken() ?: break
        tokenCount++
        accumulator.append(token)
        if (token.isNotEmpty()) emit(token)
    }
    nativeAddTurn("assistant", accumulator.toString(), false)
}.flowOn(llamaDispatcher)
```

Catch cancellation and still commit the partial turn — if the user hits stop mid-generation,
those tokens are already in the KV cache. Skip the commit and Kotlin's view of the
conversation drifts from the model's.

**The conversation lives in the native KV cache, not in Kotlin.** Don't keep a message list
and re-send it each turn — that re-encodes the whole history every message, which on a phone
is the difference between a snappy reply and a ten-second stall.

## 6. Getting a model onto the device

Download a GGUF file on first run rather than bundling it — a 500 MB+ asset is a bad install
and hits Play Store size limits. Android's `DownloadManager` handles this well: survives app
death, shows progress, resumes on failure.

Sizes that work on a phone (Qwen3.5, as an example):

| Model | Size | Feel |
|---|---|---|
| 0.8B Q4_K_M | ~500 MB | Fast everywhere |
| 2B Q4_K_M | ~1.4 GB | Good balance on a modern phone |
| 4B UD-IQ2_M | ~1.7 GB | Noticeably better, wants a flagship device |

Rule of thumb: RAM needed ≈ file size + KV cache. A 4 GB phone running a 2 GB model gets
killed by the OS on backgrounding — run inference in a foreground service if it needs to
survive that.

## Recap

1. `GGML_CPU_ALL_VARIANTS=ON` + `GGML_BACKEND_DL=ON` — this is most of your tokens/sec.
2. `useLegacyPackaging = true`, or those backends silently fail to load.
3. One thread for all JNI calls; cap generation at 2–4 threads, skipping efficiency cores.
4. Buffer incomplete UTF-8 before `NewStringUTF`.
5. Don't `@FastNative` slow calls like `load()`.

Once the bridge works, the model is just a suspend function that streams strings — everything
built on top is plain Kotlin.
