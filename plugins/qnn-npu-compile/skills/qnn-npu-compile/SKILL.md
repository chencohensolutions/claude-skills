---
name: qnn-npu-compile
description: Compile ONNX models to run on Qualcomm NPUs (Hexagon HTP, via the QNN / QAIRT SDK) for Android apps, and run them with sherpa-onnx or ONNX Runtime. Use when the user wants a model (speech, audio, vision, small transformer) on a Snapdragon phone's NPU for speed or battery, or asks how to compile/convert/quantize a model for QNN, HTP, Hexagon, Snapdragon or "the NPU".
---

# Compiling models for Qualcomm NPUs (QNN HTP)

A procedure, not a script: every model needs its own export and checks. It follows how sherpa-onnx compiles its
published NPU models (`k2-fsa/sherpa-onnx`, `.github/workflows/export-*-qnn.yaml`, `.github/scripts/export-qnn/`,
`scripts/qnn/`), which are known to work. Read the matching workflow there before starting; adapt, don't invent.

## 0. Decide whether the NPU is worth it

- **Measure first** on the target phone: CPU % and wall time of the model as it runs now (`/proc/<pid>/stat` ticks
  every 2 s, `top -H -p <pid>` per thread), and battery over a fixed task if power is the goal. The NPU pays off for
  big, steady matrix work (encoders, long batch passes); for a small model on 1–2 threads the gain may be small.
