# Aria

[😊 Hugging Face](https://huggingface.co/rhymes-ai/Aria) | 
[📄 Paper](https://arxiv.org/pdf/2410.05993) | 
[📚 Blog](https://rhymes.ai/blog-details/aria-first-open-multimodal-native-moe-model) | 
[🌐 WebDemo](https://rhymes.ai/) 


## Introduction
Aria is a multimodal native MoE model. It features:
- State-of-the-art performance on various multimodal and language tasks, superior in video and document understanding;
- Long multimodal context window of 64K tokens;
- 3.9B activated parameters per token, enabling fast inference speed and low fine-tuning cost.
  

## Philosophy

Aria aims to be an open multimodal Mixture-of-Experts model that activates only 3.9B parameters per token, enabling fast inference. Our goal is to provide broad modality coverage while maintaining a fully open-source, modular design so developers can easily extend and build upon Aria.

## News
- 2024.10.10: We release Aria!

## Quick Start

### Installation

```bash
pip install -e .
# or install with dev dependencies if you want to contribute to the project
pip install -e .[dev] 

pip install flash-attn --no-build-isolation
```

### Inference

Aria has 25.3B total parameters, it can be loaded in one A100 (80GB) GPU with bfloat16 precision.

Here is a code snippet to show you how to use Aria with Hugging Face Transformers.

```python
import requests
import torch
from PIL import Image
from transformers import AutoModelForCausalLM, AutoProcessor

model_id_or_path = "rhymes-ai/Aria"

model = AutoModelForCausalLM.from_pretrained(model_id_or_path, device_map="auto", torch_dtype=torch.bfloat16, trust_remote_code=True)

processor = AutoProcessor.from_pretrained(model_id_or_path, trust_remote_code=True)

image_path = "https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/diffusers/cat.png"

image = Image.open(requests.get(image_path, stream=True).raw)

messages = [
    {
        "role": "user",
        "content": [
            {"text": None, "type": "image"},
            {"text": "what is the image?", "type": "text"},
        ],
    }
]

text = processor.apply_chat_template(messages, add_generation_prompt=True)
inputs = processor(text=text, images=image, return_tensors="pt")
inputs["pixel_values"] = inputs["pixel_values"].to(model.dtype)
inputs = {k: v.to(model.device) for k, v in inputs.items()}

with torch.inference_mode(), torch.cuda.amp.autocast(dtype=torch.bfloat16):
    output = model.generate(
        **inputs,
        max_new_tokens=500,
        stop_strings=["<|im_end|>"],
        tokenizer=processor.tokenizer,
        do_sample=True,
        temperature=0.9,
    )
    output_ids = output[0][inputs["input_ids"].shape[1]:]
    result = processor.decode(output_ids, skip_special_tokens=True)

print(result)
```

We offer additional inference methods, such as utilizing [vLLM](https://github.com/vllm-project/vllm) for enhanced performance. For comprehensive details, please refer to [docs/inference.md](docs/inference.md).

### Cookbook
Checkout these [inference examples](https://github.com/rhymes-ai/Aria/tree/main/inference/notebooks) that demonstrate how to use Aria on various applications such as chart understanding, PDF reading, video understanding, etc, available with both Hugging Face Transformers and [vLLM](https://github.com/vllm-project/vllm) backends.


## System Workflow

Below is a high level view of how data moves through Aria's training and inference pipeline:

```
Dataset files (train.jsonl / test.jsonl) ──▶ `load_local_dataset` ──▶ `mix_datasets`
      │
      └── images or video frames
                │
                ▼
           `collate_fn` (calls `apply_chat_template_and_tokenize` and `AriaVisionProcessor`)
                │
                ▼
             `SFTTrainer` in `aria/train.py`
                │
                ▼
        fine‑tuned model for use with `aria/inference.py` or vLLM
```

The dataset format is described in [custom_dataset.md](docs/custom_dataset.md). `aria/data.py` loads each dataset directory containing `train.jsonl`/`test.jsonl` and image or video files. `mix_datasets` can combine multiple datasets according to the sampling weights defined in the configuration file. During training, `aria/train.py` invokes `mix_datasets`, processes the samples with `collate_fn`, and passes them to `SFTTrainer`. After training, the resulting model can be used for inference with `aria/inference.py` or via the vLLM backend as shown in [docs/inference.md](docs/inference.md).
=======
## Architecture

Aria consists of three main building blocks:

- **Vision encoder** ([aria/model/vision_encoder.py](aria/model/vision_encoder.py))
- **Multi-modal projector** ([aria/model/projector.py](aria/model/projector.py))
- **MoE language model** ([aria/model/moe_lm.py](aria/model/moe_lm.py))

During inference an image is first processed by the vision encoder, producing patch-level embeddings and an attention mask. The projector converts these embeddings into a sequence of visual tokens, which are then concatenated with the textual prompt and fed into the MoE language model to generate the final response.

```mermaid
graph LR
    A[Image] --> B(vision_encoder.py)
    B --> C(projector.py)
    C --> D(moe_lm.py)
```

## Fine-tuning

We offer both LoRA fine-tuning and full parameter tuning, using various dataset types:
- Single-image datasets
- Multi-image datasets
- Video datasets

For a quick try, visit the [examples](./examples) folder and choose one of the fine-tuning examples.

### Prepare dataset
Please refer to [custom_dataset.md](docs/custom_dataset.md) for how to prepare your dataset.

### Fine-tune with LoRA

After preparing your dataset, follow these steps to fine-tune Aria using LoRA:

1. Open the configuration file `recipes/config_lora.yaml`. Locate the `dataset_mixer` section and update it with your dataset paths:

```yaml
dataset_mixer:
  "path/to/dataset1": 1
  "path/to/dataset2": 0.5
  "path/to/dataset3": 2
```

> **Note on dataset mixing:** Aria supports combining multiple datasets with different sampling rates. In the example above:
> - `dataset1` will be used entirely (weight 1)
> - `dataset2` will use 50% of its data (weight 0.5)
> - `dataset3` will be used twice (weight 2)

2. Start the fine-tuning process by running the following command on one A100 (80GB) or H100 (80GB) GPU:

```bash
python aria/train.py --config recipes/config_lora.yaml
```

3. For multi-GPU training, use the [`accelerate`](https://huggingface.co/docs/accelerate/index) library:

```bash
accelerate launch --config_file recipes/accelerate_configs/zero2.yaml aria/train.py --config recipes/config_lora.yaml --num_processes [number_of_gpus]
```

   - Choose from pre-configured accelerate settings in `recipes/accelerate_configs/`
   - Adjust the `--num_processes` argument to match your available GPUs
   - For custom configurations, refer to the [accelerate documentation](https://huggingface.co/docs/accelerate/usage_guides/deepspeed)
  
4. Inference with the fine-tuned model:

   See [inference with LoRA support](docs/inference.md#2-inference-with-lora-support) for how to inference with the fine-tuned model.

### Full parameter fine-tuning

Everything is the same as the LoRA fine-tuning process, except for the configuration file `recipes/config_full.yaml`.

Full parameter tuning consumes more GPU memory, thus multiple GPUs are required. The following command has been tested on 8 A100 (80GB) GPUs.

```bash
accelerate launch --config_file recipes/accelerate_configs/zero2.yaml aria/train.py --config recipes/config_full.yaml
```

If you encounter out-of-memory errors, try reducing the `per_device_train_batch_size` in the config file. Adjust the `gradient_accumulation_steps` accordingly to maintain the effective training batch size.

```yaml
per_device_train_batch_size: 8
gradient_accumulation_steps: 2
```

Memory consumption varies across datasets. Generally, more memory is required for multi-image and video datasets. Adjust the `deepspeed_config` parameters to optimize memory consumption, such as using `zero_stage` 3 and offloading parameters and optimizer to the CPU.

```yaml
deepspeed_config:
  gradient_accumulation_steps: auto
  gradient_clipping: auto
  offload_optimizer_device: cpu
  offload_param_device: cpu
  zero3_init_flag: true
  zero_stage: 3
```

### Configuration

The YAML files in `recipes` control training behavior. Key options include:

- `freeze_vit` – freeze the vision encoder weights ([`AriaModelConfig.freeze_vit`](aria/config.py#L37-L40)).
- `freeze_llm` – freeze all language model layers or specify `freeze_llm_layers` for partial freezing ([`AriaModelConfig.freeze_llm`](aria/config.py#L45-L48)).
- `max_image_size` – size of the image fed to the vision encoder, either `490` or `980` pixels ([`AriaModelConfig.max_image_size`](aria/config.py#L61-L67)).
- `moe_z_loss_coeff` – coefficient for the router z-loss in MoE layers ([`AriaModelConfig.moe_z_loss_coeff`](aria/config.py#L53-L55)).
- `moe_aux_loss_coeff` – coefficient for the auxiliary routing loss ([`AriaModelConfig.moe_aux_loss_coeff`](aria/config.py#L57-L59)).

Modify these values in `recipes/config_lora.yaml` or `recipes/config_full.yaml` to match your dataset size and hardware. The LoRA config is a good starting point for single-GPU setups, while the full config exposes all parameters for larger runs.

#### Inference with Your Trained Model

First, you need to extract the FP32 consolidated weights from ZeRO 1, 2, or 3 DeepSpeed checkpoints:
```bash
cd /path/to/your/output/dir
python zero_to_fp32.py . pytorch_model.bin
```

See [inference.md](docs/inference.md) for instructions on how to perform inference with the fine-tuned model.

## Citation
If you find our work helpful, please consider citing.
```
@article{aria,
  title={Aria: An Open Multimodal Native Mixture-of-Experts Model}, 
  author={Dongxu Li and Yudong Liu and Haoning Wu and Yue Wang and Zhiqi Shen and Bowen Qu and Xinyao Niu and Guoyin Wang and Bei Chen and Junnan Li},
  year={2024},
  journal={arXiv preprint arXiv:2410.05993},
}
```


