Var loss

## description

### background

使用模型对股票的收益情况进行预测分析，将模型的预测结果和真实情况之间的差距定义为loss。
对于大部分模型包括NN，模型训练的优化目标通常是训练batch中loss的均值。
实际实验中可能出现以下情况：模型对大部分股票都有很好的预测效果，但是对于特定的股票在关键日期的预测有极大的错误，这可能会使基金经理对此股票产生极大的误判并引发较大的回撤。
这种情况下虽然模型整体上表现良好，但可能具有较大的预测波动，对应的投资策略也有较大的风险。

有以下几个角度来考虑这个问题
- 极端数据：对于线性模型和许多非线性模型，极端数据会造成极端输出，引起该record数据较大的loss，如果用方差来衡量，那么就会有较大的波动。
比如实验中出现过一个batch的loss达到了1M，就是因为极端数据在MSE上的扩大效果
    - 数据标准化，减缓极端数据波动
    - 极端值抹平：比如5%数据抹平，将5%和95%的数据抹平或采用一个函数进行处理
- 样本权重：不同的股票具有不同样本权重，对那些用户更加关注的股票赋予更高的权重。
    - 在GBDT等算法中可以加入weight的参数
    - 在NN网络中可以在loss函数中引入weight
- loss函数
    - 在loss函数中加入样本weight
    - 在样本加入对loss波动的计算

### loss函数的选择

由计量经济学可以知道，对于使用线性模型，MSE是最优的loss function

但是由于模型训练的目标发生了改变，因此可以尝试不同的loss函数，会选取以下
- l1 loss
- nll_2
同时发现训练的时候波动很大，在数据处理的基础上可以加入平滑函数，smooth loss function
- smooth_l1_loss

### loss波动的选择

一个直接的做法是用loss的方差或者标准差来衡量波动

亦可以用一阶波动$\sum{|loss_i-loss_{mean}|}$的一阶绝对值距离来衡量。

具体的情况会随时间的变换

## 解决方法
加入了对预测误差波动的度量后，更新后的目标函数不一定是凸函数，即无法直接求出最优解。
因此使用BP网络来求解近似最优解。

实验中存在的问题：
- 由于网路训练的需求，需要设置batch_size,即并非对所有样本数据计算loss波动，而是计算这个batch集中的loss波动。起初batch_size设置为256，比较小，所以对样本的优化不明显，随后将其变大。
- 设计了三种类别进行对比

-


任务：
- classification
- regression

## Regression

建立多因子模型对股票进行建模

直接预测股票的收益

模型的loss即为真实收益和预测收益之间的差值

## Classification

进行分类问题

根据股票的收益情况给每个数据记录打上类别标签

单纯的直接预测收益有较大的误差，并且容易受到极端数据影响，也容易过拟合

1. 二分类
- 涨
- 跌

2. 多分类问题
比如三分类的问题，按照20%，60%，20%这几个分位数划分类别

具体做法而言：
- 每日的情况每一天分别划分类别
- 一段时间内的分位数

那么可以分为三类：
- 涨
- 波动
- 跌

同时分类模型也有不同的实现与视角
- 基于回归问题的分类：模型先对record进行打分，之后由分数按照上述的规则进行类别的划分，这样loss使用分类任务的loss，并且可以用模型输出的score得到rank_ic等指标
- 基于分类问题的分类：模型直接得到record在不同类别上的概率分布，再得到具体的，loss使用分类问题的loss，合理的model评测指标时AUC的分类指标，二分类与多分类的准确率问题，但是无法得到rank_ic指标值
第二种无法得到rank_ic的原因是模型直接给出的分类的类别或者record数据在不同类别上的概率，并不会给出直接的打分
    
    label generator
    打分机制
    ic_rank

## weight机制
每个样本赋予不同的权重，让模型对特定样本有更好的区分能力

GBDT的机制

NN的机制

不同的机制会产生不同的情况


## 权重的选择机制

权重是在样本的层面上对模型的训练过程进行干预，而添加样本的过程。

对样本附加权重的方法也不尽然相同。实际中也会随着具体问题发生变化。

有一些需要考虑的点和方向
- 等权或者不等权
- 权重的范围，最大的权重值或者最小的权重值
- 权重的分布方式
    - 高斯分布
    - 均匀分布
    - 双峰分布
    - 类别区块分布
- 权重赋予的函数形式
- 每次赋予权重的数据范围
    - 训练开始前对所有的数据记录进行统计分析
    - 对每个训练batch中的数据record进行展示

## 过拟合

### 评估过拟合
首先需要定义什么是过拟合

接着如何评估过拟合的程度

很多时候模型在train和validate上表现良好，但是在test上效果很差

但是对于金融股票预测的问题，test数据集是在未来的，无法得到，通过回测的test数据集又很难衡量这个问题。

### 解决过拟合

主流的方法主要是通过引入模型复杂度来减缓过拟合的现象。

- 正则项的使用，在loss函数中引入这一点
- 参数设计，模型复杂度

GBDT_like:目标函数中加入了正则项，从降低模型复杂度的角度避免了过拟合
- xgboost:参数的调控
- attention:相当于是时序变化的模型
- boosting 算法:一种comment是应该是对不同的参数产生的模型进行boosting
比如说，将20日间隔的模型和10间隔的模型boosting在一起，而不是使用两个10日间隔的模型组合在一起。
前者相当于使用不同的数据采样来对冲模型结构带来的风险，而后者则更像是设置了更大的决策树个数（在GBDT-like模型中的简单理解）

## 小样本

对于一般的机器学习任务比如图片分类，需要大量的样本数据。
但是股票通常是小样本，这就产生了一个矛盾。
现代的机器学习的主要目标是通过对大量样本的数据拟合来用网络的模型结构去逼近整体的数据分布情况。

数据增强一直是图像处理领域中解决小样本问题的一种方法，但这种方法能否迁移到股票数据的处理扩充上呢？
That's a big question.