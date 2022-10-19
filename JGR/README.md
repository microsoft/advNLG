# Joint Generator-Ranker Learning for Natural Language Generation
This repo contains the code, data and trained models for our paper [Joint Generator-Ranker Learning for Natural Language Generation](https://arxiv.org/abs/2206.13974).

## Quick Links

- [Requirements](#requirements)
- [Description of Codes](#description-of-codes)
- [Preprocessing](#preprocessing)
- [How to Run](#how-to-run)
  -  [Warm-up generator](#warm-up-generator)
  -  [First ranker training iteration](#first-ranker-training-iteration)
  -  [JGR training](#jgr-training)
  -  [Evaluate](#evaluate)
- [Checkpoints](#Checkpoints)

## Requirements

- torch>=1.7
- transformers==4.8.1
- datasets==1.12.1
- nltk==3.7
- rouge-score

## Description of Codes
- `data` -> directories to store datasets and the data preprocessing codes.
- `data_utils` -> codes for dataloader and evaluation metrics
- `model_utils` -> generator model and ranker model
- `trainer_utils` -> trainer and trainer configuration
- `warmup-generato` -> warming-up generator
- `warmup-ranker` -> first training iteration of ranker
- `run_train.py` -> the main function to run JGR

## Preprocessing

Now JGR can be used in CNN/DailyMail, SAMSum, SquadQG and Personachat. Users should download the raw data of these datasets and run the codes in `./data` to preprocess them. The codes will preprocess and re-orgnize the data samples into json file for later steps. For more details, please check the `README.md` in `./data`, it will give you the details of preprocessing each dataset.

## How to run

### Warm-up generator

To achieve a better performance of JGR, it's neccessary to pre-finetune the generator with MLE loss on the target training set. Follow the steps described in `./warmup-generator/README.md` and you will get a warm-uped generator for target dataset stored in `./warmup-generator/saves`. 

**Notes**: Before the first ranker training iteration. Your should use the warm-uped generator to generator the candidates for the first ranker training iteration. Don't forget to execute `2. Generate candidates for warming-up ranker` in `./warmup-generator/README.md`.

### First ranker training iteration

As mentioned in the paper, in order to initialize the ranker with a more general and reasonable ranking function, we increase the number of training steps and add a certain number of warm-up steps at the first ranker training iteration. Here we use the fine-tuned generator to generator the candidates for the first ranker training iteration. Follow the steps described in `./warmup-ranker/README.md` and you will get a warm-uped generator for target dataset stored in `./warmup-ranker/saves`.

### JGR training

After obtaining the warm-uped generator and ranker, you can now turn to JGR-training. Taking cnndm as example, to train JGR, run:
```
EXPORT generator=warmup-generator/saves/bart-large-cnndm # the warm-uped generator
EXPORT ranker=warmup-ranker/saves/roberta-large-cnndm # the ranker after first iteration
EXPORT save_name=JGR-large-cnndm # the save name of model

python -m torch.distributed.launch --nproc_per_node 8  run_train.py --overwrite_output_dir \
    --task_name sum --dataset_name cnndm \
    --train_data_path data/cnndm \
    --dev_data_path data/cnndm \
    --test_data_path data/cnndm \
    --load_tokenized_data False \
    --evaluate_generator True \
    --generator_num_cand_generated 8 --generator_num_cand_picked 8 \
    --num_cand_generated 16 --num_cand_picked 3 --candidate_pick_strategy bottom \
    --do_train True --do_eval True --do_predict True --prediction_loss_only False \
    --per_device_train_batch_size 2 --per_device_eval_batch_size 4 \
    --gradient_accumulation_steps 4 \
    --generator_learning_rate 5e-5 --reranker_learning_rate 1e-5 \
    --num_train_epochs 3 \
    --evaluation_strategy steps --eval_steps 1000 \
    --logging_strategy steps --logging_steps 500 \
    --save_strategy steps --save_steps 1000 --save_total_limit 20 \
    --iteration_steps 1000 --iteration_reranker_steps 500 \
    --load_best_model_at_end True \
    --metric_for_best_model generator_eval_rouge1 --greater_is_better True \
    --reranker_model_name_or_path $ranker \
    --generator_model_name_or_path $generator \
    --output_dir saves/$save_name \
    --generator_max_source_length 1020 --reranker_max_source_length 400 --generator_max_target_length 109 --reranker_max_target_length 109 \
    --cache_data \
    --disable_tqdm False 

```

The above instructions will store the trained generator and ranker in `saves/JGR-large/cnndm/generator` and `saves/JGR-large/cnndm/reranker`, respectively. For the JGR taining on other datasets, check `run_train.sh`.

### Evaluate

To evaluate the trained generator and ranker, run:

```

EXPORT generator=saves/JGR-large-cnndm/generator # the trained  generator
EXPORT ranker=saves/JGR-large-cnndm/ranker # the  trained iteration
EXPORT save_name=JGR-large-cnndm # the save name of model

python -m torch.distributed.launch --nproc_per_node 8  run_train.py --overwrite_output_dir \
    --task_name sum --dataset_name cnndm \
    --train_data_path data/cnndm \
    --dev_data_path data/cnndm \
    --test_data_path data/cnndm \
    --load_tokenized_data False \
    --generator_num_cand_generated 8 --generator_num_cand_picked 8 \
    --num_cand_generated 16 --num_cand_picked 3 --candidate_pick_strategy bottom \
    --do_predict True --prediction_loss_only False \
    --per_device_eval_batch_size 4 \
    --evaluation_strategy steps --eval_steps 1000 \
    --logging_strategy steps --logging_steps 500 \
    --save_strategy steps --save_steps 1000 --save_total_limit 20 \
    --iteration_steps 1000 --iteration_reranker_steps 500 \
    --load_best_model_at_end True \
    --metric_for_best_model generator_eval_rouge1 --greater_is_better True \
    --reranker_model_name_or_path $ranker \
    --generator_model_name_or_path $generator \
    --output_dir saves/$save_name \
    --generator_max_source_length 1020 --reranker_max_source_length 400 --generator_max_target_length 109 --reranker_max_target_length 109 \
    --cache_data \
    --disable_tqdm False 

```


## Checkpoints

We also provide our trained checkpoints for you: 

|          | Generator | Ranker |
|----------|---------|---------|
| CNNDM    | [generator.zip](https://msraprophetnet.blob.core.windows.net/jgr/saved_models/cnndm/generator.zip)  | [reranker.zip](https://msraprophetnet.blob.core.windows.net/jgr/saved_models/cnndm/reranker.zip) | 
| CNNDM (initialized with BRIO)   | [generator.zip](https://msraprophetnet.blob.core.windows.net/jgr/saved_models/cnndm-BRIO/generator.zip)  | [reranker.zip](https://msraprophetnet.blob.core.windows.net/jgr/saved_models/cnndm-BRIO/reranker.zip) | 
| SAMSum     |  [generator.zip](https://msraprophetnet.blob.core.windows.net/jgr/saved_models/samsum/generator.zip)  | [reranker.zip](https://msraprophetnet.blob.core.windows.net/jgr/saved_models/samsum/reranker.zip) | 
| SquadQG     |  [generator.zip](https://msraprophetnet.blob.core.windows.net/jgr/saved_models/squadqg/generator.zip)  | [reranker.zip](https://msraprophetnet.blob.core.windows.net/jgr/saved_models/squadqg/reranker.zip) | 
| Personachat     |  [generator.zip](https://msraprophetnet.blob.core.windows.net/jgr/saved_models/personachat/generator.zip)  | [reranker.zip](https://msraprophetnet.blob.core.windows.net/jgr/saved_models/personachat/reranker.zip) | 
