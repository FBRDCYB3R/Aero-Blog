---
title: "Fine Tuning an LLM with personal Notes to use as a Second Brain"
date: 2024-06-13  
category: Software
tags: [Software]
description: "LLM to help me study."
read_time: 10
---

# Project Write-Up: Fine-Tuning Llama 2.1 (7B) on a Budget

## Project Overview

Over the course of three weeks, I successfully fine-tuned a 7-billion parameter Large Language Model (Llama 2.1 7B) using entirely free compute resources via Google Colab.

Fine-tuning a model of this size usually requires high-end, expensive GPUs with massive amounts of VRAM. My goal was to prove that with the right optimization techniques—specifically QLoRA (Quantized Low-Rank Adaptation)—and a lot of patience, it could be done on a standard Colab T4 GPU.

Because free Colab instances disconnect after a few hours of use, I couldn't run the training in one go. Instead, I engineered a "sectional" training pipeline, heavily relying on automated checkpointing to save my progress and resume training session by session over the 21-day period.

---

## The Challenge

**Hardware Constraints:** A standard 7B model takes about 14GB of VRAM just to load in 16-bit precision, leaving zero room for the actual training process on a 16GB T4 GPU.  
**Time Constraints:** Google Colab terminates sessions randomly or after usage limits are hit (usually 4–12 hours). A full fine-tuning run on a sizable dataset takes much longer than a single session allows.

---

## The Solution & Methodology

To bypass the memory limits, I used 4-bit quantization via the `bitsandbytes` library to shrink the model's memory footprint to around 6GB. I then applied LoRA, which froze the original model weights and only trained a tiny, injected set of "adapter" parameters (~1% of the model).

To solve the time constraints, I configured the Hugging Face `SFTTrainer` to save strict, regular checkpoints to my mounted Google Drive. Every time Colab kicked me out, I simply reconnected, pointed the trainer to the latest checkpoint, and resumed exactly where I left off.

---

## Technical Implementation

### 1. Environment & Authentication

Because Llama is a gated model, I first had to install the dependencies, mount my Google Drive to store the checkpoints safely outside of the ephemeral Colab environment, and log in to Hugging Face.
```python
!pip install -q -U transformers peft trl bitsandbytes datasets accelerate

from google.colab import drive
drive.mount('/content/drive')

from huggingface_hub import login
login(token="YOUR_HF_TOKEN")
```

### 2. Loading the Model in 4-bit Precision

I loaded the base 7B model in 4-bit precision to ensure it wouldn't cause an Out of Memory (OOM) error.
```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model

model_name = "meta-llama/Llama-2-7b-chat-hf"

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_use_double_quant=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16
)

print("Loading 7B model... (This takes a minute)")
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    quantization_config=bnb_config,
    device_map="auto"
)

tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token
```

### 3. Applying LoRA

Instead of training 7 billion parameters, I targeted the attention blocks, which drastically reduced the compute load.
```python
peft_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "v_proj", "k_proj", "o_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM"
)

model = get_peft_model(model, peft_config)
model.print_trainable_parameters()
```

### 4. The "Sectional" Training Strategy

This was the most critical part of the 3-week process. I mapped the output directory directly to my Google Drive and set the save steps to a low number.
```python
from trl import SFTTrainer
from transformers import TrainingArguments
from datasets import load_dataset
import os

dataset = load_dataset("timdettmers/openassistant-guanaco", split="train")

output_dir = "/content/drive/MyDrive/Llama-2-7B-FineTune/checkpoints"

training_args = TrainingArguments(
    output_dir=output_dir,
    per_device_train_batch_size=2,
    gradient_accumulation_steps=4,
    optim="paged_adamw_32bit",
    logging_steps=10,
    learning_rate=2e-4,
    fp16=True,
    max_steps=2000,
    save_steps=50,
    save_total_limit=3,
)

trainer = SFTTrainer(
    model=model,
    train_dataset=dataset,
    peft_config=peft_config,
    dataset_text_field="text",
    max_seq_length=512,
    tokenizer=tokenizer,
    args=training_args,
)
```

### 5. Resuming from Checkpoints

Whenever my Colab session timed out, I would restart the notebook, run the setup cells, and execute the following logic to resume:
```python
checkpoints = [d for d in os.listdir(output_dir) if d.startswith("checkpoint-")]

if len(checkpoints) > 0:
    print("Found existing checkpoints. Resuming training...")
    trainer.train(resume_from_checkpoint=True)
else:
    print("No checkpoints found. Starting fresh...")
    trainer.train()

trainer.model.save_pretrained("/content/drive/MyDrive/Llama-2-7B-FineTune/final_adapter")
```

---

## Results and Reflection

By breaking the training down into manageable chunks and leveraging QLoRA, I successfully fine-tuned a 7B parameter model without paying a dime for cloud compute.

The biggest bottleneck wasn't the actual math—it was the logistics of managing Colab quotas. If I missed a day of restarting the session, the timeline extended. However, the final adapter weights dramatically improved the model's adherence to my specific conversational format, proving that consumer-grade hardware (or free cloud tiers) can absolutely handle enterprise-grade models if you use the right engineering workarounds.
