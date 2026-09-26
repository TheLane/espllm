# Полный процесс ESP-LLM -> ESP32

Исходная архитектура и основной inference engine принадлежат Ahmed Barakat и upstream: https://github.com/ahmedbarakat207/espllm

## 1. Восстановленный профиль

Целевой профиль обычного ESP32:

- vocab: 2048
- embedding: 128
- layers: 8
- attention heads: 4
- KV heads: 1
- head dim: 32
- experts: 16
- MoE hidden: 192
- trained block: 64

Старый model_weights.hpp этому профилю не соответствовал, поэтому header был сгенерирован заново.

## 2. Почему пришлось обучать заново

В рабочей копии upstream отсутствовал ожидаемый model/model_esp32.pt. Другие checkpoint-файлы относились к другим архитектурам. Поэтому использован штатный training pipeline.

## 3. Где обучали

CPU smoke test подтвердил работу training code, но полный run был слишком медленным. AMD RX 580 была проверена через Vulkan и DirectML: Vulkan не дал подходящего PyTorch training backend, а DirectML ломался на MoE scatter backward.

Поэтому полный training выполнен в Google Colab на CUDA GPU.

Команда:

    python main.py --target=esp32 --train --fp-adam

Ключевые точки:

- iter 0: val 7.7239
- iter 1000: val 1.2145
- iter 3000: val 0.9890
- iter 5000: val 0.9095
- iter 5400: val 0.9009
- iter 5900: val 0.8301
- iter 6900: early stopping

Лучший checkpoint был восстановлен перед quantization. Quantized checkpoint: около 3.76 MB.

## 4. Экспорт

    python convert_model_to_c.py esp32

Exporter подтвердил:

    n_embd=128
    n_layer=8
    n_head=4
    head_dim=32
    mlp_hidden=192
    block_size=64

Также проверены vocab=2048, group_size=64, experts=16 и kv_heads=1.

## 5. Сборка и прошивка

    pio run -e esp32dev
    pio run -e esp32dev -t upload --upload-port COM8

Результат сборки:

- Flash: 2,947,161 / 4,063,232 bytes = 72.5%
- firmware.bin: 2,947,520 bytes
- static RAM: 21,668 / 327,680 bytes = 6.6%

Физическая плата определилась как Silicon Labs CP210x USB to UART Bridge, COM8. Flash write и SHA verification прошли успешно.

## 6. Реальное ограничение RAM

Исходный runtime требовал 160 KB. Общий free heap был 351,396 B, но largest contiguous block только 114,676 B. Статическая arena 160 KB также не помещалась в DRAM linker region.

Добавлена диагностика ESP.getMaxAllocHeap().

Проверенная конфигурация:

    INFER_CTX = 48
    ARENA_SIZE = 111 * 1024

Модель обучена с block_size=64, поэтому runtime context 48 остаётся совместимым с checkpoint.

Измерено:

- arena: 112,320 / 113,664 B
- free heap after arena: 237,748 B

## 7. End-to-end inference

После boot firmware подтвердила vocab 2048, embedding 128, heads 4, layers 8, context 48 и MoE hidden 192.

Input:

    hi

Output:

    Hello! How can see you today?

Время: 2152 ms.

Panic, watchdog reset, crash или повторный boot во время этого inference не наблюдались.

Таким образом доказана цепочка:

checkpoint -> quantization -> C++ weights -> firmware -> flash -> arena -> model load -> inference -> generated tokens.

## 8. Воспроизведение

    python -m pip install tokenizers torchao
    python main.py --target=esp32 --train --fp-adam
    python convert_model_to_c.py esp32
    pio run -e esp32dev
    pio run -e esp32dev -t upload --upload-port COM8
    pio device monitor -p COM8 -b 115200

Полное обучение следует выполнять в CUDA-enabled Colab/Kaggle runtime.

## 9. Авторство

Original project: Ahmed Barakat — @ahmedbarakat207

https://github.com/ahmedbarakat207/espllm

Эта работа не претендует на авторство исходной ESP-LLM архитектуры. Мы сохраняем upstream attribution и документируем восстановление, обучение, экспорт и физическую проверку ESP32-профиля.

## 10. Ограничения

Это небольшой специализированный embedded language model. Результат ограничен размером модели, обучающим корпусом, BPE vocabulary, SRAM/flash и контекстом.
