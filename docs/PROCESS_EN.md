# Complete ESP-LLM -> ESP32 process

The original architecture and main inference engine belong to Ahmed Barakat and the upstream project: https://github.com/ahmedbarakat207/espllm

## 1. Recovered profile

Target profile:

- vocab: 2048
- embedding: 128
- layers: 8
- attention heads: 4
- KV heads: 1
- head dim: 32
- experts: 16
- MoE hidden: 192
- trained block: 64

The old model_weights.hpp did not match this profile, so the header was regenerated.

## 2. Why retraining was required

The upstream working copy did not contain the expected model/model_esp32.pt. Other checkpoint files belonged to different architectures. The project's own training pipeline was therefore used.

## 3. Training environment

A CPU smoke test proved the training code worked, but a full run was too slow. The AMD RX 580 was investigated through Vulkan and DirectML: Vulkan did not provide a suitable PyTorch training backend, while DirectML failed in MoE scatter backward.

Full training was therefore performed on a CUDA GPU in Google Colab.

Command:

    python main.py --target=esp32 --train --fp-adam

Important points:

- iter 0: val 7.7239
- iter 1000: val 1.2145
- iter 3000: val 0.9890
- iter 5000: val 0.9095
- iter 5400: val 0.9009
- iter 5900: val 0.8301
- iter 6900: early stopping

The best checkpoint was restored before quantization. Quantized checkpoint size: about 3.76 MB.

## 4. Export

    python convert_model_to_c.py esp32

The exporter confirmed:

    n_embd=128
    n_layer=8
    n_head=4
    head_dim=32
    mlp_hidden=192
    block_size=64

We also verified vocab=2048, group_size=64, experts=16 and kv_heads=1.

## 5. Build and flash

    pio run -e esp32dev
    pio run -e esp32dev -t upload --upload-port COM8

Build result:

- Flash: 2,947,161 / 4,063,232 bytes = 72.5%
- firmware.bin: 2,947,520 bytes
- static RAM: 21,668 / 327,680 bytes = 6.6%

The physical board appeared as a Silicon Labs CP210x USB to UART Bridge on COM8. Flash write and SHA verification completed successfully.

## 6. Real RAM constraint

The original runtime requested 160 KB. Total free heap was 351,396 B, but the largest contiguous block was only 114,676 B. A static 160 KB arena also failed because it overflowed the DRAM linker region.

ESP.getMaxAllocHeap() diagnostics were added.

Verified configuration:

    INFER_CTX = 48
    ARENA_SIZE = 111 * 1024

The model was trained with block_size=64, so runtime context 48 remains checkpoint-compatible.

Measured:

- arena: 112,320 / 113,664 B
- free heap after arena: 237,748 B

## 7. End-to-end inference

After boot, firmware reported vocab 2048, embedding 128, heads 4, layers 8, context 48 and MoE hidden 192.

Input:

    hi

Output:

    Hello! How can see you today?

Time: 2152 ms.

No panic, watchdog reset, crash or repeated boot was observed during this inference.

The complete chain is proven:

checkpoint -> quantization -> C++ weights -> firmware -> flash -> arena -> model load -> inference -> generated tokens.

## 8. Reproduction

    python -m pip install tokenizers torchao
    python main.py --target=esp32 --train --fp-adam
    python convert_model_to_c.py esp32
    pio run -e esp32dev
    pio run -e esp32dev -t upload --upload-port COM8
    pio device monitor -p COM8 -b 115200

Full training should be performed in a CUDA-enabled Colab/Kaggle runtime.

## 9. Attribution

Original project: Ahmed Barakat — @ahmedbarakat207

https://github.com/ahmedbarakat207/espllm

This work does not claim authorship of the original ESP-LLM architecture. We preserve upstream attribution and document the recovery, training, export and physical ESP32 verification work.

## 10. Limitations

This is a small specialized embedded language model. Results are constrained by model size, training corpus, BPE vocabulary, SRAM/flash limits and context.
