在看Daniel大佬关于MotionMatching的文章时，看到了一篇关于神经网络的文章，上面提到了Data Normalization，原文链接在这里：https://theorangeduck.com/page/neural-network-not-working，然后就利用业余时间学习整理了下~

# 1 概念

由于归一化和标准化这两个中文概念在网络上争议很大，所以下面的内容不再细分，全部使用Data Normalization来代替。可移步[标准化和归一化什么区别？](https://www.zhihu.com/question/20467170)

Feature scaling（特征缩放）是一种用于归一化数据的自变量或特征范围的方法。在数据处理中，它也被称为Data Normalization，通常在数据预处理步骤中执行。

使用Data Normalization的原因：由于原始数据的取值范围变化很大，在一些机器学习算法中，如果不进行Data Normalization，目标将无法正常工作。在统计中中，样本数据都是多个维度的，一个样本是用多个特征来表征的，这些特征的单位也不一致，如果不进行Normalization，那么这些特征对最终结果的影响很可能是不一样的。而使用Data Normalization后，每个特征具有相同的尺度，也对最终结果的贡献大约成比例。

方法：

1、**Rescaling (min-max normalization)**

- 最简单的方法，把特征数据缩放在指定范围内。

- [0,1]缩放的公式为：$x'=\frac{x-min(x)}{max(x)-min(x)}$（x是原始值，x'是Normalization后的值）
  
- [a,b]缩放的公式为：$x'=a+\frac{(x-min(x))(b-a)}{max(x)-min(x)}$（x是原始值，x'是Normalization后的值，a为最小值，b为最大值）

2、**Mean normalization**

- 公式为：$x'=\frac{x-average(x)}{max(x)-min(x)}$（x是原始值，x'是Normalization后的值）

3、**Standardization (Z-score Normalization)**

- 在机器学习中，可以处理各种类型的数据，这些数据可以包含多个维度。特征标准化使数据中每个特征的值具有均值为0和标准差为1的服从标准正态分布的特点。

- 公式为：$x'=\frac{x-\bar{x}}{\sigma}$（x是原始值，x'是Normalization后的值， $\overline{x}$ = average(x)是特征向量的均值，分母是标准差)
  
- 更适用于正态分布的数据：

  ![](pictures\Z-score Normalization.png)

4、**Scaling to unit length**

- 缩放特征向量的分量，使完整向量的长度为1。

- 公式为：$x'=\frac{x}{||x||}\\$

# 2 基础数学知识

表征数据的离散程度：离差平方和、（总体）方差和（总体）标准差，它们的共同性质：最小值为0。数据的离散程度越大，它们的值也越大。

1、**平均数**：严格来说，应该称为算数平均数或均值，其他还有几何平均、调和平均等平均数。

2、**离差平方和**：

- 离差：距离平均值的远近状况。

- 离差平方和公式：$SS=\Sigma(x_{i} - \bar{x})^{2}$
  
- 离差平方和缺点：数据的个数越多时，它的值也会变得越大，所以实际很少使用它作为表征离散程度的指标。

3、**（总体）方差**：

- 解决了离差平方和的缺点。

- 公式：$\sigma^{2}=\frac{\Sigma(X-\mu)^{2}}{N}\\$（ $\sigma^{2}$  为总体方差，X为变量， $\mu$为总体均值，N为总体倒数。）

4、**标准差**：表现离散程度。表示一组数据平均离散程度的指标。标准差最小值为0（完全不离散，全为相同数据），而数据的离散程度越大，标准差的值就越大。并且消除了负偏差。

- 总体标准差：从本质上讲与（总体）方差是相同的。总体是真正想调查的对象的集合。$SD=\sqrt{\frac{1}{N}\sum_{i=1}^N(x_{i}-\mu)^{2}}$（$\mu$为平均值$\bar{x}$）
  
- 样本标准差：样本是从总体中被选出来的人所形成的集合，n < N。$s=\sqrt{\frac{1}{N-1}\sum_{i=1}^N(x_{i}-\bar{x})^{2}}\\$

# 3 UE4/UE5应用

在UE4/UE5引擎中，我们常用的标准化方式是min-max normalization和Z-score Normalization。

1、在UE4中：

![](pictures\UE4标准化.png)

