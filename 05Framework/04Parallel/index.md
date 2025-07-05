<!--Copyright © ZOMI 适用于[License](https://github.com/chenzomi12/AISystem)版权许可-->

# 分布式并行

什么是大模型？大模型模型参数量实在太大，需要分布式并行训练能力一起来加速训练过程。分布式并行是在大规模 AI 集群上工作的，想要加速就需要软硬件协同，不仅仅要解决通信拓扑的问题、集群组网的问题，还要了解上层 MOE、Transform 等新兴算法。通过对算法的剖析，提出模型并行、数据并行、优化器并行等新的并行模式和通信同步模式，来加速分布式训练的过程。最小的单机执行单元里面，还要针对大模型进行混合精度、梯度累积等算法，进一步压榨集群的算力！

> 希望这个系列能够给大家、朋友们带来一些些帮助，也希望自己能够继续坚持完成所有内容哈！

**内容大纲**

> `PPT`和`字幕`需要到 [Github](https://github.com/chenzomi12/AISystem) 下载，网页课程版链接会失效哦~
>
> 建议优先下载 PDF 版本，PPT 版本会因为字体缺失等原因导致版本很丑哦~

| 小节 | 链接|
|:--:|:--:|
| 01 基本介绍 | [文章](./01Introduction.md), [PPT](./01.introduction.pdf), [视频](https://www.bilibili.com/video/BV1ve411w7DL/) |
| 02 数据并行(上) | [文章](./02DataParallel.md), [PPT](./02DataParallel.pdf), [视频](https://www.bilibili.com/video/BV1JK411S7gL/) |
| 02 数据并行(下) | [文章](./03ZeRODP.md), [PPT](./02DataParallel.pdf), [视频](https://www.bilibili.com/video/BV1JK411S7gL/) |
| 03 模型并行(上) | [文章](./04TensorParallel.md), [PPT](./03TensorParallel.pdf), [视频](https://www.bilibili.com/video/BV1vt4y1K7wT/) |
| 03 模型并行(下) | [文章](./05PipelineParallel.md), [PPT](./05PipelineParallel.pdf), [视频](https://www.bilibili.com/video/BV1WD4y1t7Ba/)|
| 04 混合并行 | [文章](./06HybridParallel.md), [PPT](./04MindsporeParallel.pdf), [视频](https://www.bilibili.com/video/BV1vt4y1K7wT/) |
| 05 分布式训练总结 | [PPT](./07Summary.pdf), [视频](https://www.bilibili.com/video/BV1av4y1S7DQ/) |

## 备注

文字课程开源在 [AISys](https://chenzomi12.github.io/)，系列视频托管[B 站](https://space.bilibili.com/517221395)和[油管](https://www.youtube.com/@ZOMI666/videos)，PPT 开源在[github](https://github.com/chenzomi12/AISystem)，欢迎取用！！！

> 非常希望您也参与到这个开源课程中，B 站给 ZOMI 留言哦！
> 
> 欢迎大家使用的过程中发现 bug 或者勘误直接提交代码 PR 到开源社区哦！
>
> 欢迎大家使用的过程中发现 bug 或者勘误直接提交 PR 到开源社区哦！
>
> 请大家尊重开源和 ZOMI 的努力，引用 PPT 的内容请规范转载标明出处哦！

    
```toc
:maxdepth: 2
:numbered:

01Introduction
02DataParallel
03ZeRODP
04TensorParallel
05PipelineParallel
06HybridParallel
07MSParallel
```
        