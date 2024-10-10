These benchmarks are from an Orange Pi 5 Pro running Ubuntu (for Rockchip) off of an m.2 NVMe SSD. mlc-llm has been compiled to run off OpenCL. mlc-llm tvm-unity and all models used were all freshly downloaded on 10/10/24

| Model                                       | Prefill tok/s | Decode tok/s |
| ------------------------------------------- | ------------- | ------------ |
| mlc-ai/SmolLM-1.7B-Instruct-q4f16_1-MLC     | 16.6          | 10.1         |
| mlc-ai/Llama-3.2-3B-Instruct-q4f16_1-MLC    | 8.8           | 6.1          |
| mlc-ai/Qwen2.5-3B-Instruct-q4f16_1-MLC      | 8.5           | 5.1          |
| mlc-ai/Phi-3.5-mini-instruct-q4f16_1-MLC    | 5.5           | 4.8          |
| mlc-ai/SmolLM-360M-Instruct-q4f16_1-MLC     | 44.4          | 25.7         |
| mlc-ai/SmolLM-1.7B-Instruct-q4f16_1-MLC     | 18            | 9.8          |
| mlc-ai/TinyLlama-1.1B-Chat-v1.0-q4f16_1-MLC | 25.9          | 13.9         |
