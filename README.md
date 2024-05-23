<div align="center">
<h1>
  星辰语音大模型-超多方言ASR
</h1>
</div>

<p align="center">
🤗 <a href="https://huggingface.co/Tele-AI/TeleSpeech-ASR1.0" target="_blank">Hugging Face</a> • 🐾 <a href="https://gitee.com/Tele-AI/TeleSpeech-ASR" target="_blank">gitee</a>️
</p>

# 目录
- [目录](#目录)
- [模型开源](#模型开源)
- [环境配置](#环境配置)
  - [微调](#微调)
  - [表征训练下游任务](#表征训练下游任务)
- [数据准备](#数据准备)
  - [特征提取](#特征提取)
  - [字典准备](#字典准备)
- [预训练模型微调](#预训练模型微调)
  - [微调](#微调-1)
  - [解码](#解码)
- [表征训练下游任务](#表征训练下游任务-1)
- [开源数据集结果](#开源数据集结果)
- [声明与协议](#声明与协议)
  - [声明](#声明)
  - [协议](#协议)

# 模型开源

星辰超多方言语音识别大模型v1.0，由30w小时无标注多方言语音数据进行训练，打破单一模型只能识别特定单一方言的困境，可支持理解粤语、上海话、四川话、温州话等30多种方言


本次发布版本和下载链接见下表

| 模型版本             | 参数量  | 下载链接     |
|---------------------|-------|---------------------|
| pretrain_base | 0.09 B | [TeleSpeech-ASR1.0-base](https://huggingface.co/Tele-AI/TeleSpeech-ASR1.0/blob/main/base.pt)  |
| pretrain_large | 0.3 B | [TeleSpeech-ASR1.0-large](https://huggingface.co/Tele-AI/TeleSpeech-ASR1.0/blob/main/large.pt)  |


# 环境配置
环境依赖
* PyTorch version >= 1.13.0
* Python version >= 3.8
* 数据准备、程序训练需要使用kaldi，请确保已正确安装：https://github.com/kaldi-asr/kaldi
  * 若已有提好的特征，程序运行时可以使用wenet开源框架中kaldi_io.py实现的方法替换kaldiio.load_mat，从而无需安装kaldi

## 微调

<a id="fairseq安装"></a>
* 安装fairseq及其依赖
```shell script
$ git clone https://github.com/pytorch/fairseq
$ cd fairseq
$ pip install --editable ./
```

* 安装kaldiio
```shell script
$ pip install kaldiio
```

## 表征训练下游任务

* 确保fairseq已正确[安装](#fairseq安装)

* 安装表征训练任务运行所需依赖
```shell script
$ cd wenet_representation
$ pip install -r requirements.txt
```

# 数据准备
## 特征提取

* 利用kaldi提取40维mfcc特征，参数设置参考`mfcc_hires.conf`
* 为各数据集准备训练用文件`data.list`，以`\t`分隔：
```
$ cat train/data.list
utt:X0000000000_100638174_S00037	feat:/data/raw_nnaudio.test.1.ark:2983479385	feat_shape:363,40	text:不惜在这种试验中毁灭包括自己在内的一切	token:不 惜 在 这 种 试 验 中 毁 灭 包 括 自 己 在 内 的 一 切	tokenid:[TOKENID]	token_shape:19,5537
utt:X0000000001_100849618_S00006	feat:/data/raw_nnaudio.test.1.ark:2984296665	feat_shape:345,40	text:在他们收到足够建立大统一模型的数据后	token:在 他 们 收 到 足 够 建 立 大 统 一 模 型 的 数 据 后	tokenid:[TOKENID]	token_shape:18,5537
...
```

## 字典准备

* 微调阶段，需要准备fairseq格式的 `dict.${label}.txt`，`${label}`为建模单元类型，如ltr, bpe等。以`dict.ltr.txt`为例：
```
是 2
好 3
...
```

* 预训练模型表征训练ASR任务阶段，需要准备wenet格式的`lang_char.txt`，相比于`dict.${label}.txt`额外添加`<blank>`, `<unk>`, `<sos/eos>`3个token，例如
```
<blank> 0
<unk> 1
是 2
好 3
...
<sos/eos> 5536
```

# 预训练模型微调

## 微调
* 准备`train.tsv`和`dev.tsv`，保存于同一训练目录下
    ```
    $ ln -s /path/to/train/data.list /path/to/train/train.tsv
    $ ln -s /path/to/dev/data.list /path/to/train/dev.tsv
    ```
* 进入data2vec_dialect路径，修改`path.sh`文件中`/path/to/fairseq`为fairseq安装路径
* 将`run_scripts/run_d2v_finetune.sh`中`/path/to/fairseq`和`/path/to/data2vec_dialect`路径替换
* 修改`task.data`为`.tsv`保存路径，如`task.data=/data/wenetspeech/train`
* 执行
    ```shell script
    $ bash run_scripts/run_d2v_finetune.sh
    ```

## 解码
* 同样修改`run_scripts/decode.sh`中的模型路径、测试数据路径等
  * `dataset.gen_subset`为测试数据路径下`tsv`文件的名称，可配置多个
* 执行
    ```shell script
    $ bash run_scripts/decode.sh
    ```

# 表征训练下游任务

* 进入wenet_representation路径，修改`path.sh`文件中`fairseq`, `data2vec_dialect`, `wenet_representation`相关路径

* 连续表征训练与解码：
  * 配置`run_d2v.sh`中dataset相关内容，执行
    ```shell script
    $ bash run_d2v.sh
    ```

* 离散表征训练与解码：
  * 首先根据`data.list`，准备离散表征对应训练文件`data.list.discrete`，修改`wenet/discrete_token/kmeans_d2v.yaml`中`model_dir`和`user_dir`，执行
    ```
    $ bash wenet/discrete_token/dump_feat.sh
    ```
  * 再配置`run_discrete.sh`中dataset相关内容，执行
    ```
    $ bash run_discrete.sh
    ```
# 开源数据集结果
* 我们选择了多个开源中文数据集进行验证，以测试集上的字错误率 (Character Error Rate, CER) 结果作为衡量标准
* 在Aishell-1上我们选择其Train集作为有监督数据进行训练，在Test集上统计CER
* 在WenetSpeech上，我们分别使用100小时训练集Train_s和1000小时训练集Train_m分别作为有监督数据进行训练，在Test_Meeting测试集上统计CER
* Babel为NIST（美国国家标准与技术研究院）举办的低资源粤语电话识别任务数据集，我们使用其提供的训练集与测试集统计CER
* KeSpeech为中文多方言测试集，我们使用1396小时训练集作为有监督数据进行训练，选择提供的Test测试集统计CER

|  模型版本         | Aishell-1 | WenetSpeech*| Babel | KeSpeech |
| ----------| -------- | ------- | ---- | ---- |
| pretrain_base | 4.7  | 18.3 / 16.4 | 22.1  | 10.9 |
| pretrain_large | 4.0 | 14.3 / 13.0 | 19.1  | 8.1 |

*WenetSpeech中的结果为分别使用 `train_s/train_m`训练后，在Test_Meeting上的CER

# 声明与协议
## 声明
我们在此声明，不要使用TeleSpeech模型及其衍生模型进行任何危害国家社会安全或违法的活动。同时，我们也要求使用者不要将TeleSpeech模型用于没有安全审查和备案的互联网服务。我们希望所有使用者遵守上述原则，确保科技发展在合法合规的环境下进行。

我们已经尽我们所能，来确保模型训练过程中使用的数据的合规性。然而，尽管我们已经做出了巨大的努力，但由于模型和数据的复杂性，仍有可能存在一些无法预见的问题。因此，如果由于使用TeleSpeech开源模型而导致的任何问题，包括但不限于数据安全问题、公共舆论风险，或模型被误导、滥用、传播或不当利用所带来的任何风险和问题，我们将不承担任何责任。

## 协议
社区使用TeleSpeech模型需要遵循《[TeleSpeech模型社区许可协议](./TeleSpeech模型社区许可协议.pdf)》。TeleSpeech模型支持商业用途，如果您计划将TeleSpeech模型或其衍生品用于商业目的，您需要通过以下联系邮箱 tele_ai@chinatelecom.cn，提交《TeleSpeech模型社区许可协议》要求的申请材料。审核通过后，将特此授予您一个非排他性、全球性、不可转让、不可再许可、可撤销的商用版权许可。

---