- **Static shapes only.** The NPU runs fixed-size graphs: fixed batch (1), fixed input length (chunk size, or 30 s
  for Whisper's encoder). A model whose input length varies needs a fixed chunk, padding, or several graphs.
- **One chip family per build.** A context binary is compiled for one Hexagon architecture (v73, v75, v79, v81…) and
  runs only there; other phones (and Exynos / Tensor / MediaTek) keep the CPU path. Plan a CPU fallback always.
- **Size limits.** Very large models (over about 1B parameters in fp16) may not fit one context: they need 8-bit
  weights (w8a16) and/or splitting into several graphs. sherpa-onnx's own Whisper builds stop at medium (769M).

Chips (from sherpa-onnx `scripts/qnn/device_info.py`): SM8550 Snapdragon 8 Gen 2 (v73), SM8650 8 Gen 3 (v75),
**SM8750 8 Elite (v79)**, SM8850 (v81), QCS9100 (v73). SM8350 (v68) has no fp16: skip it. Find a phone's chip with
`adb shell getprop ro.soc.model`.

## 1. Tools (Linux x86-64 only)

The QNN SDK (QAIRT) runs on **Linux x86-64** (or Windows), not macOS. On a Mac: a Linux VM/cloud box, or Docker with
`--platform linux/amd64` (slow: emulated). sherpa-onnx builds a Docker image for it (`Dockerfile-2.40`,
`qnn-sdk-docker-build.yaml`).

- QAIRT SDK, e.g. 2.40.0.251030: from Qualcomm (free account), or the copy sherpa-onnx's CI downloads
  (`huggingface.co/csukuangfj/qnn-toolkit`, `v2.40.0.251030.zip`). Check its license allows your use.
- `source qairt/<version>/bin/envsetup.sh` (sets `QNN_SDK_ROOT`), then
  `sudo ${QNN_SDK_ROOT}/bin/check-linux-dependency.sh` and `python3 ${QNN_SDK_ROOT}/bin/check-python-dependency`.
- Python 3.10 venv with `numpy<2 onnx==1.17.0 onnxruntime onnxsim` plus the model's own export deps.
- Android NDK on `PATH` (`ndk-build`), for the aarch64 model libraries; `LDFLAGS="-Wl,-z,max-page-size=16384"`
  (16 KB pages, required on Android 15+).

## 2. Export ONNX for the NPU

Re-export from the original checkpoint (PyTorch), not from an ONNX made for CPUs:
- `--dynamic-batch 0` (batch 1), **no int8 quantization in the ONNX** (`--enable-int8-quantization 0`: quantize in
  QNN instead), `--use-int32-inputs 1` (no int64 tensors), fixed chunk/length.
- Simplify (`onnxsim`) and check every input/output has a fixed shape (sherpa-onnx `scripts/pyannote/segmentation/show-onnx.py`).
- Run the ONNX on CPU (onnxruntime) against the original on a few real inputs: same output (or same text).

Examples: streaming Zipformer2 (icefall `zipformer/export-onnx-streaming.py`, see sherpa-onnx
`scripts/zipformer-transducer/x-asr/qnn/export-streaming.sh`); Whisper (sherpa-onnx `scripts/whisper/qnn/export_onnx.py`,
encoder and decoder as separate graphs).

## 3. Convert, build, compile (per graph, per chip)

For each ONNX graph `$m` (e.g. encoder, decoder, joiner):

```sh
# a) ONNX → QNN C++ graph + weights. fp16 activations: --act_bitwidth 16 --bias_bitwidth 32.
#    For 8-bit weights (big models): add quantization with a calibration --input_list of real inputs.
qnn-onnx-converter --input_network $m.onnx --output_path out/$m.cpp --act_bitwidth 16 --bias_bitwidth 32

# b) HTP config for the chip (sherpa-onnx scripts/qnn/generate_config.py: soc_id, dsp_arch, vtcm).
python3 generate_config.py --soc SM8750 --graph-name $m --output-dir out/config-$m --qnn-sdk-root $QNN_SDK_ROOT

# c) Model libraries: x86_64 (to compile the context on the host) and aarch64-android.
python3 $QNN_SDK_ROOT/bin/x86_64-linux-clang/qnn-model-lib-generator -c out/$m.cpp -b out/$m.bin -o out/model_libs

# d) Context binary for the chip: what the phone loads.
$QNN_SDK_ROOT/bin/x86_64-linux-clang/qnn-context-binary-generator \
  --backend $QNN_SDK_ROOT/lib/x86_64-linux-clang/libQnnHtp.so \
  --model out/model_libs/x86_64-linux-clang/lib$m.so \
  --output_dir out/binary --binary_file $m \
  --config_file out/config-$m/htp_backend_extensions.json
```

Keep the converter and generator logs: unsupported ops and quantization warnings show there.

## 4. Run on the phone

- Ship the QNN runtime libraries for Android aarch64 (`libQnnHtp.so`, `libQnnHtpV79Skel.so` and stub for the chip,
  `libQnnSystem.so`; sherpa-onnx publishes them as `qnn-libs-<version>.tar.bz2`). The Skel library runs on the DSP:
  sherpa-onnx's `OnlineRecognizer.prependAdspLibraryPath(dir)` (or `ADSP_LIBRARY_PATH`) must point at its folder.
- sherpa-onnx: `QnnConfig(backendLib = ".../libQnnHtp.so", contextBinary = ".../$m.bin", systemLib = ".../libQnnSystem.so")`
  on the model config (e.g. `OnlineTransducerModelConfig(..., qnnConfig = ...)`), `provider = "qnn"`.
- ONNX Runtime alternative: the QNN execution provider (`QNNExecutionProvider`, `backend_path=libQnnHtp.so`), which can
  also cache context binaries itself.
- Pick per device: if `ro.soc.model` matches a compiled chip and the libraries load, NPU; else CPU. Never assume.

## 5. Verify

1. **Same output:** run the same audio/input on CPU and NPU; for speech, word error rate on 10+ real clips. 16-bit
   activations are usually lossless; 8-bit weights must be checked.
2. **Speed:** real-time factor and first-output latency on the phone.
3. **Power:** CPU % of the process, and battery drop over a fixed 30–60 min task, CPU vs NPU, screen off, same
   brightness and network. Repeat; thermal state changes results.
4. Record results (chip, SDK version, model commit, quantization, numbers) next to the model.

## Pitfalls

- Dynamic shapes, int64 inputs, unsupported ops (some attention or control-flow ops): fix in the export (wrap the
  model, fix lengths), not in QNN.
- Context binaries are tied to the SDK version family: build with the same QAIRT version as the runtime libraries.
- A model that falls back to CPU for some ops silently loses most of the gain: check the logs for CPU partitions.
- Licenses: the model's (redistributing a compiled form is distributing the model) and the QNN SDK's.
- Alternative route without local Linux: Qualcomm AI Hub (cloud compile and profile on real devices; needs an
  account; check its terms for your app).
