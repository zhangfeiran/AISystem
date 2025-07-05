# 分布式算法

什么是大模型？大模型模型参数量实在太大，需要分布式并行训练能力一起来加速训练过程。分布式并行是在大规模 AI 集群上工作的，想要加速就需要软硬件协同，不仅仅要解决通信拓扑的问题、集群组网的问题，还要了解上层 MOE、Transform 等新兴算法。通过对算法的剖析，提出模型并行、数据并行、优化器并行等新的并行模式和通信同步模式，来加速分布式训练的过程。最小的单机执行单元里面，还要针对大模型进行混合精度、梯度累积等算法，进一步压榨集群的算力！

内容大纲

> *建议优先下载或者使用 PDF 版本，PPT 版本会因为字体缺失等原因导致版本很丑哦~*

| 编号  | 名称    | 名称                | 备注                                                                                       |
| --- | -- | -- | ------- |
|     | 分布式算法 | 06 大模型训练的挑战       | [PPT](./01Challenge.pdf), [视频](https://www.bilibili.com/video/BV1Y14y1576A/)      |
|     | 分布式算法 | 07 算法：大模型算法结构     | [PPT](./02AlgorithmArch.pdf), [视频](https://www.bilibili.com/video/BV1Mt4y1M7SE/) |
|     | 分布式算法 | 08 算法：亿级规模 SOTA 大模型 | [PPT](./03AlgorithmSota.pdf), [视频](https://www.bilibili.com/video/BV1em4y1F7ay/) |

备注

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


```
        