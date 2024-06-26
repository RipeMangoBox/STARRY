# Transformer

参考内容：
1. [\[1706.03762\] Attention Is All You Need](https://arxiv.org/abs/1706.03762)
2. 文章：
	1. [Transformer模型详解（图解最完整版） - 知乎](https://zhuanlan.zhihu.com/p/338817680)**（绝赞）**
	2. [The Illustrated Transformer – Jay Alammar – Visualizing machine learning one concept at a time.](https://jalammar.github.io/illustrated-Transformer/)
	3. [分析Transformer模型的参数量、计算量、中间激活、KV cache - 知乎](https://zhuanlan.zhihu.com/p/624740065)
3. 视频：
	1. [1.从全局角度概括Transformer\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV1Di4y1c7Zm?p=1)
	2. [【李宏毅机器学习2021】自注意力机制 (Self-attention) (上)\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV1v3411r78R/?p=1)
4. 代码：
	1. [Multi-Headed Attention (MHA)](https://nn.labml.ai/Transformers/mha.html)
	2. [GitHub - jadore801120/attention-is-all-you-need-pytorch: A PyTorch implementation of the Transformer model in "Attention is All You Need".](https://github.com/jadore801120/attention-is-all-you-need-pytorch)


解答以下疑惑：
1. Attention中的QKV是如何获得的；
2. QKV是向量还是矩阵；
3. Self Attention和Cross Attention有什么区别；
4. 如何确定选择什么内容作为query；
5. Attention计算后，如何检索结果
6. Cross-Attention中，QKV的规模相比Self-Attention有无变化？如果有，面对不同规模的Q和K，如何进行相似度计算？
7. Multi-head的意义是什么；
8. CNN和Self-Attention的异同；
9. Transformer可训练参数有哪些；

**注**
1. 本篇中的向量和矩阵均针对self-attenion而非cross-attention。
2. 本篇的self-attention公式采用以下形式，与Attention原文一致。在这种表示方式下，QKV的每一行是一个向量。
$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$
李宏毅老师ppt是将一列看成一个向量，所以采取的是$`K^TQ`$的点积形式
![Pasted image 20240329120009](https://github.com/RipeMangoBox/STARRY/assets/134421090/38e78fb6-30b9-49ba-90c4-1c13a9c16a77)
![Pasted image 20240329120145](https://github.com/RipeMangoBox/STARRY/assets/134421090/5d525404-be07-4150-964f-d0c1016720ff)

-----------------------------------
## 向量相似度概念
首先明确向量内积的概念：
1. 两个向量a和b同向，a.b=|a||b|；
2. a和b垂直，a.b=0；
3. a和b反向，a.b=-|a||b|。
所以两个向量的点乘可以表示两个向量的相似度，越相似方向越趋于一致，a点乘b数值越大。

有了这个先验知识，回到self-attention上

$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$
Phere: 

- $Q$ is the query matrix, 
- $K$ is the key matrix, 
- $V$ is the value matrix, 
- $d_k$ is the dimension of $K$.

上面是self-attention的公式，Q和K的点乘表示Q和K元素之间(每个元素都是向量)的相似程度，$`\text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)`$称为**attention score**，利用**attention score**对V进行加权和得到输出结果。输出结果代表**根据attention score综合考虑整个序列后的结果**。

## Attention 流程
Transformer是一个Seq-2-Seq模型，也就是输入一个序列，经过Encoder和Decoder映射，得到的输出也是一个序列（但输入和输出的长度未必一样）。

![Pasted image 20240329194820](https://github.com/RipeMangoBox/STARRY/assets/134421090/8d3c602e-8fee-4a92-9533-0ada6d7474f5)


Transformer的架构图如下，由N组Encoder和Decoder构成，**每个Encoder输出的维度与inputs是相同的；Decoder也是如此**。

![Pasted image 20240329195140](https://github.com/RipeMangoBox/STARRY/assets/134421090/36f5d5ab-4124-4704-950a-3b926246f287)


### Encoder
输入一个$`N \times d`$的矩阵，每行代表一个词向量；输出也是一个$`N \times d`$的编码信息矩阵C。**Encoder实际做的事情是统计序列中每个token对序列语义的贡献程度，并将结果输出为编码信息矩阵C；而对Encoder的训练则是为了让其具备这种关注重要信息的能力。**
##### positional encoding
input输入是一个序列。对于序列中的每个token，我们首先将其映射为一个embedding $`a^i`$，然后加上与$`a^i`$等长的位置向量$`e^i`$，**对相加结果**经过$`W^Q,W^K,W^V`$权重矩阵的线性映射，得到QKV三个矩阵。对于multi-head的不同attention head，使用不同的$`W^Q,W^K,W^V`$。
![Pasted image 20240329200653](https://github.com/RipeMangoBox/STARRY/assets/134421090/d461fac3-cb91-491d-a4a1-ee3312153625)

##### Layer Norm
Add：对multi-head输出的向量经过加上残差连接；
Norm：对Add结果进行layer norm。Layer Norm会将每一层神经元的输入都转成均值方差都一样的，这样可以加快收敛。

使用Layer Norm而不是Batch Norm，原因是BN对一个batch中所有vector的同一个维度进行norm，而**在NLP任务中，BN往往是没有意义的**。比如下面这张图，每列是一个token的vector，对第一句的“爱”和第二句的“天”的vector进行归一化就没有意义（BN）。而对每一个序列自身的整体vector进行norm是有意义的，因为这个vector就蕴含这个序列的语义（LN）。
![Pasted image 20240329200116](https://github.com/RipeMangoBox/STARRY/assets/134421090/83e5d0f0-673d-452a-9e54-64791137de76)


##### Feed Forward Network
将multi-head concat后的{Z1, Z2, ……, Zn}**映射到与inputs相同的维度**。

### Decoder
Decoder 根据 Encoder 提供的知识和先前的解码信息生成目标序列的输出，实现序列到序列的翻译或生成任务。Transformer的Decoder是一个自回归模型，即基于自己过去的状态预测未来的输出。

一个Decoder block可以看作是由casual attention，cross attention，FFN三个部分组成，且inputs依次经过这三个模块。
	*casual attention*：强调因果，即我们只能基于已有的输出预测下一个词。decoder基于Teacher Forcing训练，需要mask掉后续的输入；
	*cross attention*：利用Encoder学习到的编码矩阵**计算得到**KV，用casual attention的输出Z计算Q，进行attention。由于引入了Encoder的知识，Decoder的每一位单词都可以利用到 Encoder 所有单词的信息 (这些信息无需 **Mask**)。

Decoder中的mask的目的是让当前时刻只能基于已知的信息进行attention，如下图所示。
![Pasted image 20240329203809](https://github.com/RipeMangoBox/STARRY/assets/134421090/bba5e5cd-4f69-4f97-ba16-e581c666031c)

那autoregressive model如何停止呢？方法是在输入侧加入Begin Token，在输出侧加入End Token，使得End Token也可以被预测为输出，从而结束预测。
![Pasted image 20240329204247](https://github.com/RipeMangoBox/STARRY/assets/134421090/81a38a98-9b76-4921-adb5-aa92c3fefbc6)
![Pasted image 20240329204155](https://github.com/RipeMangoBox/STARRY/assets/134421090/ea100da1-bb82-4ef4-b7b6-6dcc56d0caee)

## Attention Training
Teacher forcing用于实现**Decoder**在使用mask的同时的并行化training。
	所谓Teacher forcing，即模型在每个时间步骤上的输入都是真实的目标标记，而不是先前时间步骤上模型自身生成的标记。这样可以确保模型在训练过程中接收到正确的输入，有助于更快地收敛到较好的解。
在Transformer中，直接给Decoder正确的输入，并通过mask机制并行处理不同的当前位置，然后将Decoder输出的softmax分布于GT分布做交叉熵。

## 疑惑解答
### Attention中的QKV是如何获得的
QKV均由输入经过线性变换获得。对于self-attention，QKV均来自相同序列，数学表示如下
$$Q_{i} = W^{Q} * x_{i}$$
$$K_i = W^{K}* x_i$$
$$V_i = W^{V}* x_i$$
若是对于cross-attention，则Q和KV不来自同一个序列，Q需改为
$$Q_{i} = W^{Q} * y_{i}$$
上述权重矩阵$`W^Q,W^K,W^V`$均通过网络学习获得。需要注意的是：
$`W^Q,W^K,W^V`$通过整个训练数据集的学习得到，在一个*Scaled Dot Product-Attention*模块中共享，但是在*Multi-Head Attention*的不同head之间不共享。（即下图中，不同图层构成不同的*Scaled Dot Product-Attention*，他们有各自的$`W^Q,W^K,W^V`$)![Pasted image 20240318154149](https://github.com/RipeMangoBox/STARRY/assets/134421090/39ce0ea8-06c0-46f0-9d25-ae1639217aeb)

### QKV是向量还是矩阵
QKV可以是向量，其中*query和key都由维度为$`d_{k}`$*，value的维度为$`d_{v}`$ ； 也可以是矩阵，矩阵由若干向量拼接而成，方便并行化处理。
![Pasted image 20240318152931](https://github.com/RipeMangoBox/STARRY/assets/134421090/9e20e9cb-8e59-4a37-a371-f3ca679d3502)

换句话说，QKV都由一系列*行特征向量*组成，维度：
$`Q`$: $`N \times d_{k}`$

$`K`$: $`M \times d_{k}`$

$`V`$: $`M \times d_{v}`$

### 理解QKV的关系
以图书借阅请求为例。
Q是一组图书借阅请求；V是图书馆；对于图书馆中的每本书，有一个对应的K，这个K代表图书的标签，比如“惊悚”，“校园”等对这本书高度概括的信息，类似于这本书的身份证。

借书的流程就是图书管理员将Q和图书馆中所有书的K比对，然后根据比对结果，有的书匹配度高，有的书匹配度低。我们最终能得到一本按照匹配度加权的书。

需要注意的是，一千个人眼中有一千个哈姆雷特，所以V在不同的读者眼中是不同的；对于K也是如此，有的人觉得这本书是搞笑题材的，但也有人认为搞笑只是表面，悲剧才是内核；即使是对于Q，不同的图书管理员理解是不一样的。因此，我们需要引入多种标准来衡量QKV（即multi-head的不同head拥有各自的$`W^Q,W^K,W^V`$）。


**Attention计算的结果是V向量的加权和，因此只要先找到V，就能绑定确认K，自然就能决定谁是Q。**

### self-attention和cross-attention区别
1. 自注意力（我查我自己）:我们想知道句子中某个词在整个句子中的分量（或者相关文本），怎么设计？
- 句子本身乘以三个矩阵得到Q,K,V，每个词去查整个句子。
![Pasted image 20240318170020](https://github.com/RipeMangoBox/STARRY/assets/134421090/9dd1739a-80ff-44fd-9690-9cf8705bf643)


2. 交叉注意力（查别人）:Transformer模型的Decoder中，由Decoder的输入经过变换作为query，由Encoder的输出经变换作为key和value（数据库）。value和query来自不同的地方，就是交叉注意力。可以看到key和value一定是代表着同一个东西。即\[Q,(K,V)]。由于Encoder是见过整个序列的，因此其输出的编码矩阵包含了序列中所有token间的关联程度，因此可以作为一种知识，提供KV给Decoder进行query。

### 如何通过Attention计算获取检索结果
Attention计算后，数值从高到低便是匹配度的从高到低。

### Cross-Attention中，Q和K来自不同的序列，如何进行维度匹配
可以使用线性映射实现两者的维度匹配，代码如下：
```
# 假设 Q 是文本特征，K 是图像特征
Q = ...  # 形状为 [batch_size, seq_len, d_text]
K = ...  # 形状为 [batch_size, num_patches, d_image]

# 使用线性层将 Q 和 K 映射到相同的维度 d_k
WQ = nn.Linear(d_text, d_k)  # 文本到共享维度的映射
WK = nn.Linear(d_image, d_k)  # 图像到共享维度的映射

# 应用映射
Q_mapped = WQ(Q)  # 形状变为 [batch_size, seq_len, d_k]
K_mapped = WK(K)  # 形状变为 [batch_size, num_patches, d_k]

# 现在 Q_mapped 和 K_mapped 可以进行内积操作
```

### Multi-head的意义
一组$`W^Q,W^K,W^V`$，就代表一种相关性度量。但是对于输入序列，可能不止存在一种合适的相关性（**different types of relevance**），因此使用multi-head来度量多种相关性，最终再一次进行加权和是更合理的做法。

### CNN和Self-Attention的异同
李宏毅老师一张图概括很好：
1. CNN感受野是局部的，人为规定的；而SA的感受野是整个序列
2. CNN是SA的特例，SA进行特定限制后能得到与CNN完全相同的能力；即CNN的function set是SA function set的子集；
3. 由第二点可知，SA比CNN更加地flexible，因此当数据量没有达到一定规模时，SA的表现会弱于CNN，反之超越。
![Pasted image 20240329121530](https://github.com/RipeMangoBox/STARRY/assets/134421090/f35bea0d-4d07-4eae-ad1b-b2d5f6c293c2)



### Transformer可训练参数有哪些
1. 获取QKV的三个线性映射矩阵$`W^Q,W^K,W^V`$和偏置bias;
2. positional embedding若采用学习的方式，需具体分析；
3. layer norm的缩放参数$`\gamma`$和平移$`\beta`$;
4. FFN层；
5. Decoder输出的linear层；
6. token映射为embedding的词嵌入矩阵

关于参数量的具体计算，可见[分析Transformer模型的参数量、计算量、中间激活、KV cache - 知乎](https://zhuanlan.zhihu.com/p/624740065)
