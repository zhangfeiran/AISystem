<!--Copyright © 适用于[License](https://github.com/chenzomi12/AISystem)版权许可-->

# 模型转换流程

用户在使用 AI 框架时，可能会遇到训练环境和部署环境不匹配的情况，比如用户用 Caffe 训练好了一个图像识别的模型，但是生产环境是使用 TensorFlow 做预测。

因此就需要将使用不同训练框架训练出来的模型相互联系起来，使用户可以进行快速的转换。模型转换主要有**直接转换**和**规范式转换**两种方式，本文将详细介绍这两种转换方式的流程以及相关的技术细节。

## 模型转换设计思路

**直接转换**是将网络模型从 AI 框架直接转换为适合目标框架使用的格式。例如下图中的 MindSpore Converter 直接将 AI 框架 MindSpore 的格式转换成推理引擎 IR 的格式。

**规范式转换**设计了一种开放式的文件规范，使得主流 AI 框架可以实现对该规范标准的支持。例如不是直接转换 Pytorch 格式，而是把 Pytorch 转换为 ONNX 格式，或者把 MindSpore 转换成 ONNX 格式，再通过 ONNX Converter 转换成推理引擎 IR。主流 AI 框架基本上都是支持这两种转换技术的。

![模型转换](../../imageswtf/04Inference-04Converter-images-01Introduction01.png)

### 直接转换流程

直接转换的流程如下：

1. 内容读取：读取 AI 框架生成的模型文件，并识别模型网络中的张量数据的类型/格式、算子的类型和参数、计算图的结构和命名规范，以及它们之间的其他关联信息。
   
2. 格式转换：将第一步识别得到的模型结构、模型参数信息，直接代码层面翻译成推理引擎支持的格式。当算子较为复杂时，可在 Converter 中封装对应的算子转换函数来实现对推理引擎的算子转换。
   
3. 模型保存：在推理引擎下保存模型，可得到推理引擎支持的模型文件，即对应的计算图的显示表示。

直接转换过程中需要考虑多个技术细节，例如不同 AI 框架对算子的实现可能有差异，需要确保转换后的算子能够在目标框架中正确运行；不同框架可能对张量数据的存储格式有不同的要求，如 NCHW（批量数、通道数、高度、宽度）和 NHWC（批量数、高度、宽度、通道数）等，需要在转换过程中进行格式适配；某些框架的算子参数可能存在命名或含义上的差异，需要在转换过程中进行相应调整；为了保证转换后的模型在目标框架中的性能，可能需要对某些计算图进行优化处理，如算子融合、常量折叠等。

### 直接转换实例

以下代码演示了如何加载一个预训练的 TensorFlow 模型并进行直接转换为 PyTorch 模型的过程：

```python
import TensorFlow as tf
import torch
import torch.nn as nn

# 定义一个简单的 TensorFlow 模型
class SimpleModel(tf.keras.Model):
    def __init__(self):
        super(SimpleModel, self).__init__()
        self.dense1 = tf.keras.layers.Dense(64, activation='relu')
        self.dense2 = tf.keras.layers.Dense(10, activation='softmax')

    def call(self, inputs):
        x = self.dense1(inputs)
        return self.dense2(x)

# 1. 内容读取
# 创建并训练一个简单的 TensorFlow 模型
(x_train, y_train), _ = tf.keras.datasets.mnist.load_data()
x_train = x_train.reshape(-1, 784) / 255.0
y_train = tf.keras.utils.to_categorical(y_train, num_classes=10)
tf_model = SimpleModel()
tf_model.compile(optimizer='adam', loss='categorical_crossentropy', metrics=['accuracy'])
tf_model.fit(x_train, y_train, epochs=5)

# 2. 格式转换
# 定义对应的 PyTorch 模型结构
class PyTorchModel(nn.Module):
    def __init__(self):
        super(PyTorchModel, self).__init__()
        self.dense1 = nn.Linear(784, 64)
        self.relu = nn.ReLU()
        self.dense2 = nn.Linear(64, 10)

    def forward(self, x):
        x = x.view(-1, 784)
        x = self.dense1(x)
        x = self.relu(x)
        x = self.dense2(x)
        return x

# 将 TensorFlow 模型的参数转移到 PyTorch 模型中
pytorch_model = PyTorchModel()
with torch.no_grad():
    pytorch_model.dense1.weight = nn.Parameter(torch.tensor(tf_model.layers[0].get_weights()[0].T))
    pytorch_model.dense1.bias = nn.Parameter(torch.tensor(tf_model.layers[0].get_weights()[1]))
    pytorch_model.dense2.weight = nn.Parameter(torch.tensor(tf_model.layers[1].get_weights()[0].T))
    pytorch_model.dense2.bias = nn.Parameter(torch.tensor(tf_model.layers[1].get_weights()[1]))

# 3. 模型保存
# 保存转换后的 PyTorch 模型
torch.save(pytorch_model.state_dict(), 'pytorch_model.pth')

# 模型转换完成，可以使用 PyTorch 模型进行推理或继续训练
```

上述代码首先定义了一个简单的 TensorFlow 模型 SimpleModel 并在 MNIST 数据集上进行了训练。然后定义一个对应的 PyTorch 模型 PyTorchModel，其结构与 TensorFlow 模型相同。将 TensorFlow 模型中的参数转移到 PyTorch 模型中，确保权重参数正确地转移。最后保存转换后的 PyTorch 模型，以便在 PyTorch 中进行推理。

### 模型转换工具

这里列出部分可实现不同框架迁移的模型转换器：

| convertor | [mxnet](http://data.dmlc.ml/models/) | [caffe](https://github.com/BVLC/caffe/wiki/Model-Zoo)   | [caffe2](https://github.com/caffe2/caffe2/wiki/Model-Zoo) | [CNTK](https://www.microsoft.com/en-us/cognitive-toolkit/features/model-gallery/) | [theano](https://github.com/Theano/Theano/wiki/Related-projects)/[lasagne](https://github.com/Lasagne/Recipes) | [neon](https://github.com/NervanaSystems/ModelZoo) | [pytorch](https://github.com/pytorch/vision) | [torch](https://github.com/torch/torch7/wiki/ModelZoo) | [keras](https://github.com/fchollet/deep-learning-models)  | [darknet](https://pjreddie.com/darknet/imagesnet/) | [TensorFlow](https://github.com/TensorFlow/models)  | [chainer](http://docs.chainer.org/en/stable/reference/caffe.html) | [coreML/iOS](https://developer.apple.com/documentation/coreml) | [paddle](https://github.com/PaddlePaddle/models) | ONNX |
| --- |:--:|:--:|:--:|:--:|:--:|:-:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|**[mxnet](http://data.dmlc.ml/models/)**  |   -   | [MMdnn](https://github.com/Microsoft/MMdnn) [MXNet2Caffe](https://github.com/cypw/MXNet2Caffe) [Mxnet2Caffe](https://github.com/wranglerwong/Mxnet2Caffe) | [MMdnn (through ONNX)](https://github.com/Microsoft/MMdnn) | [MMdnn](https://github.com/Microsoft/MMdnn) | None | None | [MMdnn](https://github.com/Microsoft/MMdnn) [gluon2pytorch](https://github.com/nerox8664/gluon2pytorch) | None | [MMdnn](https://github.com/Microsoft/MMdnn) | None | [MMdnn](https://github.com/Microsoft/MMdnn) | None | [mxnet-to-coreml](https://github.com/apache/incubator-mxnet/tree/master/tools/coreml) [MMdnn](https://github.com/Microsoft/MMdnn) | None | None |
|**[caffe](https://github.com/BVLC/caffe/wiki/Model-Zoo)**  | [mxnet/tools/caffe_converter](https://github.com/dmlc/mxnet/tree/master/tools/caffe_converter) [ResNet_caffe2mxnet](https://github.com/nicklhy/ResNet_caffe2mxnet) [MMdnn](https://github.com/Microsoft/MMdnn) |  - | [CaffeToCaffe2](https://caffe2.ai/docs/caffe-migration.html#caffe-to-caffe2) [MMdnn (through ONNX)](https://github.com/Microsoft/MMdnn) | [crosstalkcaffe/CaffeConverter](https://github.com/Microsoft/CNTK/tree/master/bindings/python/cntk/contrib/crosstalkcaffe) [MMdnn](https://github.com/Microsoft/MMdnn) | [caffe_theano_conversion](https://github.com/an-kumar/caffe-theano-conversion) [caffe-model-convert](https://github.com/kencoken/caffe-model-convert) [caffe-to-theano](https://github.com/piergiaj/caffe-to-theano) |[caffe2neon](https://github.com/NervanaSystems/caffe2neon) | [MMdnn](https://github.com/Microsoft/MMdnn) [pytorch-caffe](https://github.com/marvis/pytorch-caffe) [pytorch-resnet](https://github.com/ruotianluo/pytorch-resnet) | [谷歌 net-caffe2torch](https://github.com/kmatzen/谷歌 net-caffe2torch) [mocha](https://github.com/kuangliu/mocha) [loadcaffe](https://github.com/szagoruyko/loadcaffe) | [keras-caffe-converter](https://github.com/AlexPasqua/keras-caffe-converter) [caffe_weight_converter](https://github.com/AlexPasqua/caffe_weight_converter) [caffe2keras](https://github.com/qxcv/caffe2keras) [nn_tools](https://github.com/hahnyuan/nn_tools) [keras](https://github.com/MarcBS/keras) [caffe2keras](https://github.com/OdinLin/caffe2keras) [Deep_Learning_Model_Converter](https://github.com/jamescfli/Deep_Learning_Model_Converter) [MMdnn](https://github.com/Microsoft/MMdnn) | [pytorch-caffe-darknet-convert](https://github.com/marvis/pytorch-caffe-darknet-convert) | [MMdnn](https://github.com/Microsoft/MMdnn) [nn_tools](https://github.com/hahnyuan/nn_tools) [caffe-TensorFlow](https://github.com/ethereon/caffe-TensorFlow) | None | [CoreMLZoo](https://github.com/mdering/CoreMLZoo) [apple/coremltools](https://apple.github.io/coremltools/) [MMdnn](https://github.com/Microsoft/MMdnn) | [X2Paddle](https://github.com/PaddlePaddle/X2Paddle) | [caffe2onnx](https://github.com/inisis/caffe2onnx) |
|**[caffe2](https://github.com/caffe2/caffe2/wiki/Model-Zoo)**| None | None | - | ONNX | None | None | ONNX | None | None | None | None | None | None | None | None |
|**[CNTK](https://www.microsoft.com/en-us/cognitive-toolkit/features/model-gallery/)**| [MMdnn](https://github.com/Microsoft/MMdnn) | [MMdnn](https://github.com/Microsoft/MMdnn) | ONNX [MMdnn (through ONNX)](https://github.com/Microsoft/MMdnn) | - | None | None | ONNX [MMdnn](https://github.com/Microsoft/MMdnn) | None | [MMdnn](https://github.com/Microsoft/MMdnn) | None | [MMdnn](https://github.com/Microsoft/MMdnn) | None | [MMdnn](https://github.com/Microsoft/MMdnn) | None | None |
|**[theano](https://github.com/Theano/Theano/wiki/Related-projects)/[lasagne](https://github.com/Lasagne/Recipes)**| None | None | None | None |   -   | None | None | None | None | None | None | None | None | None | None |
|**[neon](https://github.com/NervanaSystems/ModelZoo)**| None | None | None | None | None |   -   | None | None | None | None | None | None | None | None | None |
|**[pytorch](https://github.com/pytorch/vision)** | [MMdnn](https://github.com/Microsoft/MMdnn) | [brocolli](https://github.com/inisis/brocolli) [PytorchToCaffe](https://github.com/xxradon/PytorchToCaffe) [MMdnn](https://github.com/Microsoft/MMdnn) [pytorch2caffe](https://github.com/longcw/pytorch2caffe) [pytorch-caffe-darknet-convert](https://github.com/marvis/pytorch-caffe-darknet-convert) | [onnx-caffe2](https://github.com/onnx/onnx-caffe2) [MMdnn (through ONNX)](https://github.com/Microsoft/MMdnn) | ONNX [MMdnn](https://github.com/Microsoft/MMdnn) | None | None |   -   | None | [MMdnn](https://github.com/Microsoft/MMdnn) [pytorch2keras](https://github.com/nerox8664/pytorch2keras) [nn-transfer](https://github.com/gzuidhof/nn-transfer) | [pytorch-caffe-darknet-convert](https://github.com/marvis/pytorch-caffe-darknet-convert) | [MMdnn](https://github.com/Microsoft/MMdnn) [pytorch2keras](https://github.com/nerox8664/pytorch2keras) (over Keras) [pytorch-tf](https://github.com/leonidk/pytorch-tf) | None | [MMdnn](https://github.com/Microsoft/MMdnn) [onnx-coreml](https://github.com/onnx/onnx-coreml) | None | None |
|**[torch](https://github.com/torch/torch7/wiki/ModelZoo)** | None | [fb-caffe-exts/torch2caffe](https://github.com/Meta/fb-caffe-exts#torch2caffe) [mocha](https://github.com/kuangliu/mocha) [trans-torch](https://github.com/Teaonly/trans-torch) [th2caffe](https://github.com/e-lab/th2caffe) | [Torch2Caffe2](https://github.com/ca1773130n/Torch2Caffe2) | None | None | None |[convert_torch_to_pytorch](https://github.com/clcarwin/convert_torch_to_pytorch)|   -   | None | None | None | None | [torch2coreml](https://github.com/prisma-ai/torch2coreml) [torch2ios](https://github.com/woffle/torch2ios) | None | None |
|**[keras](https://github.com/fchollet/deep-learning-models)**  | [MMdnn](https://github.com/Microsoft/MMdnn) | [keras-caffe-converter](https://github.com/AlexPasqua/keras-caffe-converter) [MMdnn](https://github.com/Microsoft/MMdnn) [nn_tools](https://github.com/hahnyuan/nn_tools) [keras2caffe](https://github.com/uhfband/keras2caffe) | [MMdnn (through ONNX)](https://github.com/Microsoft/MMdnn) | [MMdnn](https://github.com/Microsoft/MMdnn) | None | None | [MMdnn](https://github.com/Microsoft/MMdnn) [nn-transfer](https://github.com/gzuidhof/nn-transfer) | None |   -   | None | [nn_tools](https://github.com/hahnyuan/nn_tools) [convert-to-TensorFlow](https://github.com/goranrauker/convert-to-TensorFlow) [keras_to_TensorFlow](https://github.com/alanswx/keras_to_TensorFlow) [keras_to_TensorFlow](https://github.com/amir-abdi/keras_to_TensorFlow) [MMdnn](https://github.com/Microsoft/MMdnn) | None | [apple/coremltools](https://apple.github.io/coremltools/)  [model-converters](https://github.com/triagemd/model-converters) [keras_models](https://github.com/Bulochkin/keras_models) [MMdnn](https://github.com/Microsoft/MMdnn) | None | None |
|**[darknet](https://pjreddie.com/darknet/imagesnet/)**| None | [pytorch-caffe-darknet-convert](https://github.com/marvis/pytorch-caffe-darknet-convert) | None | [MMdnn](https://github.com/Microsoft/MMdnn) | None | None | [pytorch-caffe-darknet-convert](https://github.com/marvis/pytorch-caffe-darknet-convert) | None | [MMdnn](https://github.com/Microsoft/MMdnn) |   -   | [DW2TF](https://github.com/jinyu121/DW2TF) [darkflow](https://github.com/thtrieu/darkflow) [lego_yolo](https://github.com/dEcmir/lego_yolo) | None | None | None | None |
|**[TensorFlow](https://github.com/TensorFlow/models)**  | [MMdnn](https://github.com/Microsoft/MMdnn) | [MMdnn](https://github.com/Microsoft/MMdnn) [nn_tools](https://github.com/hahnyuan/nn_tools)| [MMdnn (through ONNX)](https://github.com/Microsoft/MMdnn) | [crosstalk](https://github.com/Microsoft/CNTK/tree/master/bindings/python/cntk/contrib/crosstalk) [MMdnn](https://github.com/Microsoft/MMdnn) | None | None | [pytorch-tf](https://github.com/leonidk/pytorch-tf) [MMdnn](https://github.com/Microsoft/MMdnn) | None | [model-converters](https://github.com/triagemd/model-converters) [nn_tools](https://github.com/hahnyuan/nn_tools) [convert-to-TensorFlow](https://github.com/goranrauker/convert-to-TensorFlow) [MMdnn](https://github.com/Microsoft/MMdnn) | None |   -   | None | [tfcoreml](https://github.com/tf-coreml/tf-coreml) [MMdnn](https://github.com/Microsoft/MMdnn) | [X2Paddle](https://github.com/PaddlePaddle/X2Paddle) | None |
|**[chainer](http://docs.chainer.org/en/stable/reference/caffe.html)**| None | None | None | None | None | None |[chainer2pytorch](https://github.com/vzhong/chainer2pytorch)| None | None | None | None | - | None | None | None |
|**[coreML/iOS](https://developer.apple.com/documentation/coreml)** | [MMdnn](https://github.com/Microsoft/MMdnn) | [MMdnn](https://github.com/Microsoft/MMdnn) | [MMdnn (through ONNX)](https://github.com/Microsoft/MMdnn) | [MMdnn](https://github.com/Microsoft/MMdnn) | None | None | [MMdnn](https://github.com/Microsoft/MMdnn) | None | [MMdnn](https://github.com/Microsoft/MMdnn) | None | [MMdnn](https://github.com/Microsoft/MMdnn) | None | - | None |
| [paddle](http://paddlepaddle.org/) | None | None | None | None | None | None | None | None | None | None | None | None | None | - | None |
|**[ONNX](https://github.com/onnx/onnx)**| None | None | None | None | None | None | [onnx2torch](https://github.com/inisis/onnx2torch) [onnx2torch](https://github.com/ENOT-AutoDL/onnx2torch) | None | None | None | None | None | None | [X2Paddle](https://github.com/PaddlePaddle/X2Paddle) | - |

## 规范式转换

下面以 ONNX 为代表介绍规范式转换技术。

### ONNX 概述

ONNX(Open Neural Network Exchange)是一种针对机器学习所设计的开放式的文件格式，用于存储训练好的模型。它使得不同的 AI 框架（如 Pytorch、MXNet）可以采用相同格式存储模型数据并交互。

ONNX 的规范及代码主要由微软，亚马逊，Meta 和 IBM 等公司共同开发，以开放源代码的方式托管在 Github 上。目前官方支持加载 ONNX 模型并进行推理的 AI 框架有：Caffe2、PyTorch、MXNet、ML.NET、TensorRT 和 Microsoft CNTK，并且 TensorFlow 也非官方的支持 ONNX。

每个 AI 框架都有自己的图表示形式和特定的 API，这使得在不同框架之间转换模型变得复杂。此外，不同的 AI 框架针对不同的优化和特性进行了优化，例如快速训练、支持复杂网络架构、移动设备上的推理等。ONNX 可以提供计算图的通用表示，帮助开发人员能够在开发或部署的任何阶段选择最适合其项目的框架。

ONNX 定义了一种可扩展的计算图模型、一系列内置的运算单元（OP）和标准数据类型。每一个计算流图都定义为由节点组成的列表，并构建有向无环图。其中每一个节点都有一个或多个输入与输出，每一个节点称之为一个 OP。这相当于一种通用的计算图，不同 AI 框架构建的计算图都能转化为它。

规范式转换需要确保源框架能够正确导出规范格式的模型文件，并且目标框架能够正确导入；需要定义良好的跨框架兼容性，包括对各种算子的定义和数据格式的支持。同时还应具备良好的扩展性，能够适应新出现的算子和模型结构。

### PyTorch 转 ONNX 实例

这里读取在直接转换中保存的 PyTorch 模型`pytorch_model.pth`，使用`torch.onnx.export()`函数来将其转换为 ONNX 格式。

```python
x = torch.randn(1, 784)

# 导出为 ONNX 格式
with torch.no_grad():
    torch.onnx.export(
        pytorch_model,
        x,
        "pytorch_model.onnx",
        opset_version=11,
        input_names=['input'],
        output_names=['output']
    )
```

如果上述代码运行成功，目录下会新增一个`pytorch_model.onnx`的 ONNX 模型文件。可以用下面的脚本来验证一下模型文件是否正确。

```python
import onnx 
 
onnx_model = onnx.load("pytorch_model.onnx") 
try: 
    onnx.checker.check_model(onnx_model) 
except Exception: 
    print("Model incorrect") 
else: 
    print("Model correct")
```

`onnx.load`函数用于读取一个 ONNX 模型。`onnx.checker.check_model`用于检查模型格式是否正确，如果有错误的话该函数会直接报错。模型是正确的，控制台中应该会打印出"Model correct"。

使用 Netron（开源的模型可视化工具）来可视化 ONNX 模型：

![ONNX 模型可视化](../../imageswtf/04Inference-04Converter-images-04Detail02.png)

点击 input 或者 output，可以查看 ONNX 模型的基本信息，包括模型的版本信息，以及模型输入、输出的名称和数据类型。

![模型基本信息](../../imageswtf/04Inference-04Converter-images-04Detail03.png)

点击某一个算子节点，可以看到算子的具体信息。比如点击第一个 Gemm 可以看到：

![算子信息](../../imageswtf/04Inference-04Converter-images-04Detail04.png)

每个算子记录了算子属性、图结构、权重三类信息:

1. 算子属性信息：即图中 attributes 里的信息，这些算子属性最终会用来生成一个具体的算子。

2. 图结构信息：指算子节点在计算图中的名称、邻边的信息。对于图中的 Gemm 来说，该算子节点叫做`/fc1/Gemm`，输入数据叫做`input`，输出数据叫做`/fc1/Gemm_output_0`。根据每个算子节点的图结构信息，就能完整地复原出网络的计算图。
   
3. 权重信息：指的是网络经过训练后，算子存储的权重信息。对于图中的 Gemm 来说，权重信息包括`fc1.weight`和`fc1.bias`。点击图中 `fc1.weight`和`fc1.bias`后面的加号即可看到权重信息的具体内容。

## 模型转换通用流程

以下是模型转换的通用流程：

1. AI 框架生成计算图（以静态图表示），常用基于源码 AST 转换和基于 Trace 的方式：
   
**基于源码 AST 转换：** 分析前端代码来将动态图代码自动转写为静态图代码，通过词法分析器和解析器对源代码进行分析，然后对抽象语法树进行转写，将动态图代码语法映射为静态图代码语法，从而避免控制流或数据依赖的缺失，确保转换后的静态图模型与原动态图模型行为一致。

**基于 Trace：** 在动态图模式下执行并记录调度的算子，然后根据记录的调度顺序构建静态图模型，并将其保存下来。当再次调用模型时，直接使用保存的静态图模型执行计算。这种方法能够捕获动态执行过程中的所有操作，确保转换后的静态图模型能够准确再现动态图模型的行为。

2. 对接主流通用算子，确保模型中的通用算子在目标框架中能够找到对应的实现。针对模型中的自定义算子，需要编写专门的转换逻辑，可能需要在目标框架中实现相应的自定义算子，或者将自定义算子替换为等效的通用算子组合。
   
3. 目标格式转换，将模型转换到一种中间格式，即推理引擎的自定义 IR。中间格式 IR 包含了模型的计算图、算子、参数等所有信息，使得模型转换更加灵活和高效。；
   
4. 根据推理引擎的中间格式 IR，导出并保存模型文件，用于后续真正推理执行使用。

![模型转换通用流程](../../imageswtf/04Inference-04Converter-images-04Detail01.png)

在模型转换过程中，要注意确保源框架和目标框架中的算子兼容，能够处理不同框架中张量数据格式的差异。此外，还可以对计算图进行优化，提升推理性能，尽可能确保模型的精度不受损失。

## 小结与思考

- 模型转换流程：涉及将神经网络模型从一种框架转换到另一种框架，包括直接转换和通过开放式文件格式如 ONNX 的规范式转换。

- 关键技术细节：在模型转换过程中，需要处理算子兼容性、张量格式差异、参数适配以及计算图优化等问题，以确保模型在目标框架中的性能和精度。

- 工具与实例：多种模型转换工具如 MMdnn 和 ONNX 支持不同框架间的迁移，示例代码展示了如何将 PyTorch 模型转换为 ONNX 格式，进而实现跨框架模型部署。

## 本节视频

<html>
<iframe src="https://player.bilibili.com/player.html?isOutside=true&aid=436059900&bvid=BV13341197zU&cid=985638392&p=1&as_wide=1&high_quality=1&danmaku=0&t=30&autoplay=0" width="100%" height="500" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>
</html>
