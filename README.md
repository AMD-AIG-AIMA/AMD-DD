# AMD Diffusion Distillation

This repository provides training recipes for the AMD Nitro models, a series of efficient text-to-image generation models that are distilled from popular diffusion models on AMD Instinct GPUs.

Compared to the [Stable Diffusion 2.1 base model](https://huggingface.co/stabilityai/stable-diffusion-2-1-base), we achieve 95.9% reduction in FLOPs at the cost of just 2.5% lower CLIP score and 2.2% higher FID.

| Model    | FID &darr; | CLIP &uarr; |FLOPs| Latency on AMD Instinct MI250 (sec)
| :---: | :---: | :---: | :---: | :---:
| Stable Diffusion 2.1 base, 50 steps (cfg=7.5) | 25.47   | 0.3286 |83.04 | 4.94
| **Stable Diffusion 2.1 Nitro**, 1 step | 26.04     | 0.3204|3.36 | 0.18

Compared to [PixArt-Sigma](https://pixart-alpha.github.io/PixArt-sigma-project/), our high resolution model achieves a 90.9% reduction in FLOPs at the cost of just 3.7% lower CLIP score and 10.5% higher FID.

| Model    | FID &darr; | CLIP &uarr; |FLOPs| Latency on AMD Instinct MI250 (sec)
| :---: | :---: | :---: | :---: | :---:
| PixArt-Sigma, 20 steps | 34.14   | 0.3289 |187.96 | 7.46
| **PixArt-Sigma Nitro**, 1 step | 37.75     | 0.3167|17.04 | 0.53


## Environment

### Docker image
Pull the following docker image from [docker hub](https://hub.docker.com/r/rocm/pytorch)

``` 
docker pull rocm/pytorch:rocm6.1.3_ubuntu22.04_py3.10_pytorch_release-2.1.2 
```

### Dependencies
Install the core python libraries by

```
pip install diffusers==0.29.2 transformers accelerate wandb torchmetrics pycocotools torchmetrics[image] open-clip-torch
```

## Synthetic data generation

Our models are distilled using synthetic data generated from the base models using prompts from [DiffusionDB](https://huggingface.co/datasets/poloclub/diffusiondb). Follow the instructions in their repo to extract prompts from the dataset and prepare a .txt file where each line corresponds to a prompt. 

We provide a sample list ```data/sample_prompts.txt``` as an example.

#### Generating data from Stable Diffusion 2.1 base
```
bash scripts/run_gen_data.sh
```

#### Generating data from PixArt-Sigma
```
bash scripts/run_gen_data_pixart.sh
```

Please remember to correctly set **"PROMPT_PATH"** and **"OUT_FOLDER"** in the scripts.


## Train models
Use the following bash script to perform distillation:
```
bash scripts/run_train.sh
```

You will need to set:
* `MODEL_NAME`: the base model from which you want to distill an efficient model
* `DATA_ROOT`: the data folder that was generated in the previous step
* Huggingface Accelerate parameters according to your training setup to use the correct number of GPUs and batchsize. You may refer to [Accelerate CLI](https://huggingface.co/docs/accelerate/en/package_reference/cli) for more details.


## Generate images
The distilled models generated by the training script are saved in Diffusers format. Use the following code snippets to perform inference with them:

**Stable Diffusion 2.1 Nitro**
```python
from diffusers import DDPMScheduler, DiffusionPipeline
import torch

scheduler = DDPMScheduler.from_pretrained("stabilityai/stable-diffusion-2-1-base", subfolder="scheduler")
pipe = DiffusionPipeline.from_pretrained("stabilityai/stable-diffusion-2-1-base", scheduler=scheduler)

ckpt_path = '<path to distilled checkpoint>'
unet_state_dict = torch.load(ckpt_path)
pipe.unet.load_state_dict(unet_state_dict)
pipe = pipe.to("cuda")

image = pipe(prompt='a photo of an astronaut riding a horse on mars',
             num_inference_steps=1,
             guidance_scale=0,
             timesteps=[999]).images[0]
```

**PixArt-Sigma Nitro**
```python
from diffusers import PixArtSigmaPipeline
import torch

pipe = PixArtSigmaPipeline.from_pretrained("PixArt-alpha/PixArt-Sigma-XL-2-1024-MS")

ckpt_path = '<path to distilled checkpoint>'
transformer_state_dict = torch.load(ckpt_path)
pipe.transformer.load_state_dict(unet_state_dict)
pipe = pipe.to("cuda")

image = pipe(prompt='a photo of an astronaut riding a horse on mars',
             num_inference_steps=1,
             guidance_scale=0,
             timesteps=[400]).images[0]
```

## Evaluation

**COCO dataset**

Download COCO val2017 images from [here](http://images.cocodataset.org/zips/val2017.zip) and annotations from [here](http://images.cocodataset.org/annotations/annotations_trainval2017.zip)

Create a root folder called `coco` and unzip these two files into this folder. The folder structure looks like:

```
coco/
├── val2017/
└── annotations/

```
To evaluate the model, run:
```
bash scripts/run_eval.sh
```

Please correctly set variables in this script including `COCO_ROOT`, `CKPT_PATH`, `MODEL`, etc. The script will generate 5k images based on 5k unique given prompts from the COCO val2017 dataset, and calculate FID and CLIP scores based on these generated images.


## License

Copyright (c) 2024 Advanced Micro Devices, Inc. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
