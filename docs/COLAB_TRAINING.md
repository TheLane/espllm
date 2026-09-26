# ESP32 training on Google Colab

This is the shortest reproducible training recipe for the verified ESP32 profile.

## 1. Select a CUDA GPU

In Google Colab choose a GPU runtime. CPU runtime is not recommended for the full run.

## 2. Clone

    git clone https://github.com/TheLane/espllm.git
    cd espllm

For reproducing the exact upstream source base, the parent repository is:

    https://github.com/ahmedbarakat207/espllm

## 3. Install

    pip install -q tokenizers torchao

## 4. Verify CUDA

    import torch
    print(torch.__version__)
    print(torch.cuda.is_available())
    print(torch.cuda.get_device_name(0))

The CUDA check must return True.

## 5. Train the ESP32 target

    !python main.py --target=esp32 --train --fp-adam

Do not substitute esp32s3. The verified profile is the regular ESP32 target:

- 128 embedding
- 8 layers
- 4 attention heads
- 1 KV head
- 16 experts
- 192 MoE hidden
- block size 64
- vocabulary 2048

## 6. Expected artifacts

    model/model_esp32.pt
    model/model_esp32.pt.best
    model/model_esp32.pt.quantized

The firmware exporter consumes the quantized artifact.

## 7. Export locally

Copy model/model_esp32.pt.quantized to the local repository and run:

    python convert_model_to_c.py esp32

Verify the generated header reports:

    n_embd=128
    n_layer=8
    n_head=4
    head_dim=32
    mlp_hidden=192
    block_size=64

## 8. Build and flash

    pio run -e esp32dev
    pio run -e esp32dev -t upload --upload-port COM8

## 9. Verify runtime

The tested ESP32 board required:

    INFER_CTX = 48
    ARENA_SIZE = 111 * 1024

because its largest contiguous heap block was 114,676 B even though total free heap was about 351 KB.

A successful boot should report the model profile, allocate the arena, load the tokenizer/model and accept serial prompts.

## 10. Verified training result

The documented run stopped early around iteration 6900. The best validation loss was 0.8301 around iteration 5900. The quantized checkpoint was approximately 3.76 MB.

This document intentionally separates the training profile from the runtime SRAM workaround. The model remains trained with block_size=64; the physical 4 MB ESP32 uses a smaller runtime context of 48.
