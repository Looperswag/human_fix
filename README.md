<p align="center">
  <h2 align="center">RealisHuman：一种用于细化生成图像中畸形人体部位的两阶段方法</h2>
  <p align="center">

  
<table align="center">
  <tr>
    <td>
      <img src="assets/fig1_00.png"><br>
    </td>
  </tr>
</table>


## 🏃‍♂️ Getting Started
To begin, download the pretrained base models for [RV-5-1](https://huggingface.co/stablediffusionapi/realistic-vision-v51/tree/main), [DINOv2](https://huggingface.co/facebook/dinov2-base/tree/main), [StableDiffusion V1.5](https://huggingface.co/benjamin-paine/stable-diffusion-v1-5), and [StableDiffusion Inpainting](https://huggingface.co/benjamin-paine/stable-diffusion-v1-5-inpainting).

Next, your can download our RealisHuman checkpoints in [Baidu Cloud](https://pan.baidu.com/s/1aqbN3hsjxjn7zPKFoEChIg?pwd=8t2q) or [Google Drive Part1](https://drive.google.com/drive/folders/1N_6aZ3MBR2kbrz9F8Jgi65UPlIQ3j49r?usp=drive_link), [Google Drive Part2](https://drive.google.com/drive/folders/1UzGaGKTJZu9-oS9yzXAgjUh9-EfLXYss?usp=drive_link).

Organize the base models and checkpoints as follows:
```bash
mkdir checkpoint && mkdir pretrained_models

.
|-- LICENSE
|-- README.md
|-- assets
|-- data
|-- submodules
|   |-- 3DDFA-V3
|   |-- DWPose
|   `-- hamer-main
|-- realishuman
|-- configs
|-- checkpoint
|   |-- stage1_face
|   |   `-- checkpoint-stage1-face.ckpt
|   |-- stage1_hand
|   |   `-- checkpoint-stage1-hand.ckpt
|   |-- stage2_face
|   |   `-- checkpoint-stage2-face.ckpt
|   `-- stage2_hand
|       `-- checkpoint-stage2-hand.ckpt
|-- pretrained_models
|   |-- DINO
|   |   `-- dinov2
|   |-- RV
|   |   `-- rv-5-1
|   `-- StableDiffusion
|       |-- sd-1-5
|       `-- stable-diffusion-inpainting


```
## ⚒️ Installation

You can install the required environment using conda:
```bash
conda env create -f environment.yaml
conda activate RealisHuman
```
or with `pip`:
```bash
pip3 install -r requirements.txt
```
Additionally, you will need to set up environments for [DWPose](https://github.com/IDEA-Research/DWPose), [HaMeR](https://github.com/geopavlakos/hamer) and [3DDFAv3](https://github.com/wang-zidu/3DDFA-V3). Please refer to their official setup guides for detailed configuration steps.

# 🚀 Training and Inference 

## Data Preparation
Structure your data directory as follows:
```
data
|-- images
|   |-- 3ddfa
|   |-- dwpose
|   |-- hamer
|   |-- image
|   `-- results
```

Use the following command to extract DWPose data:

```shell
cd submodules/DWPose
conda activate {YOUR_DWPose_Environment}
python ControlNet-v1-1-nightly/dwpose_infer_example.py --input_path {PATH_TO_IMAGE_DIR}/image --output_path {PATH_TO_SAVE_PKL}/dwpose
```

To refine generated images with malformed hands, estimate the hand meshes using HaMeR:
```shell
cd submodules/hamer-main
conda activate {YOUR_HaMeR_Environment}
python demo_image.py --img_folder {PATH_TO_IMAGE_DIR}/image --out_folder {PATH_TO_SAVE_HAMER}/hamer --full_frame
```
In case you encounter the error "AttributeError: 'NoneType' object has no attribute 'glGetError'", try the following:
```shell
apt-get install -y python-opengl libosmesa6
```

If you want to refine generated images with malformed faces, estimate the face meshes using 3DDFAv3:
```shell
cd submodules/3DDFA-V3
conda activate {YOUR_3DDFAv3_Environment}
python demo_dir.py --inputpath {PATH_TO_IMAGE_DIR}/image --savepath {PATH_TO_SAVE_3DDFA}/3ddfa --device cuda --iscrop 1 --detector retinaface --ldm68 0 --ldm106 0 --ldm106_2d 0 --ldm134 0 --seg_visible 0 --seg 0 --useTex 0 --extractTex 0 --backbone resnet50
```


## Inference of RealisHuman

### 1. Hand Refining

#### Stage-One Pre-processing

To pre-process the hand data for stage-one, run the following command:

```bash
python data/process_hand_stage1.py
```

#### Stage-One Inference

After pre-processing, run the model to obtain the stage-one results:

```bash
CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7 torchrun --nnodes=1 --nproc_per_node=8 \
    inference_stage1.py --config configs/stage1-hand.yaml --output data/hand_example/hand_chip/repair \
    --ckpt checkpoint/stage1_hand/checkpoint-stage1-hand.ckpt
```

#### Stage-Two Processing and Inference

For stage-two, pre-process the hand data:

```bash
python data/process_hand_stage2.py
```

Then, run the model to obtain the stage-two results:

```bash
CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7 torchrun --nnodes=1 --nproc_per_node=8 \
    inference_stage2.py --config configs/stage2-hand.yaml --output data/hand_example/hand_chip/inpaint \
    --ckpt checkpoint/stage2_hand/checkpoint-stage2-hand.ckpt
```

#### Final Image Refinement

To paste the refined hand image back, execute:

```bash
python data/back_to_image_hand.py
```
Then, your can find the refined results in data/hand_example/hand_chip/results.

---

### 2. Face Refining

#### Stage-One Pre-processing

To pre-process the face data for stage-one, use the command:

```bash
python data/process_face_stage1.py
```

#### Stage-One Inference

Run the model to get the stage-one face results:

```bash
CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7 torchrun --nnodes=1 --nproc_per_node=8 \
    inference_stage1.py --config configs/stage1-face.yaml --output data/face_example/face_chip/repair \
    --ckpt checkpoint/stage1_face/checkpoint-stage1-face.ckpt
```

#### Stage-Two Processing and Inference

For stage-two, pre-process the face data:

```bash
python data/process_face_stage2.py
```

Run the model to get the stage-two results:

```bash
CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7 torchrun --nnodes=1 --nproc_per_node=8 \
    inference_stage2.py --config configs/stage2-face.yaml --output data/face_example/face_chip/inpaint \
    --ckpt checkpoint/stage2_face/checkpoint-stage2-face.ckpt
```

#### Final Image Refinement

To paste the refined face image back, run the following command:
```bash
python data/back_to_image_face.py
```
If you wish to integrate the refined results for faces and hands, run the following command:
```bash
python data/back_to_image_face.py --sub_dir results_hand
```
Then, your can find the refined results in data/face_example/face_chip/results.


## Train of RealisHuman
You also can train the model with your own data with the following command:
```shell
CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7  torchrun --nnodes=1 --nproc_per_node=8 \
    train_stage1.py --config configs/stage1-xxx.yaml

CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7  torchrun --nnodes=1 --nproc_per_node=8 \
    train_stage2.py --config configs/stage2-xxx.yaml 
```

## 🙏 Acknowledgements
We would like to thank the [Animatediff](https://github.com/guoyww/AnimateDiff) and [AnimateAnyone]() teams for their awesome codebases.

## Citation
```BibTeX
@misc{wang2024realishumantwostageapproachrefining,
      title={RealisHuman: A Two-Stage Approach for Refining Malformed Human Parts in Generated Images}, 
      author={Benzhi Wang and Jingkai Zhou and Jingqi Bai and Yang Yang and Weihua Chen and Fan Wang and Zhen Lei},
      year={2024},
      eprint={2409.03644},
      archivePrefix={arXiv},
      primaryClass={cs.CV},
      url={https://arxiv.org/abs/2409.03644}, 
}
