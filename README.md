# TacotronV2 + WaveRNN
1. 开源中文语音数据集[标贝](https://www.data-baker.com/open_source.html)(女声)训练中文[TacotronV2](https://github.com/Rayhane-mamah/Tacotron-2)，实现中文到声学特征(Mel)转换的声学模型。在GTA模式下，利用训练好的TacotronV2合成标贝语音数据集中中文对应的Mel特征，作为声码器[WaveRNN](https://github.com/fatchord/WaveRNN)的训练数据。在合成阶段，利用TactornV2和WaveRNN合成高质量、高自然度的中文语音。
2. 从[THCHS-30](http://www.openslr.org/18/)任选一个speaker的语音数据集，微调TacotronV2中的部分参数，实现说话人转换。
3. Tensorflow serving + Tornado 部署TacotronV2中文语音合成服务。   

由于[TacotronV2](https://github.com/Rayhane-mamah/Tacotron-2)中采用Location sensitive attention，对长句字的建模能力不好(漏读、重复)，尝试了[GMM attention](https://github.com/syang1993/gst-tacotron/blob/master/models/gmm_attention_wrapper.py)、[Discrete Graves Attention](https://github.com/mozilla/TTS/blob/master/layers/common_layers.py#L113)[issue](https://github.com/mozilla/TTS/issues/346)、[Forward attention](https://github.com/mozilla/TTS/blob/master/layers/common_layers.py#L193)，能有效地解决对长句的建模能力，加快模型收敛速度。

------------------------------------

## 1 训练中文Tacotron V2

### 1.1 训练数据集预处理
#### 1.1.1 中文标点符号处理
对于中文标点符号，只保留'，。？！'四种符号，其余符号按照相应规则转换到这四个符号之一。

#### 1.1.2 中文到拼音转换
利用字拼音文件和词拼音文件，实现中文到拼音转换，能有效消除多音字的干扰。具体步骤如下：       
1. 对于每个句子中汉字从左到右的顺序，优先从词拼音库中查找是否存在以该汉字开头的词并检查该汉字后面的汉字是否与该词匹配，若满足条件，直接从词库中获取拼音，若不满足条件，从字拼音库中获取该汉字的拼音。     
2. 对于数字(整数、小数)、ip地址等，首先根据规则转化成文字，比如整数2345转化为二千三百四十五，再转化为拼音。
3. 由于输入是文字转化而来的拼音序列，所以在合成阶段，允许部分或全部的拼音输入。    


### 1.2 训练数据集预处理
执行如下脚本，生成TacotronV2的训练数据集
> python preprocess.py

### 1.2 训练TacotronV2模型
执行如下脚本，训练TacotronV2模型
> python train.py --model='Tacotron-2'

### 1.3 合成语音
修改脚本 generate_mel_by_gta_for_tested_text.py 中text，执行如下脚本，合成语音
> python generate_mel_by_gta_for_tested_text.py

### 1.4 改进部分
* 由于[TacotornV2]()中采用的注意力机制是Location sensitive attention，对长句子的建模能力不太好，分别尝试了以下三种注意力机制：
    * 采用GMM attention(三种形式)
    * Discretized Graves attention(GMM attention的变种)
    * Forward attention

> 同时，由于语音合成中的音素(拼音)到声学参数(Mel频谱)是从左到右的单调递增的对应关系，在Inference阶段，对attention中的alignments的处理能够进一步提高模型对长句子的语音合成效果，在Location sensitive attention和Forward attention中还能达到控制语速的效果。



## 2 训练WaveRNN模型
### 2.1 训练数据集准备
利用训练好的TacotronV2对标贝语音数据集在GTA(global teacher alignment)模式下，生成对应的Mel特征。需要注意的如下：
1. TacotronV2中的mel输出的范围为[-hparmas.max_abs_value, hparams.max_abs_value]，而WaveRNN中的mel的范围[0, 1]，故需要将TacotronV2输出mel特征变为[0, 1]范围内。
2. TacotronV2中的hop_size为256，需要将WaveRNN中的voc_upsample_factors的值改为(4, 8, 8)(注上采样的比例, 或者(x, y, z)，并且x * y * z = hop_size)。


## 3 speaker adaptive
参照代码[TactronV2](https://github.com/Rayhane-mamah/Tacotron-2)支持finetune，在finetune阶段，固定decoder层前的所有层的参数(embedding层、CHBG、encoder层等)，用少量的新数据集训练从checkpoint中恢复的模型，达到speaker adpative的目的。

## 4 服务部署
采用Tensorflow Serving + Docker 来部署训练好的TacotronV2语音服务，由于需要对文本进行处理，还搭建了Flask后台框架，最终的语音合成的请求过程如下：       
请求过程：页面 -> Flask后台 -> Tensorflow serving    
响应过程：Tensorflow serving -> Flask后台 -> 页面



## 额外参照文献
- [location-relative attention mechanisms for robust long-form speech synthesis](https://arxiv.org/pdf/1910.10288)