2、在UE5中，Pose Search插件使用了Z-score Normalization：

（PoseSearch插件可用于MotionMatching功能。在MotionMatching中，要从Pose数据库中取出最合适的Pose，所以会有很多特征来进行查找对比，这些特征也都影响最终的运行效果，比如trajectory的位置、Pose的速度等，而每个特征带来的Cost范围可能会不一样，也就是度量不同，如果不对特征数据进行Normalization，后期将要反复调整权重，最终效果可能也不好。所以在MotionMatching中，Normalization是很重要的一步。）

![](pictures\UE5标准化.png)

这个函数使用矩阵进行计算，是一个不错的方法。

与Z-score算法不一样的是，这里没有使用标准方差，而是使用的平均绝对偏差。关于原因，注释也解释了一下，并且放了paper的名字。注释：标准偏差更强调异常值，因为离差平方会以指数方式增加方差，而不是以相加的方式增加方差，而平方和的平方根不会增加方差消除这种偏见。

paper内容：介绍了标准方差（SD）和平均绝对偏差（MD）的概念，从历史原因分析为什么大多数采用SD而不是MD，为什么采用MD，最后总结概括。也就是说，在某些情况下，MD更优于SD。MD在数据包含微小误差或者不能形成完全的正态分布时效率更好，而且它还更容易使用，更能容忍极端值。SD优越性在于随机选取样本，更依赖于正态分布的数据，SD更强调较大的偏差，SD和MD的相对效率取决于是否存在在观察中没有任何错误。但是正态分布具有数据污染小的相对优势。所以MD实际上在观察和测量中会出现小错误的情况下比SD更有效率（比SD的效率高出一倍以上）。

3、其他：这样当然要提一下Daniel Holden大佬的MotionMatching代码了，也做了Normalization。https://github.com/orangeduck/Motion-Matching

在database.h文件中，这个函数使用的是标准的Z-score Normalization，有兴趣的同学可以去看看~

```c++
void normalize_feature(
    slice2d<float> features,
    slice1d<float> features_offset,
    slice1d<float> features_scale,
    const int offset, 
    const int size, 
    const float weight = 1.0f)
{
    // First compute what is essentially the mean 
    // value for each feature dimension
    for (int j = 0; j < size; j++)
    {
        features_offset(offset + j) = 0.0f;    
    }
    
    for (int i = 0; i < features.rows; i++)
    {
        for (int j = 0; j < size; j++)
        {
            features_offset(offset + j) += features(i, offset + j) / features.rows;
        }
    }
    
    // Now compute the variance of each feature dimension
    array1d<float> vars(size);
    vars.zero();
    
    for (int i = 0; i < features.rows; i++)
    {
        for (int j = 0; j < size; j++)
        {
            vars(j) += squaref(features(i, offset + j) - features_offset(offset + j)) / features.rows;
        }
    }
    
    // We compute the overall std of the feature as the average
    // std across all dimensions
    float std = 0.0f;
    for (int j = 0; j < size; j++)
    {
        std += sqrtf(vars(j)) / size;
    }
    
    // Features with no variation can have zero std which is
    // almost always a bug.
    assert(std > 0.0);
    
    // The scale of a feature is just the std divided by the weight
    for (int j = 0; j < size; j++)
    {
        features_scale(offset + j) = std / weight;
    }
    
    // Using the offset and scale we can then normalize the features
    for (int i = 0; i < features.rows; i++)
    {
        for (int j = 0; j < size; j++)
        {
            features(i, offset + j) = (features(i, offset + j) - features_offset(offset + j)) / features_scale(offset + j);
        }
    }
}
```

# 4 参考

Feature scaling：https://en.wikipedia.org/wiki/Feature_scaling

Standard score：https://en.wikipedia.org/wiki/Standard_score

Normalization (statistics)：https://en.wikipedia.org/wiki/Normalization_(statistics)

Motion-Matching：https://github.com/orangeduck/Motion-Matching

数据什么时候需要做中心化和标准化处理？：https://www.zhihu.com/question/37069477

Revisiting a 90-Year-Old Debate: The Advantages of the Mean Deviation：https://emilkirkegaard.dk/en/wp-content/uploads/Revisiting-a-90-year-old-debate-the-advantages-of-the-mean-deviation.pdf
