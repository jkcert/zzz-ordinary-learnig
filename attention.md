# Attention #
三部曲：
+ score function ：度量环境向量与当前输入向量的相似性；找到当前环境下，应该 focus 哪些输入信息
+ alignment function ：计算 attention weight，通常都使用 softmax 进行归一化
+ generate context vector function ：根据 attention weight，得到输出向量

## generate context vector function ##
### hard attention ###
随机采样，采样集合是输入向量的集合，采样的概率分布是alignment function 产出的 attention weight。因此，hard attention 的输出是某一个特定的输入向量。
### soft attention ###
一个带权求和的过程，求和集合是输入向量的集合，对应权重是 alignment function 产出的 attention weight。  
*其中soft attention 是更常用的（后文提及的所有 attention 都在这个范畴），因为它可导，可直接嵌入到模型中进行训练*

>#### global/local attention ####
>soft attention又分为 global和local attention  
>global attention 是所有输入向量作为加权集合，使用 softmax 作为 alignment function  
>local 是部分输入向量才能进入这个池子。为什么用 local，背后逻辑是要减小噪音，进一步缩小重点关注区域。

## score function ##
如果两个向量在同一个空间，那么可以使用 dot 点乘方式（或者 scaled dot product，scaled 背后的原因是为了减小数值，softmax 的梯度大一些，学得更快一些）。  
如果不在同一个空间，需要一些变换（在一个空间也可以变换），additive 对输入分别进行线性变换后然后相加，multiplicative 是直接通过矩阵乘法来变换。

### self-attention ###
#### why self-attention? ####
相对于 CNN，要 constant path length 不要 logarithmic path length , 要 variable-sized perceptive field，不要固定 size 的 perceptive field；  
相对于 RNN，考虑长距离依赖，还要可以并行！  
这就是 self attention, 和 convolution 有点儿神似，它摒弃了 CNN 的局部假设，想要寻找长距离的关联依赖。  
+ constant path length & variable-sized perceptive field ：任意两个位置（特指远距离）的关联不再需要通过 Hierarchical perceptive field 的方式，它的 perceptive field 是整个句子，所以任意两个位置建立关联是常数时间内的。
+ parallelize : 没有了递归的限制，就像 CNN 一样可以在每一层内实现并行。

#### what is self-attention ####
+ QKV 都是对输入 x 的线性映射。  
+ score-function 使用 scaled-dot product。  
+ multihead 的方式将多个 head 的输出 z，进行 concat 后，通过线性变换得到最后的输出 z。  
![self-attention.png](https://upload-images.jianshu.io/upload_images/2509688-f613a0fe5b64831a.png "self-attention.png")

#### transformer中的self-attention ####
+ Encoder self-attention：Encoder 阶段捕获当前 word 和其他输入词的关联；  
+ MaskedDecoder self-attention ：Decoder 阶段捕获当前 word 与已经看到的解码词之间的关联，从矩阵上直观来看就是一个带有 mask 的三角矩阵；  
+ Encoder-Decoder Attention：就是将 Decoder 和 Encoder 输入建立联系，和之前那些普通 Attention 一样；  
![transformer.png](https://upload-images.jianshu.io/upload_images/2509688-158e290a98d90bcd.png "transformer.png")
