# tfbert
- 基于tensorflow 1.x 的bert系列预训练模型工具
- 支持多GPU训练，支持梯度累积，支持pb模型导出，自动剔除adam参数
- 采用dataset 和 string handle配合，可以灵活训练、验证、测试，在训练阶段也可以使用验证集测试模型，并根据验证结果保存参数。


## 说明


config、tokenizer参考的transformers的实现。

内置有自定义的Trainer，像pytorch一样使用tensorflow1.14，具体使用下边会介绍。

目前内置 [文本分类](run_classifier.py)、[文本多标签分类](run_element_extract.py)、[命名实体识别](run_ner.py)例子。

内置的几个例子的数据处理代码都支持多进程处理，实现方式参考的transformers。

内置代码示例数据集[百度网盘提取码：rhxk](https://pan.baidu.com/s/1lYy7BJdadT0LJfMSsKz6AA)
## 支持模型

bert、electra、albert、nezha、wobert、ChineseBert（GlyceBert）

## requirements
```
tensorflow==1.x
tqdm
jieba
```
目前本项目都是在tensorflow 1.x下实现并测试的，最好使用1.14及以上版本，因为内部tf导包都是用的

import tensorflow.compat.v1 as tf

## **使用说明**
#### **Config 和 Tokenizer**
使用方法和transformers一样
```python
from tfbert import BertTokenizer, BertConfig

config = BertConfig.from_pretrained('config_path')
tokenizer = BertTokenizer.from_pretrained('vocab_path', do_lower_case=True)

inputs = tokenizer.encode_plus(
'测试样例', text_pair=None, max_length=128, padding="max_length", add_special_tokens=True)

config.save_pretrained("save_path")
tokenizer.save_pretrained("save_path")

```
多卡运行方式，需要设置环境变量CUDA_VISIBLE_DEVICES，内置trainer会读取参数：
```
CUDA_VISIBLE_DEVICES=1,2 python run.py
```
详情查看代码样例

## **XLA和混合精度训练训练速度测试**

使用哈工大的rbt3权重进行实验对比，数据为example中的文本分类数据集。
开启xla和混合精度后刚开始训练需要等待一段时间优化，所以第一轮会比较慢，
等开启后训练速度会加快很多。最大输入长度32，批次大小32，训练3个epoch，
测试环境为tensorflow1.14，GPU是2080ti。

| use_xla | mixed_precision | first epoch (s/epoch) | second epoch (s/epoch) | eval accuracy |
| :------: | :------: | :------: | :------: | :------: |
| False | False | 76 | 61 | 0.9570 |
| True | False | 73 | 42 | 0.9584 |
| True | True | 85 | 37 | 0.9582 |

开启混合精度比较慢，base版本模型的话需要一两分钟，但是开启后越到后边越快，训练步数少的话可以只开启xla就行了，如果多的话
最好xla和混合精度（混合精度前提是你的卡支持fp16）都打开。

## 可加载中文权重链接
| 模型简称 | 下载链接 |
| :------- | :--------- |
| **`BERT wwm 系列`** | **[Chinese-BERT-wwm](https://github.com/ymcui/Chinese-BERT-wwm)**|
| **`BERT-base, Chinese`<sup>Google</sup>** | [Google Cloud](https://storage.googleapis.com/bert_models/2018_11_03/chinese_L-12_H-768_A-12.zip) |
| **`ALBERT-base, Chinese`<sup>Google</sup>** | [google-research/albert](https://github.com/google-research/albert) |
| **`MacBERT, Chinese`**    | **[MacBERT](https://github.com/ymcui/MacBERT)**|
| **`ELECTRA, Chinese`**    | **[Chinese-ELECTRA](https://github.com/ymcui/Chinese-ELECTRA)**|
| **`ERNIE 1.0.1, Chinese`**    | **[百度网盘(xrku)](https://pan.baidu.com/s/13eRD6uVnr4xeUfYXk8XKIw)**|
| **`ERNIE gram base, Chinese`**    | **[百度网盘(7xet)](https://pan.baidu.com/s/1qzIuduI2ZRJDZSnNqTfscw)**|
| **`ChineseBert, Chinese`**    | **[base(sxhj)](https://pan.baidu.com/s/1ehO52PQd6TFVhOu5RiRtZA)** **[large(zi0r)](https://pan.baidu.com/s/1IifQuRFhpwWzLJHvMR9gOQ)**|


## **更新记录**
- 2021/7/31 内置模型新增香侬科技开源的ChineseBert，见[glyce_bert](tfbert/models/glyce_bert.py)，目前官方只有torch版本。
  模型增加了字形和拼音特征作为embedding表示，获得了和mac bert接近的效果，官方见[ChineseBert](https://github.com/ShannonAI/ChineseBert)
。tf权重已经转好，可自行下载。

- 2021/5/19 增加机器阅读理解示例代码，以dureader2021比赛数据为例，应该兼容大部分squad格式的数据。
  同时更新tokenizer代码，贴合transformers使用接口，大部分直接整合的transformers的tokenizer

- 2021/5/9 增加fgm，pgd，freelb接口，代码见[adversarial.py](tfbert/adversarial.py),
  使用方式，在trainer的build_model中传入adversarial_type即可，这两天没GPU和相应数据集，所以功能还没测试。

- 2021/4/18 花了一天时间重整Trainer，新增一个Dataset类。由于更新有点多，还没来得及写太多注释，敬请见谅。具体更新：
  1. trainer封装了train、evaluate、predict方法，具体见新版的使用例子。
  2. 写了一个Dataset类，支持简单的数据包装，也可以直接导出tf的dataset类型，具体[dataset.py](tfbert/data/dataset.py). 
  3. 去除了原版需要自定义shapes和types的方式（原有data代码还没删），都可以通过新增的Dataset类下的方法直接自行获取。
  

- 2021/4/17 新增SimpleTrainer，采用feed dict的方式进行调用，操作简单，但是相比Trainer的dataset方式要慢好多，
  随便写了个例子[simple_trainer.py](simple_trainer.py)，以后有时间再完善
- tf.layers.dropout 需要将training设置为None才会根据tf.keras.backend.learning_phase()进行mode判定。
  之前默认的training为False，dropout都没起作用，非常抱歉。
- 增加resize_word_embeddings方法，可对已保存权重文件的embedding部分就行词表大小修改。
  具体见[resize_word_embeddings方法](tfbert/utils.py)
- 对抗训练暂不可用...代码实现错误
- 2021年2月22日 增加FGM对抗训练方式，可以在trainer.build_model()时设置use_fgm为True，
  即可开启fgm对抗训练，目前未测试效果。

- 2021年2月8日  毕业论文写完了，花了点时间进行大更新，此次更新对原有代码重组，进一步提升训练速度。使用NVIDIA的方法修改梯度累积方式，去除backward、
  zero_grad方法，统一使用train_step训练。梯度累积只需要在配置优化节点时传入梯度累积步数即可。
  最后，代码增加xla加速和混合精度训练，混合精度目前只支持部分gpu，支持情况自行百度。
  最后详情请自行看使用例子对比。

- 2020年11月14日 增加xla加速模块，可以在trainer设定use_xla传参决定是否开启，开启后可以加速训练。backward、zero_grad、train_step模式增加开启关闭操作，
可以在trainer设定use_torch_mode决定是否取消该模式，取消后不支持梯度累积，直接调用train_step进行训练，
这样会加快训练速度。

- 2020年9月23日 增加梯度累积，采用trainer.backward(), trainer.zero_grad(), trainer.train_step() 一同进行训练，参考pytorch训练方式。
- 2020年9月21日 第一次上传，支持模型bert、albert、electra、nezha、wobert。

**Reference**  
1. [Transformers: State-of-the-art Natural Language Processing for TensorFlow 2.0 and PyTorch. ](https://github.com/huggingface/transformers)
2. [TensorFlow code and pre-trained models for BERT](https://github.com/google-research/bert)
3. [ALBERT: A Lite BERT for Self-supervised Learning of Language Representations](https://github.com/google-research/albert)
4. [NEZHA-TensorFlow](https://github.com/huawei-noah/Pretrained-Language-Model/tree/master/NEZHA-TensorFlow)
5. [ELECTRA: Pre-training Text Encoders as Discriminators Rather Than Generators](https://github.com/google-research/electra)
6. [基于词颗粒度的中文WoBERT](https://github.com/ZhuiyiTechnology/WoBERT)
7. [NVIDIA/BERT模型使用方案](https://github.com/NVIDIA/DeepLearningExamples/tree/master/TensorFlow/LanguageModeling/BERT)
