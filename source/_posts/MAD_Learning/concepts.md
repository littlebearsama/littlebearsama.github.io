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
