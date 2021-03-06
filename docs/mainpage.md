# Merlin: HugeCTR # {#mainpage}

HugeCTR is a recommender specific framework which is capable of distributed training across multiple GPUs and nodes for Click-Through-Rate (CTR) estimation.
It is also a component of [**NVIDIA Merlin**](https://developer.nvidia.com/nvidia-merlin#getstarted),
which is a framework accelerating the entire pipeline from data ingestion and training to deploying GPU-accelerated recommender systems.

Design Goals:
* Fast: it's a speed-of-light CTR training framework;
* Dedicated: we consider everything you need in CTR training;
* Easy: you can start your work now, no matter you are a data scientist, a learner, or a developer.

## Version 2.2
In HugeCTR version 2.2, we add the new features like *full fp16 pipeline*, *algorithm search*, *AUC calculation*, etc, 
whilst enabling the support of the world's most advanced accelerator, NVIDIA A100 Tensor Core GPU and the modern models such as *Wide and Deep*, *Deep Cross Network*, *DeepFM*, *Deep Learning Recommendation Model (DLRM)*.
This document describes how to setup envirnment and run HugeCTR.

## Requirements ##
* cuBLAS >= 10.1
* Compute Capability >= 70 (V100)
* CMake >= 3.8
* cuDNN >= 7.5
* NCCL >= 2.0
* Clang-Format 3.8
* GCC >= 7.4.0
* ortools
### Optional, if require multi-nodes training ###
* OpenMPI >= 4.0
* UCX library >= 1.6
* HWLOC library >= 2.1.0
* mpi4py

## Build ##
### Init Git ###
Under the home directory of HugeCTR:
```shell
$ git submodule update --init --recursive
```

### Use Docker Container ###
You can choose to use Docker to simplify the environment setting up.
If you have already set up your envirnment manually, please jump to Build below.

Mare sure that you have installed [**Nvidia Docker**](https://github.com/NVIDIA/nvidia-docker) .

To build a docker image of **development environment** from the corresponding Dockerfile, run the command below.
It will install the libraries and tools required to use HugeCTR.
HugeCTR build itself must be done by yourself.
```shell
$ docker build -t hugectr:devel -f ./tools/dockerfiles/dev.Dockerfile .
```
Run with interaction mode (mount the home directory of repo into container for easy development and try):
```shell
$ docker run --runtime=nvidia --rm -it -u $(id -u):$(id -g) -v $(pwd):/hugectr -w /hugectr hugectr:devel bash
```

To build a docker image of **production environment**, run the command below.
In addition to resolving dependencies, it will build and install HugeCTR to `/usr/local/hugectr`.
Note that `SM` (the target GPU architecture list) and `NCCL_A2A` (use NCCL ALL-to-ALL or not) are also specified.
You can change them according to your environment.
```shell
$ docker build --build-arg SM="70;75;80" \
               --build-arg NCCL_A2A=on \
               -t hugectr:build \
               -f ./tools/dockerfiles/build.Dockerfile .
```
Run HugeCTR directly
```shell
$ docker run --runtime=nvidia hugectr:build huge_ctr --version
```

### Build with Release ###
Compute Capability can be specified by `-DSM=[Compute Compatibilities]`, which is SM70 by default (Tesla V100). It is also possible to set multiple Compute Capabilities, e.g., `-DSM=70` for Telsa V100 and `-DSM="70;75"` for both Telsa V100 and Telsa T4.
```shell
$ mkdir -p build
$ cd build
$ cmake -DCMAKE_BUILD_TYPE=Release -DSM=70 .. # Target is Tesla V100
$ make -j
```

Supported Compatibility and Tesla GPUs:

|Compute Compatibility|GPU|
|----|----|
|70|NVIDIA V100 (Volta)|
|75|NVIDIA T4 (Turing)|
|80|NVIDA A100 (Ampere)|

### Build with Debug ###
If build type is `Debug`, HugeCTR will print more verbose logs and execute GPU tasks in a synchronous manner.
The other options remain the same as `Release` build.
```shell
$ mkdir -p build
$ cd build
$ cmake -DCMAKE_BUILD_TYPE=Debug -DSM=70 .. # Target is Telsa V100
$ make -j
```

### Build with Validation Mode ###
This mode is designed for framework validation. In this mode loss of trainig will be shown as the average of `eval_batches` results. Only one thread and chunk will be used in DataReader. Performance will be lower than turning off.
```shell
$ mkdir -p build
$ cd build
$ cmake -DVAL_MODE=ON ..
$ make -j
```

### Build with Multi-Nodes Training Supported ###
To run with multi-nodes please build in this way and run HugeCTR with `mpirun`. For more details plese refer to `samples/dcn2nodes`
```shell
$ mkdir -p build
$ cd build
$ cmake -DENABLE_MULTINODES=ON ..
$ make -j
```

### Build with NCCL All2All Supported ###
The default collection communication library used in LocalizedSlotSparseEmbedding is [Gossip](https://github.com/Funatiq/gossip). [NCCL all2all](https://github.com/NVIDIA/nccl/tree/p2p) is also supported in HugeCTR. If you want to run with NCCL all2all, please turn on the NCCL_A2A switch in cmake. 
```shell
$ mkdir -p build
$ cd build
$ cmake -DNCCL_A2A=ON ..
$ make -j
```

## Run ##
Please refer to samples

## Coding Style and Refactor ##
Default coding style follows Google C++ coding style [(link)](https://google.github.io/styleguide/cppguide.html).
This project also uses `Clang-Format`[(link)](https://clang.llvm.org/docs/ClangFormat.html) to help developers to fix style issue, such as indent, number of spaces to tab, etc.
The Clang-Format is a tool that can auto-refactor source code.
Use following instructions to install and enable Clang-Format:
### Install ###
```shell
$ sudo apt-get install clang-format
```
### Run ###
```shell
# First, configure Cmake as usual 
$ mkdir -p build
$ cd build
$ cmake -DCLANGFORMAT=ON ..
# Second, run Clang-Format
$ cmake --build . --target clangformat
# Third, check what Clang-Format did modify
$ git status
# or
$ git diff
```

## Document Generation ##
Doxygen is supported in HugeCTR and by default on-line documentation browser (in HTML) and an off-line reference manual (in LaTeX) can be generated within `docs/`.
### Install ###
[Download doxygen](http://www.doxygen.nl/download.html)
### Generation ###
Within project `home` directory
```shell
$ doxygen
```

## Synthetic Data Generation and Benchmark ##
For quick benchmarking and research use, you can generate a synthetic dataset like below. Without any additional modification to JSON file. Both Norm format (with Header) and Raw format (without Header) dataset can be generated with `data_generator`.
- For `Norm` format: <br>
```bash
$ ./data_generator your_config.json data_folder vocabulary_size max_nnz #files #samples per file
$ ./huge_ctr --train your_config.json
```
- For `Raw` format: <br>
```bash
$ ./data_generator your_config.json
$ ./huge_ctr --train your_config.json
```

Arguments:
+ `data_folder`: You have to specify the folder to store the generated data
+ `vocabulary_size`: Vocabulary size of your target dataset
+ `max_nnz`: [1, max_nnz] values will be generated for each feature (slot) in the dataset. **Note** that `max_nnz*slot_num` should be less than `max_feature_num` in your data layer.
+ `#files`: number of data file will be generated (optional)
+ `#samples per file`: number of samples per file (optional)

## File Format ##
In total, there are three types of files used in HugeCTR training: a configuration file, model file and dataset.

### Configuration File ###
Configuration file must be in a json format, e.g., simple_sparse_embedding_fp32.json.
There are three main JSON objects in a configuration file: "solver", "optimizer", and "layers". They can be specified in any order.
* solver: the active GPU list, batchsize, model_file, etc are specified.
* optimizer: The type of optimizer and its hyperparameters are specified.
* layers: training/evaluation data (and their paths), embeddings and dense layers are specified. Note that embeddings must precede the dense layers.

### Model File ###
Model file is a binary file that will be loaded to initialize weights.
In that file, the weights are stored in the same order with the layers in configuration file. 

We provide a tutorial on how to dump a model to TensorFlow, i.e., tutorial/dump_to_tf/readMe.md. You can find more details on `Model file and it format` there.

### Data Set ###
Two format of data set are supported in HugeCTR:
#### Norm ####
Norm format consists a collection of data files and a file list.

The first line of a file list is the number of data files in the dataset.
It is followed by the paths to those files.
```shell
$ cat simple_sparse_embedding_file_list.txt
10
./simple_sparse_embedding/simple_sparse_embedding0.data
./simple_sparse_embedding/simple_sparse_embedding1.data
./simple_sparse_embedding/simple_sparse_embedding2.data
./simple_sparse_embedding/simple_sparse_embedding3.data
./simple_sparse_embedding/simple_sparse_embedding4.data
./simple_sparse_embedding/simple_sparse_embedding5.data
./simple_sparse_embedding/simple_sparse_embedding6.data
./simple_sparse_embedding/simple_sparse_embedding7.data
./simple_sparse_embedding/simple_sparse_embedding8.data
./simple_sparse_embedding/simple_sparse_embedding9.data
```

A data file (binary) consists of a header and actual tabular data.

Header Definition:
```c
typedef struct DataSetHeader_ {
  long long error_check;        // 0: no error check; 1: check_num
  long long number_of_records;  // the number of samples in this data file
  long long label_dim;          // dimension of label
  long long dense_dim;          // dimension of dense feature
  long long slot_num;           // slot_num for each embedding
  long long reserved[3];        // reserved for future use
} DataSetHeader;

```

Data Definition (each sample):
```c
typedef struct Data_{
  int length;                   // bytes in this sample (optional: only in check_sum mode )
  float label[label_dim];       
  float dense[dense_dim];
  Slot slots[slot_num];          
  char checkbits;                // checkbit for this sample (optional: only in checksum mode)
} Data;

typedef struct Slot_{
  int nnz;
  unsigned int*  keys; // changeable to `long long` with `"input_key_type"` in `solver` object of JSON config file.
} Slot;
```

#### RAW ####
RAW format is introduced in HugeCTR v2.2

Different to `Norm` format, The training Data in `RAW` format is all in one binary file and in int32, no matter Label / Dense Feature / Category Features.

The number of Samples / Dense feature / Category feature / label dimension are all delcared in configure json file.

Note that only one-hot data is accepted with this format.

Data Definition (each sample):
```c
typedef struct Data_{
  int label[label_dim];       
  int dense[dense_dim];
  int category[sparse_dim];
} Data;
```


