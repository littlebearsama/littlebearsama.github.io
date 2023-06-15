# 1. 多任务网络yolop
> 多任务分为并联级任务以及级联任务
    cudnn.benchmark = cfg.CUDNN.BENCHMARK
    torch.backends.cudnn.deterministic = cfg.CUDNN.DETERMINISTIC
    torch.backends.cudnn.enabled = cfg.CUDNN.ENABLED

>  Benchmark模式会提升计算速度，但是由于计算中有随机性，每次网络前馈结果略有差异。
torch.backends.cudnn.benchmark = True
如果想要避免这种结果波动，设置：
torch.backends.cudnn.deterministic = True

SPP SPPF
空间金字塔池化 Spatial Pyramid Pooling - Fast (SPPF)

bottleneck （沙漏型结构）
https://blog.csdn.net/hxxjxw/article/details/106582884
[详解YOLOv5中的Bottleneck](https://blog.csdn.net/weixin_55073640/article/details/122588049)

[YoloV5网络结构梳理](https://blog.csdn.net/zebra_0/article/details/120769404?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-120769404-blog-122588049.235^v38^pc_relevant_sort_base1&spm=1001.2101.3001.4242.1&utm_relevant_index=1)
01.Focus模块
02.Conv模块
03.Bottleneck模块
04.C3模块
05.SPP模块

