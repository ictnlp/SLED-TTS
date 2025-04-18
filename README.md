# SLED-TTS: Efficient Speech Language Modeling via Energy Distance in Continuous Space
[![HuggingFace](https://img.shields.io/badge/HuggingFace-FEC200?style=flat&logo=Hugging%20Face)](https://huggingface.co/collections/ICTNLP/sled-tts-680253e19c889010a1a376ac)
[![WeChat AI](https://img.shields.io/badge/WeChat%20AI-4CAF50?style=flat&logo=wechat)](https://www.wechat.com)
[![ICT/CAS](https://img.shields.io/badge/ICT%2FCAS-0066cc?style=flat&logo=school)](https://ict.cas.cn)

## This repo is under construction.


## Key features
- **Autoregressive Continuous Modeling**: SLED models speech in a continuous latent space using a speacial type of maximum mean discrepancy as the objective.
- **Streaming Synthesis**: SLED supports streaming synthesis, enabling speech generation to start as soon as the text stream begins.
- **Voice Cloning**: Capable of generating speech based on a 3-second prefix or reference utterance as prompt.



## Demo
You can check SLED in action by exploring the [demo page](https://sled-demo.github.io/).
<div style="display: flex;">
   <img src="https://github.com/user-attachments/assets/0f6ee8a0-4258-48a2-a670-5556672dbc18" width="200" style="margin-right: 20px;"/>
   <img src="https://github.com/user-attachments/assets/f48848b0-58d9-403a-86d1-80683565a4d7" width="500"/>
</div>

## Available Models on Hugging Face

We have made SLED available on [Hugging Face](https://huggingface.co/collections/ICTNLP/sled-tts-680253e19c889010a1a376ac), currently offering two distinct English models for different use cases:

1. **[SLED-TTS-Libriheavy](https://huggingface.co/ICTNLP/SLED-TTS-Libriheavy)**: This model is trained on the Libriheavy dataset and provides high-quality text-to-speech synthesis.
  
2. **[SLED-TTS-Streaming-Libriheavy](https://huggingface.co/ICTNLP/SLED-TTS-Streaming-Libriheavy)**: This variant supports **streaming decoding**, which generates a 0.6-second speech chunk for every 5 text tokens received. It’s ideal for applications requiring low-latency audio generation.


The Mandarin models are on the way! Alternatively, you can train your own SLED-TTS models by following the guidelines below.

## Usage
**We provide the training and inference code for SLED-TTS.**

### Installation
``` sh
git clone https://github.com/ictnlp/SLED-TTS.git
cd SLED-TTS
pip install -e ./
```

We currently utilize the sum of the first 8 embedding vectors from [Encodec_24khz](https://huggingface.co/facebook/encodec_24khz) as the continuous latent vector. To proceed, ensure that [Encodec_24khz](https://huggingface.co/facebook/encodec_24khz) is downloaded and cached in your HuggingFace dir.

### Inference
``` sh
CHECKPOINT=/path/to/checkpoint
CFG=2.0

# Offline Inference
python scripts/run_offline.py \
    --model_name_or_path ${CHECKPOINT} \
    --cfg ${CFG} \
    --input "My remark pleases him, but I soon prove to him that it is not the right way to speak. However perfect may have been the language of that ancient writer." \
    --seed 42

# Or Streaming Inference
python scripts/run_stream.py \
    --model_name_or_path ${CHECKPOINT} \
    --cfg ${CFG} \
    --input "My remark pleases him, but I soon prove to him that it is not the right way to speak. However perfect may have been the language of that ancient writer." \
    --seed 42
# Please note that we have simulated the generation in a streaming environment in run_stream.py for evaluating its quality.
# However, the existing code does not actually provide a streaming API.
```

### Training

*Data Processing*
#TODO

*Training Offline Model*
``` sh
OUTPUT_DIR=./runs/libriheavy
mkdir -p $OUTPUT_DIR
LOG_FILE=${OUTPUT_DIR}/log

BATCH_SIZE=8
UPDATE_FREQ=8
# assume 8 proc per node, then WORLD_SIZE * 8 * BATCH_SIZE * UPDATE_FREQ == 512

torchrun --nnodes ${WORLD_SIZE} --node_rank ${RANK} --nproc_per_node 8 --master_addr ${MASTER_ADDR} --master_port ${MASTER_PORT} \
    ./scripts/train_libriheavy.py \
    --training_cfg 0.1 \
    --num_hidden_layers 12 --diffloss_d 6 --noise_channels 128 \
    --dataloader_num_workers 8 \
    --dataloader_pin_memory True \
    --remove_unused_columns False \
    --label_names audio_inputs \
    --group_by_speech_length \
    --do_train \
    --do_eval \
    --eval_strategy steps \
    --eval_steps 10000 \
    --prediction_loss_only \
    --per_device_train_batch_size ${BATCH_SIZE} \
    --per_device_eval_batch_size 24 \
    --gradient_accumulation_steps ${UPDATE_FREQ} \
    --bf16 \
    --learning_rate 5e-4 \
    --weight_decay 0.01 \
    --adam_beta1 0.9 \
    --adam_beta2 0.999 \
    --adam_epsilon 1e-8 \
    --max_grad_norm 1.0 \
    --max_steps 300000 \
    --lr_scheduler_type "linear" \
    --warmup_steps 32000 \
    --logging_first_step \
    --logging_steps 100 \
    --save_steps 10000 \
    --save_total_limit 10 \
    --output_dir ${OUTPUT_DIR} \
    --report_to tensorboard \
    --disable_tqdm True \
    --ddp_timeout 3600 --overwrite_output_dir

```

*Training Streaming Model*
``` sh
OUTPUT_DIR=./runs/libriheavy_stream
mkdir -p $OUTPUT_DIR
LOG_FILE=${OUTPUT_DIR}/log

BATCH_SIZE=8
UPDATE_FREQ=8
# assume 8 proc per node, then WORLD_SIZE * 8 * BATCH_SIZE * UPDATE_FREQ == 512

torchrun --nnodes ${WORLD_SIZE} --node_rank ${RANK} --nproc_per_node 8 --master_addr ${MASTER_ADDR} --master_port ${MASTER_PORT} \
    ./scripts/train_libriheavy_stream.py \
    --finetune_path ./runs/libriheavy/checkpoint-300000/model.safetensors \
    --stream_n 5 --stream_m 45 \
    --training_cfg 0.1 \
    --num_hidden_layers 12 --diffloss_d 6 --noise_channels 128 \
    --dataloader_num_workers 8 \
    --dataloader_pin_memory True \
    --remove_unused_columns False \
    --label_names audio_inputs \
    --group_by_speech_length \
    --do_train \
    --do_eval \
    --eval_strategy steps \
    --eval_steps 10000 \
    --prediction_loss_only \
    --per_device_train_batch_size ${BATCH_SIZE} \
    --per_device_eval_batch_size 24 \
    --gradient_accumulation_steps ${UPDATE_FREQ} \
    --bf16 \
    --learning_rate 3e-4 \
    --weight_decay 0.01 \
    --adam_beta1 0.9 \
    --adam_beta2 0.999 \
    --adam_epsilon 1e-8 \
    --max_grad_norm 1.0 \
    --max_steps 100000 \
    --lr_scheduler_type "linear" \
    --warmup_steps 10000 \
    --logging_first_step \
    --logging_steps 100 \
    --save_steps 10000 \
    --save_total_limit 10 \
    --output_dir ${OUTPUT_DIR} \
    --report_to tensorboard \
    --disable_tqdm True \
    --ddp_timeout 3600 --overwrite_output_dir
```


## Code Contributors

- [Zhengrui Ma](https://scholar.google.com/citations?user=dUgq6tEAAAAJ)
- [Chenze Shao](https://scholar.google.com/citations?user=LH_rZf8AAAAJ)



## Ackonwledgement
This work is inspired by following great works:
- A Proper Loss Is All You Need: Autoregressive Image Generation in Continuous Space via Score Maximization
- Autoregressive Image Generation without Vector Quantization
- A Spectral Energy Distance for Parallel Speech Synthesis

## Citation
#TODO
