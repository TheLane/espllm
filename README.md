# ESP-LLM on ESP32 — trained, quantized and verified

> **A practical ESP32 LLM/MoE project: training on Google Colab GPU, ternary quantization, C++ export, and real inference on a 4 MB ESP32 Dev Module.**

**Русский:** Этот репозиторий содержит воспроизводимый результат отдельного аудита и запуска ESP-LLM: мы восстановили целевой ESP32-профиль, обучили его в Google Colab на GPU, экспортировали квантованные веса, собрали firmware и подтвердили генерацию токенов на физическом ESP32.

**English:** This repository documents a reproducible ESP-LLM workflow: recovering the intended ESP32 profile, training it on a Google Colab GPU, exporting quantized weights, building firmware, and verifying real token generation on physical ESP32 hardware.

## Attribution / Авторство

This project is a **fork and documented continuation of the original work by Ahmed Barakat**.

- Original repository: https://github.com/ahmedbarakat207/espllm
- Original author: [Ahmed Barakat / @ahmedbarakat207](https://github.com/ahmedbarakat207)
- This repository: https://github.com/TheLane/espllm
- The original code, architecture and design remain credited to the original author.
- Our additions focus on the ESP32 training/recovery workflow, reproducible documentation, generated ESP32 model artifacts, and hardware verification.

Please consult the original repository for the upstream project's current license and attribution terms.

## Verified ESP32 result

| Item | Verified value |
|---|---:|
| Target | ESP32 Dev Module / ESP32-D0WD-V3 |
| CPU | 240 MHz Xtensa LX6 |
| Flash | 4 MB |
| Vocabulary | 2048 BPE tokens |
| Embedding | 128 |
| Transformer layers | 8 |
| Attention heads | 4 |
| KV heads | 1 (MQA) |
| MoE experts | 16 |
| MoE hidden size | 192 |
| Trained block size | 64 |
| Runtime context on tested board | 48 |
| Quantized checkpoint | ~3.76 MB |
| Firmware | ~2.95 MB |
| Runtime arena | 111 KiB configured |
| Measured arena use | 112,320 / 113,664 B |
| Free heap after arena | 237,748 B |
| First verified inference | 2.152 s |
| Hardware test | PASS |

The tested board could not provide a single contiguous 160 KiB heap block even though total free heap was ~351 KiB. The runtime was therefore adapted to a 48-token inference context and a 111 KiB arena. The model itself was trained/exported with a 64-token block size; runtime context is intentionally smaller for SRAM constraints.

## What we actually did

1. Audited the upstream source and Git history.
2. Identified the intended ESP32 architecture: **128 / 8 / 4 / 1 / 16 / 192**.
3. Found that the repository did not contain the expected trained `model_esp32.pt` artifact.
4. Recreated the model from scratch on a **Google Colab CUDA GPU** using the project's own training code.
5. Training stopped early at about iteration 6900; the best checkpoint was around iteration 5900.
6. Best validation loss observed in that run: **0.8301**.
7. Exported the quantized checkpoint (`model_esp32.pt.quantized`).
8. Regenerated `src/model_weights.hpp` and verified every architecture dimension.
9. Installed PlatformIO locally and built the standalone `esp32dev` firmware.
10. Flashed the firmware to a real ESP32 through **COM8**.
11. Diagnosed the ESP32 heap fragmentation/contiguous-allocation limit.
12. Adjusted runtime context from 64 to 48 and arena configuration to fit the actual board.
13. Confirmed model loading, arena allocation, and real text generation over serial.

For the full bilingual procedure, see:

- [Русская документация: полный процесс](docs/PROCESS_RU.md)
- [English documentation: complete process](docs/PROCESS_EN.md)
- [Google Colab training recipe](docs/COLAB_TRAINING.md)
- [Verified ESP32 firmware artifact](artifacts/firmware-esp32dev.bin)

## Quick start

### 1. Clone

```bash
git clone https://github.com/TheLane/espllm.git
cd espllm
```

### 2. Install training dependencies

```bash
python -m pip install tokenizers torchao
```

For training the ESP32 target, use a CUDA-capable Colab/Kaggle runtime. A normal CPU machine is not a practical replacement for the full training run.

### 3. Train the ESP32 model

```bash
python main.py --target=esp32 --train --fp-adam
```

The run uses the ESP32 profile already defined in `main.py` and produces:

```
model/model_esp32.pt
model/model_esp32.pt.best
model/model_esp32.pt.quantized
```

### 4. Export C++ weights

```bash
python convert_model_to_c.py esp32
```

This regenerates:

```
src/model_weights.hpp
```

Verify that it reports:

```
n_embd=128
n_layer=8
n_head=4
head_dim=32
mlp_hidden=192
block_size=64
```

### 5. Build

The normal ESP32 environment is named `esp32dev`:

```bash
pio run -e esp32dev
```

### 6. Flash

```bash
pio run -e esp32dev -t upload --upload-port COM8
```

Replace `COM8` with your actual port.

### 7. Monitor

```bash
pio device monitor -p COM8 -b 115200
```

## Repository map

```
.
├── docs/
│   ├── PROCESS_RU.md
│   └── PROCESS_EN.md
├── model/
├── src/
├── main.py
├── convert_model_to_c.py
├── build_dataset.py
├── flash.py
├── run.py
├── platformio.ini
├── dataset.txt
├── bpe-vocab.json
└── bpe-merges.txt
```

Large generated model artifacts may be distributed separately from Git history if repository limits require it. The source code and exact commands remain documented here.

## Search keywords

ESP32 LLM, ESP32 language model, tiny LLM, embedded LLM, microcontroller LLM, MoE, Mixture of Experts, BitNet, ternary quantization, 1.58-bit, INT8 activations, MQA, RoPE, ESP32 AI, edge AI, TinyML, PlatformIO, Arduino ESP32, Google Colab training, on-device inference, local LLM, small language model.

## Important limitations

- This is a small, specialized language model, not a general-purpose modern LLM.
- Quality is strongly tied to the training dataset.
- The tested 4 MB ESP32 board has strict contiguous-SRAM constraints.
- Runtime context on the verified board is 48 tokens even though the model was trained with a 64-token block.
- The project should be treated as an embedded-AI experiment and engineering reference.

## License and upstream status

This repository is a public fork of the upstream project. We intentionally preserve attribution to the original author. No upstream license is invented here; consult the original repository for the authoritative licensing terms.

If you improve the ESP32 profile, training recipe, quantization path, or hardware runtime, please open an issue or pull request.

## Acknowledgements

Special thanks to the original author, Ahmed Barakat, for the ESP-LLM architecture and implementation that made this experiment possible.

Original project: https://github.com/ahmedbarakat207/espllm
