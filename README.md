# Rotate-Captcha-Crack

[中文](README_zh-cn.md) | English

Predict the rotation angle of given picture through CNN. This project can be used for rotate-captcha cracking.

Test result:

![test_result](https://user-images.githubusercontent.com/48282276/224320691-a8eefd23-392b-4580-a729-7869fa237eaa.png)

Three kinds of model are implemented, as shown in the table below.

| Name        | Backbone          | Cross-Domain Loss (less is better) | Params  | FLOPs  |
| ----------- | ----------------- | ---------------------------------- | ------- | ------ |
| RotNet      | ResNet50          | 71.7920°                           | 24.246M | 4.132G |
| RotNetR     | RegNetY 3.2GFLOPs | 19.1594°                           | 18.468M | 3.223G |
| RCCNet_v0_5 | RegNetY 3.2GFLOPs | 42.7774°                           | 17.923M | 3.223G |

RotNet is the implementation of [`d4nst/RotNet`](https://github.com/d4nst/RotNet/blob/master/train/train_street_view.py) over PyTorch. `RotNetR` is based on `RotNet`, with `RegNet` as its backbone and class number of 180. The average prediction error is `19.1594°`, obtained by 64 epochs of training (2 hours) on the [Google Street View](https://www.crcv.ucf.edu/data/GMCP_Geolocalization/) dataset.

The Cross-Domain Test uses [Google Street View](https://www.crcv.ucf.edu/data/GMCP_Geolocalization/) and [Landscape-Dataset](https://github.com/yuweiming70/Landscape-Dataset) for training, and Captcha Pictures from Baidu (thanks to @xiangbei1997) for testing.

The captcha picture used in the demo above comes from [RotateCaptchaBreak](https://github.com/chencchen/RotateCaptchaBreak/tree/master/data/baiduCaptcha)

## Try it!

### Prepare

+ Device supporting CUDA10+ (mem>=4G for training)

+ Python>=3.8,<3.12

+ PyTorch>=1.11

+ Clone the repository and install all requiring dependencies

```shell
git clone --depth=1 https://github.com/Starry-OvO/rotate-captcha-crack.git
cd ./rotate-captcha-crack
pip install .
```

**DO NOT** miss the `.` after `install`

+ Or, if you prefer `venv`

```shell
git clone --depth=1 https://github.com/Starry-OvO/rotate-captcha-crack.git
python -m venv ./rotate-captcha-crack --system-site-packages
cd ./rotate-captcha-crack
# Choose the proper script to acivate venv according to your shell type. e.g. `./Script/active*`
python -m pip install -U pip
pip install .
```

### Download the Pretrained Models

Download the `*.zip` files in [Release](https://github.com/Starry-OvO/rotate-captcha-crack/releases) and unzip them all to the `./models` dir.

The directory structure will be like `./models/RCCNet_v0_5/230228_20_07_25_000/best.pth`

The names of models will change frequently as the project is still in beta status. So, if any `FileNotFoundError` occurs, please try to rollback to the corresponding tag first.

### Test the Rotation Effect by a Single Captcha Picture

If no GUI is presented, try to change the debugging behavior from showing images to saving them.

```bash
python test_captcha.py
```

### Use HTTP Server

+ Install extra dependencies

```shell
pip install aiohttp httpx[cli]
```

+ Launch server
  
```shell
python server.py
```

+ Another Shell to Send Images

```shell
 httpx -m POST http://127.0.0.1:4396 -f img ./test.jpg
```

## Train Your Own Model

### Prepare Datasets

+ For this project I'm using [Google Street View](https://www.crcv.ucf.edu/data/GMCP_Geolocalization/) and [Landscape-Dataset](https://github.com/yuweiming70/Landscape-Dataset) for training. You can collect some photos and leave them in one directory. Without any size or shape requirement.

+ Modify the `dataset_root` variable in `train.py`, let it points to the directory containing images.

+ No manual labeling is required. All the cropping, rotation and resizing will be done soon after the image is loaded.

### Train

```bash
python train_RotNetR.py
```

### Validate the Model on Test Set

```bash
python test_RotNetR.py
```

## Details of Design

Most of the rotate-captcha cracking methods are based on [`d4nst/RotNet`](https://github.com/d4nst/RotNet), with `ResNet50` as its backbone. `RotNet` treat the angle prediction as a classification task with 360 classes, then use `CrossEntropy` to compute the loss.

Yet `CrossEntropy` will bring a significant metric distance of about $358°$ between $1°$ and $359°$, clearly defies common sense, it should be a small value like $2°$. Meanwhile, the [`angle_error_regression`](https://github.com/d4nst/RotNet/blob/a56ea59818bbdd76d4dd8d83b8bbbaae6a802310/utils.py#L30-L36) proposed by [d4nst/RotNet](https://github.com/d4nst/RotNet) is less effective. That's because when dealing with outliers, the gradient leads to a non-convergence result. You can easily understand this through the subsequent comparison between loss functions.

My regression loss function `RotationLoss` is based on `MSELoss`, with an extra cosine-correction to decrease the metric distance between $±k*360°$.

$$ \mathcal{L}(dist) = {dist}^{2} + \lambda_{cos} (1 - \cos(2\pi*{dist})) $$

Why `MSELoss` here? Because the `label` generated by 
self-supervised method is guaranteed not to contain any outliers. So our design does not need to consider the outliers. Also, `MSELoss` won't break the derivability of loss function.

The loss function is derivable and *almost* convex over the entire $\mathbb{R}$. Why say *almost*? Because there will be local minimum at $predict = \pm 1$ when $\lambda_{cos} \gt 0.25$.

Finally, let's take a look at the figure of two loss functions:

<p align="center">

![loss](https://github.com/Starry-OvO/rotate-captcha-crack/assets/48282276/1dd9e0b4-e40d-4205-8500-14cf27e187dd)

</p>
