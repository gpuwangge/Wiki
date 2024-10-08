# stability.ai
stability.ai是推出Stable Diffusion的公司， 于2020年创办, 致力于像Open AI一样构建开源AI项目。  
Stable Diffusion是其首个开源产品。  
Midjourney是Stable Diffusion的主要竞争对手。OpenAI的DALLE也是竞品，但实力稍逊。  
Stability AI的大语言模型是StableLM。目前有3亿到7亿个参数可用(3B, 7B)，后续将提升到15亿到65亿个参数。  
StableLM和Stable Diffusion的代码托管在GitHub上。  
OpenAI由于受金主微软的影响，Open程度不如stability.ai。  
与DALL-E不同的是，stable.ai允许用户在没有监管的情况下使用和构建模型。用户可以使用该软件进行商业创作。  
2022年8月，stability.ai发布了DreamStudio应用程序，用户可以通过这个程序根据自然语言的描述创建图像。  
虽然Stable Diffusion是开源的，但如果要使用DreamStudio还是需要付费。  

# 安装Stable Diffusion
本章讨论如何在Windows环境下安装纯净Stable Diffusion WebUI，并且使用官方SDXL1.0生成图像。  
(https://github.com/AUTOMATIC1111/stable-diffusion-webui)  
WebUI是被越南人AUTOMATIC1111整合了web ui界面的Stable Diffusion版本。  
最低配置：显存至少8GB，本地磁盘15GB。另外准备几十GB用来保存模型。  
**`1.安装git`**  
**`2.安装python`**  
根据提示，必须Python 3.10.6  
checking "Add Python to PATH"  
https://www.python.org/downloads/release/python-3106/  
安装完成后，打开console，使用如下命令：  
```
python
```
如果显示Python 3.10.6，则说明安装成功了  
**`3.git clone这个库`**  
size约40mb  
**`4.运行webui-user.bat`**  
大概安装十五分钟吧  
会下载文件到这个repo文件夹里，最后size为9.37GB  
最后会出现文字:  
Running on local URL: http://127.0.0.1:7860  
并且会自动把这个页面弹到浏览器里  
Prompt里面写上Cat，点击Generate，会生成一张随机猫图  
默认的大模型是：  
stable-diffusion-webui\models\Stable-diffusion\v1-5-pruned-emaonly.safetensors  
这是2022年10月发布的官方1.5版本  
这样就表示安装成功了  
注意运行的时候webui.bat窗口不可以关闭  
以后再次运行webui-user.bat的时候，就只用24秒就打开  
**`5.SDXL模型安装(Optional)`**  
https://huggingface.co/stabilityai/stable-diffusion-xl-base-1.0/tree/main  
下载这个文件：  
sd_xl_base_1.0.safetensors (6.94 GB)  
扔到stable-diffusion-webui\models\Stable-diffusion\这个文件夹下  
在WebUI选择这个.safetensor文件  
同样用猫测试，如果可以生成就成功了  
SDXL可以生成1024x1024的大图片而不出现诡异的效果  

# Stable Diffusion模型种类介绍
定义：模型是经过训练学习后得到的程序文件  
模型中存储的不是一张张可视的原始图片，而是将图像特征解析后的代码，因此模型更像是一个储存了图片信息的大脑。  
Stable Diffusion的官方模型是耗费了几十万美元成本获得的模型，并且是免费开源的。然而，其效果很多时候并不好。原因是官方模型是一个通用模型。假如我们想生成一个头像，最好用头像专用模型。  
并且，现有的模型都是在官方模型的基础上，加入少量文本图像数据，并配合微调模型的训练方法获得的定制模型。这样训练成本大大降低。一般只需要一张消费级显卡数小时的工作。  
通过官方模型微调得到的主模型叫做Checkpoint(GB)。实际上，实践发现光有主模型还不够好。主模型就像一本主教材，还需要搭配其他的辅助教材。  
这些辅助教材也成为扩展模型。常见的扩展模型有Embeddings(KB), LoRA(MB), HyperNetworks(MB)和VAE(MB)。  

## Checkpoint
主模型，体积大，效果好，但不够灵活。常见训练方法叫做Dreambooth。官方训练一个Ckpt模型需要上万张图片，模型size通常为2GB, 4GB, 7GB等。但并不是模型越大越好。  
模型被存放在\models\Stable-diffusion文件夹中  

## Embeddings
是轻量级扩展模型。训练方法是Textual Inversion，对提示词部分进行操作。很多画手，脸部模型出错的话，可以用Embeddings模型来解决。  
模型被存放在\embeddings文件夹中  

## LoRA(Low-Rank Adaptation Models)
是最热门的扩展模型，常用于固定角色特征。如果把主模型比喻成一本书，Embeddings就是便利贴，LoRA就像是夹页。  
保存位置是\models\Lora  

## HyperNetworks
是低配LoRA，不如LoRA流行。  

## VAE
补充VAE功能。有时候生成的图像发灰，就是因为主模型的VAE文件损坏。  
这时候就可以使用WebUI提供的外置VAE选项来使用未损坏的VAE文件。  
存放位置是\models\VAE  
因为VAE是辅助大模型用的，可以将大模型对应的VAE修改为同样的名字，这样切换Checkpoint的时候VAE也会自动切换。  

## 模型的功能类型  
1、固定对象特征：比如生成一个Jinx，训练通常简单  
2、固定图像风格：比如生成一个二次元卡通图片，训练难度中等  
3、概念艺术表达：比如生成一个赛博朋克场景，训练难度较高  

## 模型后缀  
.ckpt后缀的文件即可能是Checkpoint模型，也可能是LoRA模型或VAE模型。  
.safetensors后缀的模型是为了解决可能被攻击的程序漏洞，速度也更快。尽量选这个。  
这些后缀只表达保存数据的格式，内部数据没有太大区别，可以通过工具转换  

## 如何判断优秀模型
出图结果准确，没有乱加细节，图像正常，文件健康  
有的模型虽然出图精美，但是千篇一律，风格被固定死，即使加上了LoRA也无法改变人物特征  

# SDXL模型风格示例
Stable DiffusionXL(SDXL) 1.0 是2023年7月官方新出的大模型。模型开源，免费商用。是世界上最大参数级的开放绘图模型。1.0基础版使用35亿参数，精修版使用66亿参数。  
并且，不同于以往的512~768尺寸分辨率，SDXL使用1024x1024的分辨率，保证了我们生成这个分辨率的图片不再会产生多人多头的问题了。  
另外，提示词功能也更加全面，只需要少量词汇就可以生成精美图片了。也支持更多风格。  
有一些其他大模型不需要任何LoRA，也能产生各种画风：GhostMix, Deliberate, ReV, DreamShaper  

## 固定图像风格
txt2img, Width=1024, Height=1024  
necromancer, anime  
necromancer, photographic  
necromancer, digital art  
<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/anime_necromancer.png" alt="alt text" width="150" height="150">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/photographic_necromancer.png" alt="alt text" width="150" height="150">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/digitalart_necromancer.png" alt="alt text" width="150" height="150">  
</p>  

necromancer, Comic Book  
necromancer, Fantasy Art  
necromancer, Analog Film  
<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/comicbook_necromancer.png" alt="alt text" width="150" height="150">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/fantasyart_necromancer.png" alt="alt text" width="150" height="150">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/analogfilm_necromancer.png" alt="alt text" width="150" height="150">  
</p>  

necromancer, Neon Punk  
necromancer, Isometric  
necromancer, Low Poly  
<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/neonpunk_necromancer.png" alt="alt text" width="150" height="150">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/isometric_necromancer.png" alt="alt text" width="150" height="150">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/lowpoly_necromancer.png" alt="alt text" width="150" height="150">  
</p>  

necromancer, Origami  
necromancer, Line Art  
necromancer, Cinematic  
<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/origami_necromancer.png" alt="alt text" width="150" height="150">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/lineart_necromancer.png" alt="alt text" width="150" height="150">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/cinematic_necromancer.png" alt="alt text" width="150" height="150">  
</p>  

necromancer, 3D Model  
necromancer, Pixel Art    
<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/3dmodel_necromancer.png" alt="alt text" width="150" height="150">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/pixelart_necromancer.png" alt="alt text" width="150" height="150">  
</p>  

## 固定对象特征
本章讨论SDXL对于生成一致性虚拟角色的效果。所谓虚拟，是指凭空创造的，现实中不存在的角色。所谓一致性，是指角色的风格、特征具有一致性，使得人类可以一眼辨认出现在不同画面的角色是同一个角色。  
### Seed
Stable Diffusion在生成图片的时候，使用Seed作为随机数。当创建一致性角色的时候，我们希望每张图片的Seed是一样的。这样，当我们生成了一个满意角色的时候，需要将当前图片的Seed记录下来以备复用。  
如果当时没有记下来，也可以在WebUI的图片信息里找到这个数字。  
如果Seed=-1，则代表没有手动设定Seed，生成图片的时候将采用随机Seed。  
差异Seed：这是一个可选属性。差异Seed提供了额外一个参数，它与Seed共同决定图片生成的效果。差异强度也可以设置，靠近强度0表示效果接近Seed，靠近强度1表示效果接近差异Seed。  
### 虚拟人名
在Promp里给生成的角色随便取一个名字，就比较容易生成长相相似的角色。  
原理猜测：Stable Diffusion会在模型库里寻找跟这个名字对应的人脸。如果该名子是一个名人，那么大概率就会抽取这个名人的脸来做模特。  
如果是一个不存在的虚拟名字，实际上是找不到对应的人脸的。但是会根据一定规律生成一张脸。  
只要模型没有变更，生成的脸大概率是比较一致的。  
还有一种做法，就是找感兴趣的不同的明星名字拼起来，后面放一个系数，这样可以生成由名人脸组合的新的虚拟脸，效果也是比较一致的。  
```
[Taylor Swift : Gal Gadot : 0.5]
```

### 实例
锁定Seed=2771310497, Batch size=6  
```
1girl, [uxia], necromancer, fantasy art, upper body, studio light, medieval age cloth, black hair, brown eyes, ((battlefield background)), blurry background
Negative prompt: helmet
Steps: 30, Sampler: Euler a, Schedule type: Automatic, CFG scale: 7, Seed: 2771310497, Size: 512x768, Model hash: 31e35c80fc, Model: sd_xl_base_1.0, Version: v1.9.4
```

<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/uxia1.png" alt="alt text" width="150" height="225">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/uxia2.png" alt="alt text" width="150" height="225">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/uxia3.png" alt="alt text" width="150" height="225">  
</p> 

<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/uxia4.png" alt="alt text" width="150" height="225">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/uxia5.png" alt="alt text" width="150" height="225">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/uxia6.png" alt="alt text" width="150" height="225">  
</p> 

通过调整提示词可以创建不同场景，服饰和道具的角色。  

<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/SD3.gif" alt="alt text" width="225" height="225">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/SD1.gif" alt="alt text" width="225" height="225">  
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/SD2.gif" alt="alt text" width="225" height="225">  
</p> 


如果要修改默认UI设置可以更改ui-config.json这个文件  

# 图生图 (img2img)
如果发现如下错误：  
**A tensor with all NaNs was produced in Unet**  
用记事本打开webui-user.bat这个文件，补齐如下参数：  
**set COMMANDLINE_ARGS= --medvram --xformers --no-half**  

## 局部重绘 (Inpaint)
局部重绘是图生图的一个子功能。如果对直接绘图的结果基本满意，但是其中有不理想的部分，可以使用这个功能。  
点击img2img, 点击inpaint，上传图片，点击图片右上角的"Use Brush"按钮，就可以涂抹要修改的区域。  
比较重要的参数有
Inpaint area: 如果选Only masked，是生成一个包含Mask区域的矩形方框进行绘制，然后仅替换Mask区域。这个模式生成快，如果重绘区域比较独立可以选这个。    
如果选Whole picture，则把全图画一遍，再替换掉Mask区域。  
Denoising Strengh: 默认是0.75。这个值越接近1，重绘结果离原始图的差别就越大。  
重绘的时候，最好在原图的Prompt的基础上稍作修改一下，不然容易生成风格完全不同的补丁。  
(也要记住把inpaint的size也调整成原本的大小)  



# Reference
https://foresightnews.pro/article/detail/18576  
https://www.yuque.com/a-chao/sd/nxcwfkw7vmfw8dz6  
https://www.uisdc.com/stable-diffusion-guide-4  
https://m.paoka.com/info/1059  
https://indienova.com/u/aigc/blogread/33638  
https://huke88.com/article/8093.html
https://blog.csdn.net/FL1623863129/article/details/130922349  
https://www.uisdc.com/stable-diffusion-guide-3



