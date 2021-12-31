[![GitHub issues](https://img.shields.io/github/issues/EleutherAI/gpt-neox)](https://github.com/EleutherAI/gpt-neox/issues)
[<img src="https://raw.githubusercontent.com/wandb/assets/main/wandb-github-badge-28.svg" alt="Weights & Biases monitoring" height=20>](https://wandb.ai/eleutherai/neox)

# GPT-NeoX

This repository records [EleutherAI](https://www.eleuther.ai)'s work-in-progress for training large-scale language models on GPUs. Our current framework is based on NVIDIA's [Megatron Language Model](https://github.com/NVIDIA/Megatron-LM) and has been augmented with techniques from [DeepSpeed](https://www.deepspeed.ai) as well as some novel optimizations. 

We aim to make this repo a centralized and accessible place to gather techniques for training large-scale autoregressive language models, and accelerate research into large-scale training. Additionally, we hope to train and open source a 175B parameter GPT-3 replication along the way. 

If you are interested in contributing, please [join our Discord](https://discord.gg/zBGx3azzUn) and head to the `#gpt-neox` channel. We're working with cloud compute provider [CoreWeave](https://www.coreweave.com/) for training, and hope to release the weights of smaller models as we progress up to 175B parameters.

If you're looking for our TPU codebase, see [GPT-Neo](https://github.com/EleutherAI/gpt-neo).

- [Why GPT-NeoX](#why-gpt-neox)
- [Quick Start](#quick-start)
- [Features](#features)
  * [3D Parallelism](#3d-parallelism)
  * [Model Structure](#model-structure)
  * [Optimizers](#optimizers)
  * [High-Precision Training](#high-precision-training)
- [Datasets](#datasets)
  * [Preconfigured Datasets](#preconfigured-datasets)
  * [Using Custom Data](#using-custom-data)
  * [Using and Training Tokenizers](#using-and-training-tokenizers)
- [Training and Finetuning](#training-and-finetuning)
- [Inference](#inference)
- [Evaluation](#evaluation)
- [Distilling](#distilling)
- [Monitoring](#monitoring)
  * [Weights & Biases](#wandb)
  * [Tensorboard](#tensorboard)
- [Administrative Notes](#administrative-notes)
  * [Citing GPT-NeoX](#citing-gpt-neox)
  * [Licensing](#licensing)
  * [Acknowledgements](#acknowledgements)

## Why GPT-NeoX

**Straightforward configuration:** Other libraries such as Megatron-LM require you configure them using command line arguments and global variables, which can often be difficult to work with and iterate upon. We offer straightforward configuration using YAML files, which enables you to launch training runs across hundreds of GPUs with a single line bash script. Additionally, we hope to make data preparation easier on the user by providing scripts to automatically download and pretokenize a number of large-scale datasets.

**Diverse Modeling Options:** We provide a wide collections of options for constructing your model.

**Scalability:** Our codebase is efficient for and effective at training transformers with tens of billions of parameters.

## Quick Start

### Environment and Dependencies

<aside>

**Warning:** Our codebase relies on [DeeperSpeed](https://github.com/EleutherAI/DeeperSpeed), our fork of the [DeepSpeed](https://github.com/microsoft/DeepSpeed) library with some added changes. We strongly recommend using Anaconda, a virtual machine, or some other form of environment isolation before continuing. Failure to do so may cause other repositories that rely on DeepSpeed to break.

</aside>

#### Host Setup

First make sure you are in an environment with Python 3.8 or later with an appropriate version of PyTorch 1.8 or later installed.

To install the remaining basic dependencies, run `pip install -r requirements/requirements.txt`. Afterwards, run `python ./megatron/fused_kernels/setup.py install` from the repository root to install fused kernels.

There are other optional features we support when the required packages are installed. [Fused optimizers](https://nvidia.github.io/apex/optimizers.html) are available if [Apex](https://github.com/NVIDIA/apex) is installed. Features enabled by DeepSpeed/DeeperSpeed can be installed by installing DeepSpeed/DeeperSpeed with those features, or alternatively installing the corresponding requirements file found in `./requirements`. You may need to use the appropriate version of `cupy` to match the local CUDA version when installing the requirements for 1-bit Adam.

#### Containerized Setup

It may be preferable to run GPT-NeoX in a container, and we provide a Dockerfile configured to take care of much of the installation process for this purpose.

To use this option, first build an image named `gpt-neox` from the repository root directory with `docker build -t gpt-neox -f Dockerfile .`. We also host pre-built images on Docker Hub at `leogao2/gpt-neox`.

You can then run a container based on this image. For instance, the below snippet mounts the cloned repository (`gpt-neox`) directory to `/gpt-neox` in the container and uses [nvidia-docker](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) to make four GPUs (numbers 0-3) accessible to the container. [As noted by the NCCL documentation](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/troubleshooting.html#sharing-data), both `--shm-size=1g` and `--ulimit memlock=-1` are important to prevent Docker from allocating too little shared memory.
```
nvidia-docker run --rm -it -e NVIDIA_VISIBLE_DEVICES=0,1,2,3 --shm-size=1g --ulimit memlock=-1 --mount type=bind,src=$PWD,dst=/gpt-neox gpt-neox
```

### Running the Code
Once you've installed all the requirements and set up your model configuration, the next step is obtaining and preprocessing your dataset. We provide a data processing library that is easily interfaced with via the function `prepare_data.py`. Calling `python prepare_data.py enron -t CharLevelTokenizer -d ./data/` will download the dataset `enron`, tokenize it with a character-level tokenizer, and save the results to `./data/`. 

GPT-NeoX parameters are defined in a YAML configuration file which is passed to the `deepy.py` launcher. We provide baseline examples for the models found in the paper [Language Models are Few Shot Learners](https://arxiv.org/abs/2005.14165). Items such as file locations that are dependant on your particular system go in `local_setup.yml`. We have filled it out with some placeholder examples, but you will need to update this for your system.

All functionality follows the pattern `python ./deepy.py [main_function.py] -d [config_dir] [path/to/config1.yml] [path/to/config2.yml] ...`. 
We currently offer three main functions:
1. `train.py` is used for training and finetuning models.
2. `evaluate.py` is used to evaluate a trained model using the [language model evaluation harness](https://github.com/EleutherAI/lm-evaluation-harness).
3. `generate.py` is used to sample text from a trained model.

For now, run `python ./deepy.py train.py -d configs small.yml local_setup.yml` to begin training a model and complete this tutorial.

## Features

GPT-NeoX offers a wide variety of state-of-the-art and bespoke features.

### 3D Parallelism

- GPT-NeoX offers full 3D parallelism (data, model and pipeline parallel) using DeepSpeed, allowing you to scale model training to hundreds of billions of parameters across multiple GPUs.

### Model Structure

- **Positional Encodings:**

    - Choose between [sinusoidal](https://arxiv.org/abs/1706.03762), [T5-style relative bias](https://arxiv.org/abs/1910.10683), [GPT-style learned](https://cdn.openai.com/research-covers/language-unsupervised/language_understanding_paper.pdf), [Rotary Position Embedding](https://arxiv.org/abs/2104.09864), and [no](https://arxiv.org/abs/1905.04226)[ne](https://arxiv.org/abs/2102.11174). Use the <code><var>pos-emb</var></code> field to select a positional encoding.

- **Sparsity:**

    - Deepspeed's sparse attention kernels are supported, but don't work with CUDA 11.0+, and require a specific hardware setup (V100s/RTX2080s/A100s). Add `"sparsity": "all"` to your config file to use sparse attention on all layers, or `"sparsity": "interspersed"` to use it every other layer. To use sparsity, first run `pip install -r requirements/requirements-sparseattention.txt` to install Triton.

- **Normalization:**

    - We offer a choice of [LayerNorm](https://arxiv.org/abs/1607.06450), [ScaleNorm](https://arxiv.org/abs/1910.05895) and [RMSNorm](https://arxiv.org/abs/1910.07467). Use the <code><var>norm</var></code> field to select a normalization.

### Optimizers

- GPT-NeoX supports [Adam](https://arxiv.org/abs/1412.6980), CPUAdam, [1-Bit Adam](https://arxiv.org/abs/2102.02888), [SM3](https://arxiv.org/abs/1901.11150) and [MADGRAD](https://arxiv.org/abs/2101.11075) optimizers, as well as DeepSpeed's [Zero Redundancy Optimizer](https://arxiv.org/abs/1910.02054). Use the <code><var>optimizer</var></code> and (if applicable) <code><var>zero_optimization</var></code> fields to configure your optimizer.

- **Zero Redundancy Optimizer (ZeRO):**
    - ZeRO Stage 1 works seamlessly with GPT-NeoX, while ZeRO Stage 2 requires pipeline parallelism be disabled. We are additionally working on integrating ZeRO Stage 3 into the codebase. Turning on ZeRO is as simple as adding one field to your configuration file.

### High-Precision Training

- Choose between single precision, half precision, and bfloat16 operations to get the most performance out of your avaliable compute. Use the <code><var>precision</var></code> field to configure your precision settings.
    - Prior to [NCCL 2.10.3](https://docs.nvidia.com/deeplearning/nccl/release-notes/rel_2-10-3.html) and [PyTorch 1.10](https://github.com/pytorch/pytorch/releases/tag/v1.10.0), models trained in bfloat16 were required to to perform all-reduce operations in single precision. This can enabled with `"fp32_allreduce": True`.
    - Additionally, you have to run `python ./megatron/fused_kernels/setup.py install` (assuming you're inside the root directory of the repository) to be able to use bfloat16 (may require root access).

## Datasets


### Preconfigured Datasets

For demonstrative purposes we've hosted the Enron Emails corpus and made it available for downloading. Running `python ./prepare_data.py` will download the tokenizer files and dataset, pretokenize the dataset, and save it into a folder named `./data`.

In the future we will also be adding a single command to preprocess our 800GB language modelling dataset, [The Pile](https://arxiv.org/abs/2101.00027), and all its constituent datasets.

To prepare your own dataset for training, format it as one large [jsonl](https://jsonlines.org/)-formated file with each item in the list of dictionaries being a separate document.
The document text should be grouped under one JSON key, i.e `"text"`. 

Next make sure to download the GPT2 tokenizer vocab, and merge files from the following links:

- Vocab: https://s3.amazonaws.com/models.huggingface.co/bert/gpt2-vocab.json
- Merge: https://s3.amazonaws.com/models.huggingface.co/bert/gpt2-merges.txt

### Using Custom Data

### Using and Training Tokenizers

You can now pretokenize your data using `tools/preprocess_data.py`.

Usage:

```
preprocess_data.py [-h] --input INPUT [--json-keys JSON_KEYS [JSON_KEYS ...]] [--split-sentences] [--keep-newlines] --tokenizer-type {BertWordPieceLowerCase,BertWordPieceCase,GPT2BPETokenizer} [--vocab-file VOCAB_FILE] [--merge-file MERGE_FILE] [--append-eod]
                          --output-prefix OUTPUT_PREFIX [--dataset-impl {lazy,cached,mmap}] [--workers WORKERS] [--log-interval LOG_INTERVAL]

input data:
  --input INPUT         Path to input JSON
  --json-keys JSON_KEYS [JSON_KEYS ...]
                        space separate listed of keys to extract from json. default = "text".
  --split-sentences     Split documents into sentences.
  --keep-newlines       Keep newlines between sentences when splitting.

tokenizer:
  --tokenizer-type {GPT2BPETokenizer}
                        What type of tokenizer to use.
  --vocab-file VOCAB_FILE
                        Path to the vocab file
  --merge-file MERGE_FILE
                        Path to the BPE merge file (if necessary).
  --append-eod          Append an <eod> token to the end of a document.

output data:
  --output-prefix OUTPUT_PREFIX
                        Path to binary output file without suffix
  --dataset-impl {lazy,cached,mmap}

runtime:
  --workers WORKERS     Number of worker processes to launch
  --log-interval LOG_INTERVAL
                        Interval between progress updates

```

For example:

```bash
python tools/preprocess_data.py \
            --input data/mydataset.jsonl \
            --output-prefix data/mydataset \
            --vocab data/gpt2-vocab.json \
            --dataset-impl mmap \
            --tokenizer-type GPT2BPETokenizer \
            --merge-file gpt2-merges.txt \
            --append-eod
```

You would then run training with the following settings added to your configuration file:

```yaml
  "data-path": "data/mydataset/mydataset",
```

## Training and Finetuning

Training is launched using `deepy.py`, a wrapper around DeepSpeed's launcher, which launches the same script in parallel across many GPUs / nodes.

The general usage pattern is:

```bash
python ./deepy.py train.py [path/to/config1.yml] [path/to/config2.yml] ...
```

You can pass in an arbritrary number of configs which will all be merged at runtime.

You can also optionally pass in a config prefix, which will assume all your configs are in the same folder and append that prefix to their path.

Example usage:

```bash
python ./deepy.py train.py -d configs small.yml local_setup.yml
```

This will deploy the `train.py` script on all nodes with one process per GPU. The worker nodes and number of GPUs are specified in the `/job/hostfile` file (see [parameter documentation](configs)), or can simply be passed in as the `num_gpus` arg if running on a single node setup.
* Model parameters are defined in the config file `configs/small.yml`.
* Data path parameters are defined in the config file `configs/local_setup.yml`. If you are an EleutherAI member and using the [Kubernetes cluster](kubernetes), the `eleutherai_cluster.yml` config should be used instead.

## Inference

We support three types of generation from a pretrained model:
1. Unconditional generation
2. Conditional generation based on an input read from a file
3. Interactive generation, which allows for multiple rounds of back-and-forth between a user and the language model via a command line interface

All three types of text generation can be launched via `python ./deepy.py generate.py -d configs small.yml local_setup.yml text_generation.yml` with the appropriate values set in `configs/text_generation.yml`.

## Evaluation

GPT-NeoX supports evaluation on downstream tasks through the [language model evaluation harness](https://github.com/EleutherAI/lm-evaluation-harness).

To evaluate a trained model on the evaluation harness, add an <code><var>eval_tasks</var></code> field to your config file and call `python ./deepy.py evaluate.py -d configs your_configs.yml`.

## Distilling

Coming soon! Check out the `distill-gpt-neox` branch to try distilling a model.

## Monitoring

In addition to storing logs locally, we provide built-in support for two popular experiment monitoring frameworks: [Weights & Biases](https://wandb.ai/site) and [TensorBoard](https://www.tensorflow.org/tensorboard/)

<h3 id="wandb">Weights & Biases</h3>

EleutherAI is currently using [Weights & Biases to record our experiments](https://wandb.ai/eleutherai/neox). If you are logged into Weights & Biases on your machine&mdash;you can do this by executing `wandb login`&mdash;your runs will automatically be recorded. There are two optional fields associated with Weights & Biases: <code><var>wandb_group</var></code> allows you to name the run group and <code><var>wandb_team</var></code> allows you to assign your runs to an organization or team account.

### TensorBoard

We also support using TensorBoard via the <code><var>tensorboard-dir</var></code> field. Dependencies required for TensorBoard monitoring can be found in and installed from  `./requirements/requirements-tensorboard.txt`.

## Administrative Notes

### Citing GPT-NeoX

If you have found GPT-NeoX helpful in your work, you can cite this repository as

```bibtex
@software{gpt-neox,
  author = {Andonian, Alex and Anthony, Quentin and Biderman, Stella and Black, Sid and Gali, Preetham and Gao, Leo and Hallahan, Eric and Levy-Kramer, Josh and Leahy, Connor and Nestler, Lucas and Parker, Kip and Pieler, Michael and Purohit, Shivanshu and Songz, Tri and Wang, Phil and Weinbach, Samuel},
  title = {{GPT-NeoX}: Large Scale Autoregressive Language Modeling in PyTorch},
  url = {http://github.com/eleutherai/gpt-neox},
  year = {2021}
}
```

In the above BibTex entry, names are in alphabetical order, and the year corresponds to the project's open-source release.

### Licensing

This repository hosts code that is part of EleutherAI's GPT-NeoX project. Copyright &copy; 2021, EleutherAI contributors (in alphabetical order): Alex Andonian, Quentin Anthony, Stella Biderman, Sid Black, Preetham Gali, Leo Gao, Eric Hallahan, Josh Levy-Kramer, Connor Leahy, Lucas Nestler, Kip Parker, Michael Pieler, Shivanshu Purohit, Tri Songz, Phil Wang, Samuel Weinbach. Licensed under the Apache License:

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at
    
        http://www.apache.org/licenses/LICENSE-2.0
    
    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.

This repository is based off code written by NVIDIA that is licensed under the Apache License, Version 2.0. In accordance with the Apache License, all files that are modifications of code originally written by NVIDIA maintain a NVIDIA copyright header. All files that do not contain such a header are original to EleutherAI contributors. When the NVIDIA code has been modified from its original version, that fact is noted in the copyright header. All derivative works of this repository must preserve these headers under the terms of the Apache License.

For full terms, see the `LICENSE` file. If you have any questions, comments, or concerns about licensing please email us at contact@eleuther.ai.

### Acknowledgements

We run our experiments on a Kubernetes cluster generously provided by [CoreWeave](https://coreweave.com/).
