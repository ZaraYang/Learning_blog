# 理解置信度传播算法及其拓展

## 1、推断和图模型

​		本文将要描述和解释置信度传播算法（Belief propagation，BP），该算法可以用于（近似的）解决“推断”问题。推断问题存在于很多科学领域中，所以一个解决该问题的好方法在不停地被重新提出也并不奇怪。事实上，很多截然不同的算法比如前向后向（Forward-backward）算法、维特比（Viterbi）算法、Gallager code 和 turbocode 的迭代解码（iterative decodeing）算法，Pearl的贝叶斯网络信念传播算法，卡曼滤波（Kalman filter）和 物理中的传递矩阵方法（transfer-martix approach）都是BP算法在不同科学领域的变种。

理解一个方法在多学科问题上的应用将会对该方法产生更加深刻的认识，因此我们从AI领域，机器视觉领域，统计物理领域和digital communications literatures领域中选取简单的推断问题，这将会让我们有机会看到在解决这些问题时对应的不同图模型，第一节是一个对于基础知识的简单复习，但我们希望这些知识能够帮助读者理解这些不同领域中的问题之间的深度相似性。

### 1.1 贝叶斯网络

在人工智能领域的文献中，贝叶斯网络可能是最流行的图模型，贝叶斯网络构建的专家系统涉及了很多领域例如医疗诊断，地图学习，语言理解，启发式搜索等。我们会使用一个医学诊断问题作为例子，加入我们希望能够构建一台自动为患者诊断的机器，对于每一名患者，我们都会得到一些信息（可能不完整），例如病人的症状和检测结果，然后我们想*推断*某个或某些能够造成这些症状的疾病的概率。我们同样假设我们已经知道（来自医学专家的建议）不同症状、检测结果和疾病之间的统计学关联。例如，让我们思考一下Lauritzen 和 Spiegelhalter所虚构的“Asia”例子，如图一。

模型中展现了不同变量之间定性的统计学关联情况，可以用以下文字表述：

1. 患者若近期有亚洲（A）的旅行经历会提高其患肺结核的概率（T）。
2. 吸烟（S）是患肺癌（L）和支气管炎（B）的危险因素。
3. 是否存在肺结核或者肺癌（E）可以经由X光（X）检测，但是仅有X光无法区分二者。
4. 呼吸困难（D）可能由支气管炎或者肺癌和肺结核的存在导致。

图1中的每一个节点都代表了一个拥有离散概率分布的变量，我们用 ![](https://latex.codecogs.com/svg.latex?x_{i}) 来表示变量  ![](https://latex.codecogs.com/svg.latex?i) 的不同可能的状态。除了贝叶斯网络中描述的变量之间的定性相关性之外，还存在定量的统计学关系存在于图中每条边的箭头中，每个箭头都代表了一个条件概率：例如，我们将患者并不吸烟却患上肺癌的概率写为  ![](https://latex.codecogs.com/svg.latex?p(x_{L}|p_{S}) )，对于这种链接，我们称S节点为L节点的父节点（parent），因为L节点的状态是以S节点的状态为条件的。某些节点例如D节点可能会有不止一个父节点，在这种情况下，我们可以根据节点所有的父节点在定义条件概率，例如  ![](https://latex.codecogs.com/svg.latex?P(x_{D}|x_{E},x_{B}) )为呼吸困难的条件概率。

贝叶斯网络定义了一种独立结构：某一节点处在某一种状态的概率仅仅直接取决于其父节点的状态。而对于像A，S一样的节点，我们引入类似 ![](https://latex.codecogs.com/svg.latex?P(x_{S})) 的不以任何父节点为条件的概率。 总的来说，一个贝叶斯网络（或者其他我们提到的图模型）在稀疏时最好用，因为在这时大多数的节点都有一个直接的统计学关联。

在我们的例子中，联合概率即患者具有出多种症状，检测结果和疾病的概率为  ![](https://latex.codecogs.com/svg.latex?p(\{x\})=p(x_A,x_S,x_T,x_L,x_B,x_E,x_D,x_X))  
该联合概率可以被写成概率图中所有父节点（更类似与根节点）和所有条件概率的连乘：

![](https://latex.codecogs.com/svg.latex?p(\{x\})=p(x_A)p(x_S)p(x_T|x_A)p(x_L|x_S)p(x_B|x_S)p(x_E|x_L,x_T)p(x_D|x_B,x_E)p(x_X|x_E))

   更广义的说，一个贝叶斯网络就是一个具有N个随机变量![](https://latex.codecogs.com/svg.latex?x_{i})的有向无环图（directed acyclic graph），其联合概率可以表示为：

![](https://latex.codecogs.com/svg.latex?p(x_1,x_2,...,x_N)=\prod_{i=1}^{N}p(x_i|Par(x_i)))

其中![](https://latex.codecogs.com/svg.latex?Par(x_{i})) 代表节点 ![](https://latex.codecogs.com/svg.latex?i) 的父节点的状态，若节点 ![](https://latex.codecogs.com/svg.latex?i) 没有父节点，则：
![](https://latex.codecogs.com/svg.latex?p(x_i|Par(x_i))=p(x_{i})) 
有向无环图代表着，we mean that the arrows do not loop around in a cycle-it is still possible that the links form a loop when one ignores the arrows.

我们的目标是计算特定变量的边缘概率（marginal probability），例如：我们可能想要计算某病人患有特定疾病的概率。计算这些边缘概率的过程，即为“推断”。从数学上说，对系统中其他节点所代表的变量做积分即可得到边缘概率，例如，如果想要得到最后一个变量p_{N} 的边缘概率，则需要计算：

![](https://latex.codecogs.com/svg.latex?p(x_{N})=\sum_{x_1}\sum_{x_2}...\sum_{x_{N}-1}p(x_1,x_2,...,x_N)) 

我们将我们计算出来的近似的边缘概率成为“信念”（belief），并将节点i的信念表示为 ![](https://latex.codecogs.com/svg.latex?b(x_{i}))。

如果我们有关于一些节点的信息（例如在Asia例子中我们已经知道某患者不吸烟），然后我们固定对应的变量并且在积分时不会将那个节点的未知状态加进去。我们将这样的节点成为已知节点，相反的其他未知节点则被称为隐含节点（hidden），在我们的图模型中，我们将使用实心圆来代表已知节点，空心圆代表未知节点。

对于小型的贝叶斯网络，我们可以轻易地对其做积分来使其边缘化，但不幸的是，需要被加数的数量会随着隐含变量的增加指数上涨，BP算法的优点就是在（近似）计算边缘概率的时候计算复杂度会随着变量的增加线性增加，因此，BP算法在实践中可以被用作大型贝叶斯网络中统计数据的”推断引擎“（误？）。在回到BP算法 之前，我们需要先介绍一些其他的推断问题。