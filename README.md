## 🍮 🦙 Flan-Alpaca: Instruction Tuning from Humans and Machines

This repository contains code for extending the [Stanford Alpaca](https://github.com/tatsu-lab/stanford_alpaca)
synthetic instruction tuning to existing instruction-tuned models such as [Flan-T5](https://arxiv.org/abs/2210.11416).
The pretrained models and demos are available on HuggingFace 🤗 :
[Base](https://huggingface.co/declare-lab/flan-alpaca-base) (220M),
[Large](https://huggingface.co/declare-lab/flan-alpaca-large) (770M),
[XL](https://huggingface.co/declare-lab/flan-alpaca-xl) (3B),
XXL (11B, Coming soon)

### Why?

[Alpaca](https://crfm.stanford.edu/2023/03/13/alpaca.html) represents an exciting new direction
to approximate the performance of large language models (LLMs) like ChatGPT cheaply and easily.
Concretely, they leverage an LLM such as GPT-3 to generate instructions as synthetic training data.
The synthetic data which covers more than 50k tasks can then be used to finetune a smaller model.
However, the original implementation is less accessible due to licensing constraints of the
underlying [LLaMA](https://ai.facebook.com/blog/large-language-model-llama-meta-ai/) model.
Furthermore, users have noted [potential noise](https://github.com/tloen/alpaca-lora/issues/65) in the synthetic
dataset. Hence, it may be better to explore a fully accessible model that is already trained on high-quality (but
less diverse) instructions such as [Flan-T5](https://arxiv.org/abs/2210.11416).

### Usage

```
from transformers import pipeline

prompt = "Write an email about an alpaca that likes flan"
model = pipeline(model="declare-lab/flan-alpaca-xl")
model(prompt, max_length=128, do_sample=True)

# Dear AlpacaFriend,
# My name is Alpaca and I'm 10 years old.
# I'm excited to announce that I'm a big fan of flan!
# We like to eat it as a snack and I believe that it can help with our overall growth.
# I'd love to hear your feedback on this idea. 
# Have a great day! 
# Best, AL Paca
```

### Setup

Install dependencies and download data. We used the cleaned data
from [Alpaca-LoRA](https://github.com/tloen/alpaca-lora.git) for training.

```
conda create -n paca python=3.8 -y
conda activate paca
pip install -r requirements.txt
mkdir -p data
wget https://raw.githubusercontent.com/tatsu-lab/stanford_alpaca/main/alpaca_data.json -O data/alpaca.json
wget https://raw.githubusercontent.com/tloen/alpaca-lora/main/alpaca_data_cleaned.json -O data/alpaca_clean.json
```

Preprocess training dataset:

```
python data_loading.py preprocess_alpaca \
--path_in data/alpaca_clean.json \
--path_out data/train.json
```

### Training

The following command will finetune the Flan-T5-XL (8hrs on a single A6000 GPU).

```
python training.py --output_dir outputs/model/xl \
--use_compile \
--train_epochs 3 \
--data_path data/train.json \
--model_name_or_path "google/flan-t5-xl" \
--train_batch_size 1 \
--gradient_accumulation_steps 64
```

If the model does not fit into memory, and you have multiple GPUs, you can
try [fully-sharded data parallel](https://engineering.fb.com/2021/07/15/open-source/fsdp/) by replacing `--use_compile`
with `--use_fsdp`.

### Inference

```
python inference.py test_model \
--path "outputs/model/xl/epoch=2-step=2436.ckpt" \
--prompt "Write an email about an alpaca that likes flan"
```

### Exporting to HuggingFace Hub

Replace `declare-lab/flan-alpaca-xl` with your desired HuggingFace repo.

```
huggingface-cli login

python inference.py export_to_hub \
--path "outputs/model/xl/epoch=2-step=2436.ckpt" \
--repo declare-lab/flan-alpaca-xl
```
