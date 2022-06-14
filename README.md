# [Matryoshka Representations for Adaptive Deployment](https://arxiv.org/abs/2205.13147)
Aditya Kusupati, Gantavya Bhatt, Aniket Rege, Matthew Wallingford, Aditya Sinha, Vivek Ramanujan, William Howard-Snyder, Kaifeng Chen, Sham Kakade, Prateek Jain, Ali Farhadi

This repository contains the code for training and evaluating Matryoshka Representation with CNN backbone, along with visualizations. Training code is built upon [FFCV](https://github.com/libffcv/ffcv-imagenet) modified for MRL. The repository is organized as follows:

1. Set up
2. Matryoshka Linear Layer
3. Training ResNet50 Models
4. Evaluating Trained Models
5. Evaluating pre-trained checkpoints
5. Retrieval Performance

Lastly, we also provide code to generate the figures mentioned in our paper. Please have a look at thee `Visualization/` folder. 
## Set Up
First pip install the requirements file in this directory:
```
pip install -r requirements.txt
```
We use PyTorch Distributed Data Parallel shared over 2 A100 GPUs and FFCV for the dataloader. Please make sure to scale the learning rate according to the #GPUs used. 

### Preparing the dataset.

Generate ImageNet FFCV dataset from standard imagenet train set with the following command (`IMAGENET_DIR` should point to a PyTorch style [ImageNet dataset](https://github.com/MadryLab/pytorch-imagenet-dataset)):

```bash
# Required environmental variables for the script:
export IMAGENET_DIR=/path/to/pytorch/format/imagenet/directory/
export WRITE_DIR=/your/path/here/

# Serialize images with:
# - 500px side length maximum
# - 50% JPEG encoded, 90% raw pixel values
# - quality=90 JPEGs
.train_and_eval/train/write_imagenet.sh 500 0.50 90
```
## Matryoshka Linear Layer
`MRL.py` provides MRL linear layer, and it can be instantiated very easily using the following
```
nesting_list=[8, 16, 32, 64, 128, 256, 512, 1024, 2048]
fc_layer = MRL_Linear_Layer(nesting_list, num_classes=1000, efficient=efficient)
```
Where, `nesting_list` is the list of dimensions we want to nest on, `num_classes` is the number of output feature and efficient flag if we want to do MRL-E.

## Training ResNet50 models
First of all change the directory to `train_and_eval/train`. While training, we dump  model ckpts, training logs and run command inside a folder. 

### Training Fixed Feature Baseline

```bash 
export CUDA_VISIBLE_DEVICES=0,1,2,3

python train_imagenet.py --config-file rn50_configs/rn50_40_epochs.yaml --model.fixed_feature=2048 --data.train_dataset=/home/jupyter/dataset-ffcv/train_500_0.50_90.ffcv --data.val_dataset=/home/jupyter/dataset-ffcv/uncompressed/val_500_uncompressed.ffcv --data.num_workers=12 --data.in_memory=1 --logging.folder=trainlogs --logging.log_level=1 --dist.world_size=2 --training.distributed=1 --lr.lr=0.425
```

### Training MRL model

```bash 
export CUDA_VISIBLE_DEVICES=0,1,2,3

python train_imagenet.py --config-file rn50_configs/rn50_40_epochs.yaml --model.mrl=1 --data.train_dataset=/home/jupyter/dataset-ffcv/train_500_0.50_90.ffcv --data.val_dataset=/home/jupyter/dataset-ffcv/uncompressed/val_500_uncompressed.ffcv --data.num_workers=12 --data.in_memory=1 --logging.folder=trainlogs --logging.log_level=1 --dist.world_size=2 --training.distributed=1 --lr.lr=0.425
```

By default, we start nesting from 8 dimensions (That is 2^3). In case we want to nest representations from 16 dimensions, add an additional flag of 
```
--model.nesting_start=4
```

### Training MRL-E model

```bash 
export CUDA_VISIBLE_DEVICES=0,1,2,3,4

python train_imagenet.py --config-file rn50_configs/rn50_40_epochs.yaml --model.efficient=1 --data.train_dataset=/home/jupyter/dataset-ffcv/train_500_0.50_90.ffcv --data.val_dataset=/home/jupyter/dataset-ffcv/uncompressed/val_500_uncompressed.ffcv --data.num_workers=12 --data.in_memory=1 --logging.folder=trainlogs --logging.log_level=1 --dist.world_size=2 --training.distributed=1 --lr.lr=0.425
```

By default, we start nesting from 8 dimensions (That is 2^3). In case we want to nest representations from 16 dimensions, add an additional flag of 
```
--model.nesting_start=4
```
## Evaluating Trained Models
First of all change the directory to `train_and_eval/eval`. We have two different scripts to evaluate the Fixed Feature (FF) baseline and MRL models, respectively.

### Evaluating FF baseline
In paper we only consider fixed `nesting_list=[8, 16, 32, 64, 128, 256, 512, 1024, 2048]`, therefore we hardcode evaluating on those FF mdoels such that the name for each folder with ckpt is `sh=0_mh=0_nesting_start=3_fixed_feature={dim}/` Please change the name accordingly in while evaluating your models. Furthermore, make sure that launch directory has validation sets corresponding to each dataset (V1/V2/A/R/Sketch). To evaluate, run the following command. 

```
python eval_ff.py --path [path to ckpt folders] --dataset [V2/A/Sketch/R/V1/O]
```
If `dataset` flag is not passed, it will evaluate on standard ImageNet validation set. 

### Evaluating MRL models 
```
python eval_MRL.py --path [path to weight checkpoint, need to be .pt file] --dataset [V2/A/Sketch/R/V1/O] (--efficient)
```

## Evaluating pre-trained checkpoints
We also provide path to the saved checkpoints which were used to generate the results in our paper. However, the MRL Linear layer had a slightly different re-factored [code](eval_ckpts/NestingLayer.py), therefore we have separate scripts to directly evaluate them. Saved model ckpts can be found [here](https://drive.google.com/drive/folders/1IEfJk4xp-sPEKvKn6eKAUzvoRV8ho2vq?usp=sharing). 

Change the directory to `eval_ckpts/`. Here in addition to the scripts, there is a folder of `Notebooks/`, which contains visualization such as GradCAM images (for checkpoint models), superclass performance, model cascades and oracle upper bound.  

### Evaluating MRL models 
```
python eval_MRL.py --path [path to weight checkpoint, need to be .pt file] --dataset [V2/A/Sketch/R/V1/O] (--efficient)
```

### Evaluating FF models 
```
python eval_ff.py --path [path to ckpt folders] --dataset [V2/A/Sketch/R/V1/O]
```
If `dataset` flag is not passed, it will evaluate on standard ImageNet validation set. 

### Evaluating SVD Baseline 
We consider low rank approximation of standard ResNet50 model. By default, it loads the folder `sh=0_mh=0_nesting_start=3_fixed_feature=2048/final_weights.pt` stored in location `PATH`. 

```
python svd.py --path PATH --dataset [V2/A/Sketch/R/V1/O]
```

### Evaluating Slimmable Baseline
We consider two slimmable baselines based on this [repository](https://github.com/JiahuiYu/slimmable_networks). First is slimmable network, and other is autoslim. Please download the corresponding checkpoints from the mentioned repository. 

#### Slimmable Network 
```
python eval_slimmable_network.py --weight_path [PATH_TO_WEIGHT_PT_FILE] --config_path [PATH TO s_resnet50_train_val.yml] --dataset [V2/A/Sketch/R/V1/O]
```

#### AutoSlim Network 
```
python eval_slimmable_network.py --weight_path [PATH_TO_WEIGHT_PT_FILE] --config_path [PATH TO autoslim_resnet_train_val.yml] --dataset [V2/A/Sketch/R/V1/O] --autoslim
```

## Retrieval Performance


## Citation
If you find this project useful in your research, please consider citing:
```
@article{Kusupati2022MatryoshkaRF,
  title={Matryoshka Representations for Adaptive Deployment},
  author={Aditya Kusupati and Gantavya Bhatt and Aniket Rege and Matthew Wallingford and Aditya Sinha and Vivek Ramanujan and William Howard-Snyder and Kaifeng Chen and Sham M. Kakade and Prateek Jain and Ali Farhadi},
  journal={ArXiv},
  year={2022},
  volume={abs/2205.13147}
}
```
