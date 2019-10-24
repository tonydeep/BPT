# Segment Tree Transformer
This repo contains the code for our paper

[Segment Tree Transformer]()

The code is written in [DGL](https://github.com/dmlc/dgl) with PyTorch as backend.

## Requirements

- torchtext 0.4
- dgl 0.4
- yaml
- spacy
- PyTorch 1.1+

## Usage

For Multi-GPU training, please `export NCCL_LL_THRESHOLD=0` before running scripts because of a PyTorch bug mentioned [here](https://github.com/pytorch/pytorch/issues/20630).

The codebase has two dependencies: `graph_kernel` and `graph_builder`, the first one is for efficient graph attention on GPU with node parallel strategy written in CUDA, the second one is for efficient graph construction written in Cython. To install them:
```
cd graph_builder
python setup.py install
cd ..
cd graph_kernel
python setup.py install
cd ..
``` 

We support the following tasks with STT as backbone:
- Text Classification: `text_classification.py`
- Language Modeling: `lm.py`
- Machine Translation: `mt.py`
- Natural Language Inference: `nli.py`

All experiment settings mentioned in our paper are available at `configs/`.

```
python *.py --config configs/*.yml --gpu [GPUs]
```

Note that this repo does not contain any data files, to get dataset required for experiments, run `. get_*.sh` and the corresponding dataset would be downloaded and preprocessed.

For machine translation, we have another script `mt_infer.py` for decoding:
```
python mt_infer.py --config configs/*.yml --gpu [GPU]
``` 

Before decoding, please make sure you have finished the training using `mt.py` with the same config file.

**NOTE**:
Currently we do not support CPU training/inference.

## Results

- Character-Level Language Modeling (enwik8, metric: bpc), 12 layer
    - STT(context length=8192): 1.02
    - Adaptive Transformer: 1.02
    - Transformer-XL: 1.06
    - To reproduce: `python lm.py --config configs/enwik8-8192.yml --gpu 0,1,2,3,4,5,6,7`
- Document-Level Machine Translation (IWSLT 2015 Zh-En, metric: BLEU), base setting.
    - STT(context length=64): 19.84
    - HAN-NMT: 17.68
    - To reproduce: `python mt.py --config configs/iwslt-4-64.yml --gpu 0`
- Sentiment Analysis (IMDB, metric: accuracy), 5 layer.
    - STT+GloVe: 92.12(±0.11)
    - LSTM+CoVe: 91.8
    - Transformer+Glove: 89.24(±0.20)
    - Star Transformer: 90.50
    - To reproduce: `python text_classification.py --config configs/imdb-4.yml --gpu 0`
        - Note that our CUDA kernel uses atomic operations which may result in non-determinism, we report the mean and std of accuracy in multiple(10) runs.
        - The IMDB dataset has not official train/dev split, we follow the setting of [Bryan et al., 2017](https://arxiv.org/pdf/1708.00107.pdf) and hold out 10% samples for validation. We report the test accuracy of model with best valid loss.

For sentence level modeling, we show that STT models better inductive bias than vanilla transformer by attending fine-grained features of neighbors and coarse-grained features of far-away tokens.
- Machine Translation(WMT14 En-De, metric: BLEU), base setting.
    - STT(k=1): 26.9
    - STT(k=2): 27.4
    - STT(k=4): 27.6
    - STT(k=8): 26.7
    - Transformer-base(our implementation): 27.2
    - To reproduce: `python mt.py --config configs/wmt-*.yml --gpu 0,1,2,3,4,5,6,7`
        - We report [SacreBLEU](https://github.com/mjpost/sacreBLEU) result for reproducibility (setting: `BLEU+c.mixed+l.en-de+#.1+s.exp+t.wmt14+tok.intl+v.1.4.1`), the sacrebleu score is usually lower than that produced by `get_ende_bleu.sh` script in tensor2tensor as described [here](https://github.com/tensorflow/tensor2tensor/issues/317). 
- Natural Language Inference(SNLI, metric: accuracy)
    - STT(k=4): 88.25(±0.07)
    - Transformer: 87.89(±0.31)
    - To reproduce: `python nli.py --config configs/snli.yml --gpu 0` 
        - Like Text Classification, the result on NLI is also not stable because of randomness in our CUDA kernel, we report the mean and std of accuracy in multiple(7) runs.

## TODOs

- FP16 support (mixed-precision training/inference)
- Integrate kernels with dgl 0.5
- CPU support
