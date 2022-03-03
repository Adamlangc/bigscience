# tr11 176B ML

Large multilingual language model training

## Task

Auto-regressive objective using regular Megatron-LM GPT2 language model w/o multi-lingual dataset

Model size: 176B

## Environment

To launch the environment use [start-tr11-176B-ml](./start-tr11-176B-ml)

XXX: I need to prepare it

```
source $six_ALL_CCFRWORK/code/tr11-176B-ml/bigscience/train/tr11-176B-ml/start-tr11-176B-ml
```


## Model Setup

### Packages

- pytorch-1.11-to-be (using a release candidate and will update to final release when it's out) - we must use it for its NCCL version which supports BF16 comms (the NCCL version that comes with pt-1.10 doesn't)

- `tokenizers` requires a special branch `bigscience_fork` which also requires manual building:

```
# to build custom tokenizers make sure that if run on JZ your `~/.cargo/config.toml` contains the following:
[net]
git-fetch-with-cli = true

# if needed first:
# git clone https://github.com/huggingface/tokenizers $six_ALL_CCFRWORK/code/tokenizers
cd $six_ALL_CCFRWORK/code/tokenizers
git checkout bigscience_fork
module load rust
pip install setuptools_rust
pip install -e bindings/python
```
- `transformers` - any version

- `datasets` - any version

- `apex` - any version


### Architecture

```
NHIDDEN=14336
NLAYERS=70
NHEADS=112

GPT_ARGS=" \
    --num-layers $NLAYERS \
    --hidden-size $NHIDDEN \
    --num-attention-heads $NHEADS \
    [...]
    "
```

Sanity check:
```
$ NHIDDEN=14336; NLAYERS=70; NHEADS=112; SEQ_LEN=2048; VOCAB_SIZE=250000; python -c "h=$NHIDDEN; l=$NLAYERS; n=$NHEADS; s=$SEQ_LEN; v=$VOCAB_SIZE; print(f'Model size: {(l*(12*h**2 + 13*h) + v*h + s*h + 2*h) / 10**9 :.0f}B, hidden/layers ratio: {int(h/l)}, hidden/heads ratio: {int(h/n)}')"
Model size: 176B, hidden/layers ratio: 204, hidden/heads ratio: 128
```



### Sequence Length

Default Megatron-LM language model with 2048 tokens sequence length

```
SEQ_LEN=2048

    --seq-length $SEQ_LEN \
    --max-position-embeddings $SEQ_LEN \

```



### Replica setup

GPUs = 384 (48 nodes of 8)

```
TP=4
PP=12
DP=8
MBS=2
```

One replica is 48 GPUs (`TP*PP=4*12`)

MBS=2 performs the fastest in this setup w/o using too much additional memory.


### Global batch size

GBS = Global Batch Size

8 replicas -> with MBS=2 (8*2) can do GBS increments of 16 (2 samples per replica).

Use a schedule:

- start from 32k tokens (GBS=16)
- increase linearly to 4.2M tokens/step (GBS=2048) over 9_765_625 samples (~20B tokens)
- then continue at 4.2M tokens/step (GBS=2048) for 210M samples (430B tokens / ~102K steps)

Total: 450B tokens / 220M samples

syntax:
```
--rampup-batch-size <start batch size>  <batch size increment> <ramp-up samples>
```

At seqlen 2048 (1k tokens is bs=1), we get:

```
GLOBAL_BATCH_SIZE=2048
TRAIN_SAMPLES=220_000_000

    --rampup-batch-size 16 16 9_765_625 \
    --global-batch-size $GLOBAL_BATCH_SIZE \
```

Notes:
* `--rampup-batch-size` requires the use of `--train-samples` and can't be used with `--train-iters`.
* global batch size has to be divisible by micro-batch-size * DP_SIZE



### Optimizer

- AdamW, β1=0.9, β2=0.95, eps=1e−8
- learning rate:
   * peak=6e-5
   * warmup over 183_105 samples (375M tokens)
   * cosine decay for learning rate down to 10% of its value, over 410B tokens (after 410B tokens, training continues at 10% of the original learning rate, that is fixed `--min-lr`)
- clipping by global norm of 1 (as in GPT-3)
- weight decay of 0.1

We need lr-decay in samples, so tokens2samples = 410B / 2048 = ~200_000_000


```
LR_DECAY_SAMPLES=200_000_000
LR_WARMUP_SAMPLES=183_105  # 375M tokens

    --optimizer adam \
    --adam-beta1 0.9 \
    --adam-beta2 0.95 \
    --adam-eps 1e-8 \
    --lr 6e-5 \
    --min-lr 6e-6 \
    --lr-decay-style cosine \
    --lr-decay-samples $LR_DECAY_SAMPLES \
    --lr-warmup-samples $LR_WARMUP_SAMPLES \
    --clip-grad 1.0 \
    --weight-decay 1e-1 \
```

### std Init

This proved to be a very crucial setting in our 104B experiments and we couldn't break past the first few thousands iterations until we figured out the 0.02 default `--init-method-std` was a way too big.

1. "Transformers without Tears" paper https://arxiv.org/abs/1910.05895 prescribes: `sqrt(2/(NHIDDEN*5))`

2. The 530B training paper https://arxiv.org/abs/2201.11990 they used an even smaller init formula: `sqrt(1/(NHIDDEN*3))`

and we decided to go with the 530B one as it leads to an even smaller init value.

To make it easier to compare the two formulas, they can be rewritten as:
1. `sqrt(0.4000/NHIDDEN)`
2. `sqrt(0.3333/NHIDDEN)`

Thus: `sqrt(1/(14336*3)) = 0.00482197968631537`

```
    --init-method-std 0.0048 \
```

### Misc features


We use the added by us AliBi implementation:

