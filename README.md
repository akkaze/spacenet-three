![Architecture](satellites.jpg)

**More stuff from us**
- [Telegram](https://t.me/snakers4) 
- [Twitter](https://twitter.com/AlexanderVeysov)
- [Blog](https://spark-in.me/tag/data-science)

# 1 Hardware requirements

**Training**

- 6+ core modern CPU (Xeon, i7) for fast image pre-processing;
- The models were trained on 2 * GeForce 1080 Ti;
- Training time on my setup ~ **3 hours** for models with 8-bit images as inputs;
- Disk space - 40GB should be more than enough;

**Inference**

- 6+ core modern CPU (Xeon, i7) for fast image pre-processing;
- On 2 * GeForce 1080 Ti inference takes **3-5 minutes**;
- Graph creation takes **5-10 minutes**;

# 2 Preparing and launching the Docker environment

**Clone the repository**

`git clone https://github.com/snakers4/spacenet-three .`


**This repository contains 2 Dockerfiles**
- `/dockerfiles/Dockerfile` - this is the main Dockerfile which was used as environment to run the training and inference scripts. This one also includes the majority of the tested legact spacenet libraries used by their APLS repository
- `/dockerfiles/Dockerfile2`- this is an additional backup Docker file with newer versions of the drivers and PyTorch, just in case. Does not contain spacenet libraries

**Build a Docker image**

`
cd dockerfiles
docker build -t aveysov .
`

**Install the latest nvidia docker**

Follow instructions from [here](https://github.com/NVIDIA/nvidia-docker).
Please prefer nvidia-docker2 for more stable performance.


To test all works fine run:


`docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi`

**(IMPORTANT) Run docker container (IMPORTANT)**

Unless you use this exact command (with --shm-size flag) (you can change ports and mounted volumes, of course), then the PyTorch generators **WILL NOT WORK**. 


- nvidia-docker 2: `docker run --runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all -it -v /path/to/cloned/repository:/home/keras/notebook -p 8888:8888 -p 6006:6006  --shm-size 8G aveysov`
- nvidia-docker: `nvidia-docker -it -v /path/to/cloned/repository:/home/keras/notebook -p 8888:8888 -p 6006:6006  --shm-size 8G aveysov`

**Installing project specific software**


0. Already done for `/dockerfiles/Dockerfile`. You can do it yourself for `/dockerfiles/Dockerfile2`
1. Exec into the docker machine via `docker exec -it --user root YOUR_CONTAINER_ID /bin/bash`
2. Run these scripts one after another
```
conda install -y -c conda-forge cython
conda install -y -c conda-forge rasterio
conda install -y -c conda-forge libgdal
conda install -y -c conda-forge gdal
conda install -y -c conda-forge scikit-image
conda install -y -c conda-forge pyproj
conda install -y -c conda-forge geopandas
conda install -y -c conda-forge tqdm
conda install -y -c conda-forge shapely=1.5.16
conda install -y -c conda-forge scipy
conda install -y -c conda-forge networkx=1.11
conda install -y -c conda-forge fiona
pip3 install utm
pip3 install osmnx==0.5.1
```
3. Run these scripts one after another
```
pip3 install numba
conda install -y -c conda-forge scikit-image
```
4. Sometimes `apt install libgl1-mesa-glx` is necessary due to some Docker [quirks](https://github.com/ContinuumIO/docker-images/issues/49) 

Steps 2-3 are to ensure compatibility with legacy software from APLS [repository](https://github.com/CosmiQ/apls).
An alternative to that - is to use pip's requirements.txt in the same order.
These steps are required to run 8-bit mask creation step from APLS repository and mask creation step.
If you will be trying to re-do this step - reserve 5-6 hours for experiments.

**To start the stopped container**


`docker start -i YOUR_CONTAINER_ID`


# 3 Preparing the data and the machine for running scripts using APLS code

- Dowload the data into `data/`
- Data Download [guide](https://drive.google.com/open?id=1WJh8Q1Oj38Ahn0FiwUv1WZXoIwnY08_kL4XO8cikPo0) from the authors
- Run these commands to create 8-bit images, mask and test 8-bit images:
```
docker exec -it YOUR_CONTAINER_ID sh -c "python3 create_binary_masks.py && \
cd src && \
python3 create_8bit_test_images.py && \
"
```
    
After all of your manipulations your directory should look like:

```
├── README.md          <- The top-level README for developers using this project.
├── data
│   ├── AOI_2_Vegas_Roads_Test_Public         <- Test set for each city
│   └── AOI_2_Vegas_Roads_Train               <- Train set for each city
│       ├─ geojson
│       ├─ summaryData
│       ├─ MUL
│       ├─ RGB-PanSharpen
│       ├─ PAN
│       ├─ MUL-PanSharpen
│       ├─ MUL-PanSharpen_8bit
│       ├─ RGB-PanSharpen_8bit
│       ├─ PAN_8bit
│       ├─ MUL_8bit
│       ├─ MUL-PanSharpen_mask
│       ├─ RGB-PanSharpen_mask
│       ├─ PAN_mask
│       ├─ MUL_mask
│       └─ RGB-PanSharpen_mask
│   │
│   ...
│   │
│   ├── AOI_5_Khartoum_Roads_Test_Public      <- Test set for each city
│   └── AOI_5_Khartoum_Roads_Train            <- Train set for each city
│
├── dockerfiles                               <- A folder with Dockerfiles
│
├── src                                       <- Source code
│
└── scripts                                   <- One-off preparation scripts
```

# 4 Training the model from scratch

If all is ok, then use the following command to train the model

- optional - turn on tensorboard for monitoring progress `tensorboard --logdir='satellites_roads/src/tb_logs' --port=6006` via jupyter notebook console or via tmux + docker exec (model converges in 30-40 epochs)
- then
```
docker exec -it YOUR_CONTAINER_ID sh -c "echo 'python3 train_satellites.py \
--arch linknet34 --batch-size 6 \
--imsize 1280 --preset mul_ps_vegetation --augs True \
--workers 6 --epochs 40 --start-epoch 0 \
--seed 42 --print-freq 20 \
--lr 1e-3 --optimizer adam \
--tensorboard True --lognumber ln34_mul_ps_vegetation_aug_dice' > train.sh && \
sh train.sh""
```

# 5 Predicting masks

- You can load the [pretrained weights](https://drive.google.com/open?id=19Zd4RG0P0YwwRZ_xgip7dAmb4Bp8WqTI) to the `src/weights` folder for the above definition of the model
- Run this script
``` 
docker exec -it YOUR_CONTAINER_ID sh -c "cd src && \
echo 'python3 train_satellites.py \
--arch linknet34 --batch-size 12 \
--imsize 1312 --preset mul_ps_vegetation --augs True \
--workers 6 --epochs 50 --start-epoch 0 \
--seed 42 --print-freq 10 \
--lr 1e-3 --optimizer adam \
--lognumber norm_ln34_mul_ps_vegetation_aug_dice_predict \
--predict --resume weights/norm_ln34_mul_ps_vegetation_aug_dice_best.pth.tar\' > predict.sh && \
sh predict.sh
```



# 6 Creating graphs and submission files


Execute this script:
```
docker exec -it YOUR_CONTAINER_ID sh -c "cd src && \
python3 final_model_lstrs.py --folder norm_ln34_mul_ps_vegetation_aug_dice_predict"
```


`folder` argument is for masks containing folder name, default is `norm_ln34_mul_ps_vegetation_aug_dice_predict`.
Scipt saves a file called `norm_test.csv` into `../solutions` directory. 

The resulting file is used then as a submission file.



# 7 Additional notes

- You can run training and inference on the presets from `/src/presets.py`;
- So the model can be evaluated on RGB-PS images and / or 8-channel images as well;
- This script, for example will train an 8-channel model:
```
python3 train_satellites.py \
	--arch linknet34 --batch-size 6 \
	--imsize 1280 --preset mul_ps_vegetation --augs True \
	--workers 6 --epochs 40 --start-epoch 0 \
	--seed 42 --print-freq 20 \
	--lr 1e-3 --optimizer adam \
	--tensorboard True --lognumber ln34_mul_ps_vegetation_aug_dice
```

To train an 8-channel model you should also replace mean and std settings in the `src/SatellitesAugs.py`

```
# 8-channel settings
# normalize = transforms.Normalize(mean=[0.5, 0.5, 0.5, 0.5, 0.5, 0.5, 0.5, 0.5],
#                                 std=[1, 1, 1, 1, 1, 1, 1, 1])

# 3 channel settings
normalize = transforms.Normalize(mean=[0.485, 0.456, 0.406],
                                 std=[0.229, 0.224, 0.225])
```

- 16-bit images are also supported:

This snippet is commented in `src/SatellitesAugs.py`

```
# version compatible with 16-bit images
"""  
class ImgAugAugs(object):
    def __call__(self,
                 image):
        global seed        
        
        # poor man's flipping
        if seed%2==0:
            image = np.fliplr(image)
        elif seed%4==0:
            image = np.fliplr(image)
            image = np.flipud(image)
        
        # poor man's affine transformations
        image = rotate(image,
                     angle=seed,
                     resize=False,
                     clip=True,
                     preserve_range=True)        

        return image
"""

But be careful with normalization (`div(255)`), as 16-bit images actually are 11-bit (divide by 2048).

```

- Also the following models are supported
    - `unet11` (VGG11 + Unet)
    - `linknet50` (ResNet50 + LinkNet, 3 layers)
    - `linknet50_full` (ResNet50 + LinkNet, 4 layers)
    - `linknext` (ResNext-101-32 + LinkNet, 4 layers)
    
- Also in the repo you can find scripts to generate wide masks (i.e. wide roads have varying width) and layered masks (paved / non-paved). There are scripts in the `src/SatellitesDataset.py` that support that. They basically just replace some paths; 
    
# 8 Juputer notebooks

Use these notebooks on your own risk!

- `src/experiments.ipynb` - general debugging notebook with new models / generators / etc
- `src/play_w_stuff.ipynb` - visualizing the solutions
- `src/pipeline_experiments.ipynb`- some minor experiments with the graph creation script
