# PocketLlama

On-device LLM inference for Android using [llama.cpp](https://github.com/ggerganov/llama.cpp). Runs GGUF models locally with CPU, Vulkan, or OpenCL backends. Built for benchmarking inference performance across backends and quantizations on Snapdragon hardware.

Group project — Mobile Application Software Development, Tsinghua University, Spring 2026.

---

## Stack

- **Language**: Kotlin (app + JNI wrapper), C++ (native inference)
- **Inference**: llama.cpp (vendored in `lib/src/main/cpp/llama-source/`)
- **Build**: Gradle + CMake, Android NDK 29
- **Min SDK**: 35
- **GPU backends**: OpenCL, Vulkan (see [Backends](#backends))
- **Model format**: GGUF

---

## Project Structure

```
app/                        Android app (UI, MainActivity)
lib/                        JNI wrapper library
  src/main/cpp/
    ai_chat.cpp             C++ JNI bridge into llama.cpp
    CMakeLists.txt          Native build config (backend flags)
  src/main/java/com/arm/aichat/
    InferenceEngine.kt      Public Kotlin interface
    internal/
      InferenceEngineImpl.kt  Singleton JNI wrapper
    gguf/
      GgufMetadataReader.kt   Pure-Kotlin GGUF metadata parser
```

---

## Building

Open in Android Studio (Hedgehog or later) or build from CLI:

```bash
./gradlew assembleDebug
```

Requires NDK 29 installed. Set the NDK path in `local.properties` if not auto-detected:

```
ndk.dir=/path/to/ndk/29.x.x
```

---

## Backends

Configured via `gradle.properties`:

```properties
ENABLE_VULKAN=false
ENABLE_OPENCL=true
```

Only one should be enabled at a time. To use CPU-only, disable both.

**Status on Snapdragon 8 Elite (Adreno 830):**
| Backend | Status |
|---|---|
| CPU | Working |
| OpenCL | Working — slower than CPU for token generation (bandwidth-limited at batch_size=1), faster for prefill |
| Vulkan | Compiles but produces incorrect output — known driver issue with `GL_KHR_cooperative_matrix` on Adreno 830 |

For OpenCL, `libOpenCL.so` must be present at `app/src/main/jniLibs/arm64-v8a/libOpenCL.so`. Pull it from your device:

```bash
adb pull /vendor/lib64/libOpenCL.so app/src/main/jniLibs/arm64-v8a/
```

---

## Running

1. Build and install the APK on your device
2. Launch PocketLlama
3. Tap **Select GGUF File** and pick a `.gguf` model (or download one from HuggingFace)
4. Configure GPU layers (0 = CPU only, max = all layers on GPU) and inference params
5. Tap **Load Model**, then start chatting

Models are copied to app-internal storage on first load and reused on subsequent loads.

### Recommended models

Qwen3-4B quants from [bartowski/Qwen_Qwen3-4B-GGUF](https://huggingface.co/bartowski/Qwen_Qwen3-4B-GGUF):
- `Q8_0` — highest quality, ~4 GB
- `Q4_K_M` — good balance, ~2.5 GB
- `Q2_K` — smallest, ~1.4 GB

---

## Inference Parameters

| Parameter | Default | Description |
|---|---|---|
| GPU layers | 0 | Number of transformer layers offloaded to GPU |
| Temperature | 0.3 | Sampling temperature |
| Max reply tokens | 1024 | Maximum generated tokens per response |
| Batch size | 128 | Prompt processing batch size (n_ubatch) |

Changing GPU layers while a model is loaded requires a reload (app will warn you).

---

## Generation Stats

After each response, tap the **details** link on any assistant message to see:
- Prefill time + tok/s (prompt processing speed)
- Token count + generation tok/s
- Total time + overall tok/s

---

## Debugging

All llama.cpp log output is tagged `ai-chat`:

```bash
adb logcat -s "ai-chat"
```

To confirm GPU offload is active:

```bash
adb logcat -s "ai-chat" | grep -E "n_gpu_layers|offload"
```

To check GPU memory allocation:

```bash
adb shell dumpsys gpu | grep Proc
```