```
    --position-embedding-type alibi \
```

We use the added by us embedding norm which makes the training more stable at a small training slowdown cost and a tiny additional amount of memory.

```
    --embed-layernorm \
```


### Tokenizer


```
    --tokenizer-type PretrainedFromHF \
    --tokenizer-name-or-path bigscience-catalogue-data-dev/byte-level-bpe-tokenizer-nfkc-250k \
```



### Data


XXX: needs cleanup and notes to where and why

```
# TODO: fix data settings
BIGSCIENCE_REPO=/gpfswork/rech/six/uty16tp/code/big_science/bigscience
TRAIN_DATA_PATH=$MEGATRON_DEEPSPEED_REPO/data/train-splits.txt
VALID_DATA_PATH=$MEGATRON_DEEPSPEED_REPO/data/valid-splits.txt
TEST_DATA_PATH=$MEGATRON_DEEPSPEED_REPO/data/test-splits.txt
CATALOGUE_JSON_PATH=$BIGSCIENCE_REPO/data/catalogue/training_dataset_ratios_batch_0_per_language.json
LOAD_RATIOS_SCRIPT=$BIGSCIENCE_REPO/data/catalogue/load_ratios_meg_ds_format.py
python $LOAD_RATIOS_SCRIPT --dataset-ratios-path $CATALOGUE_JSON_PATH --split train --output-meg-ds-ratio-file $TRAIN_DATA_PATH
python $LOAD_RATIOS_SCRIPT --dataset-ratios-path $CATALOGUE_JSON_PATH --split valid --output-meg-ds-ratio-file $VALID_DATA_PATH
python $LOAD_RATIOS_SCRIPT --dataset-ratios-path $CATALOGUE_JSON_PATH --split test --output-meg-ds-ratio-file $TEST_DATA_PATH
```


### Data type

We are using `bfloat16` since it's supposed to be more stable than `float16`

```
    --bf16 \
```
and the rest is in the Deepspeed config


### Deepspeed config


The new `BF16_Optimizer` implements its own ZeRO Stage 1, hence until it gets its own stage number, we must use:
```
ZERO_STAGE=0
config_json="./ds_config.$SLURM_JOBID.json"

# Deepspeed figures out GAS dynamically from dynamic GBS via set_train_batch_size()
cat <<EOT > $config_json
{
  "train_micro_batch_size_per_gpu": $MICRO_BATCH_SIZE,
  "train_batch_size": $GLOBAL_BATCH_SIZE,
  "gradient_clipping": 1.0,
  "zero_optimization": {
    "stage": $ZERO_STAGE
  },
  "bf16": {
    "enabled": true
  },
  "steps_per_print": 2000,
  "wall_clock_breakdown": false
}
EOT

```

The new `BF16_Optimizer` accumulates grads in fp32. It doesn't shard the static buffer it reuses, which consumes 4 bytes * params additional memory, but since it's not sharding it saves on comms. Down the road, it'll be expanded to support sharding on demand.

Using Deepspeed's activation checkpointing to use a lot less GPU memory:

```
    --deepspeed-activation-checkpointing \
```



### Important environment variables

The usual set to tell where things are and that we are working w/o internet on the compute nodes:

```
export TRANSFORMERS_CACHE=$six_ALL_CCFRWORK/models
export HF_DATASETS_CACHE=$six_ALL_CCFRWORK/datasets
export HF_MODULES_CACHE=$six_ALL_CCFRWORK/modules
export HF_METRICS_CACHE=$six_ALL_CCFRWORK/metrics
export HF_DATASETS_OFFLINE=1
export TRANSFORMERS_OFFLINE=1

```


There is some complex hanging problem that occurs under certain conditions with 40+ nodes, which the following settings solves:

```
export CUDA_LAUNCH_BLOCKING=1
```
in theory it should make everything much slower but makes a tiny impact or no impact at all to the throughput.


To hide duplicated errors using this hack - will be properly fixed in pt-1.12

```
export TORCHELASTIC_ERROR_FILE=/tmp/torch-elastic-error.json
```
using `/tmp/` on purpose here so that each node will have a different target.




### Kill Switch

This is a feature that allows us to "kill" a SLURM job started by a user who isn't around at the moment, since SLURM doesn't support groups and we don't have `sudo` access. But basically we get the program to poll for a file at startup and before each iteration and it'll quit if it finds this file.

For an explanation on how it works see: [Kill Switch](../../jz/slurm/README.md#kill-switch)

XXX: sync to the final path

To arm:

```
KILL_SWITCH_PATH=$MEGATRON_DEEPSPEED_REPO/kill-switch-tr11-176B-exp1

    --kill-switch-path $KILL_SWITCH_PATH \
```

To trigger:

```
touch $MEGATRON_DEEPSPEED_REPO/kill-switch-tr11-176B-exp1
```

To deactivate and let new instances of a job run normally:

```
rm  $MEGATRON_DEEPSPEED_REPO/kill-switch-tr11-176B-exp1
```


### launcher

We are using the latest elastic-based launcher with `c10d` backend.

```
MASTER_ADDR=$(scontrol show hostnames $SLURM_JOB_NODELIST | head -n 1)
MASTER_PORT=6000
GPUS_PER_NODE=8
NNODES=$SLURM_NNODES

export LAUNCHER="python -u -m torch.distributed.run \
    --nproc_per_node $GPUS_PER_NODE \
    --nnodes $NNODES \
    --rdzv_endpoint $MASTER_ADDR:$MASTER_PORT \
    --rdzv_backend c10d \
    --max_restarts 0 \
    --tee 3 \
    "
```

`--tee 3` prefixes all logs with the local rank, which helps to unravel interleaved error messages by grepping for one of the local rank prefixes, e.g.:
```
grep `[default7]` main_log.txt
```