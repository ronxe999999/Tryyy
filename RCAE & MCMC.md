##前言

这几天发现了神网大神之一（确切来说是DL大神之一）——Yoshua Bengio还未出版的*Deep Learning*。不过大神确实是厚道啊，把很多章节的草稿放到了网上。其中最吸引我的一章是*The Manifold Perspective on Auto-Encoders*。因为一直以来感觉对自编码网络的理解总是差那么一点点，于是这几天抽空把这一章读完，顺便读了两个相关的论文，感受颇深。虽然是草稿，但读过之后确实对自编码器和流形学习的关系“浮想联翩”了。因此，今天决定在这里记录一些自己的理解。

## 1 Regularized AutoEncoder 与 Manifold Learning

说到深度学习，就不得不提到自编码器。在非常有名的Stanford深度学习教程UFLDL中，深度神经网络的建模过程可以被看做：训练AE$\rightarrow$堆叠AE$\rightarrow$链接分类/回归器。因此，AE的知名度似乎要远大于比它的“前辈”RBM（也可能是因为RBM需要Markov网络的知识）。在UFLDL中，Ng大神主要讲述了一种常见的AE——Sparse AE（其实包括Bottleneck AE）。除了SAE之外，学者们还先后提出了Denoising AE和 Contractive AE。实际上，这些都可以被归并为Regularized AE，即对AE进行限制，迫使它们学习到输入数据的特征。SAE是利用稀疏惩罚，DAE利用了类似Dropout的办法，CAE则是惩罚AE编码部分的一阶导数（由于解码部分用了同样的权重$W$，实际上也算是对整体的一阶导数加罚了）。总之，Regularized AE的特点就一个字：Penalty！罚！

为什么罚？YB大神在DAE的文章中已经给出了关于DAE的流形学习解释：由于流行假设，数据是镶嵌在高维空间的低维流形，而DAE的通过对破坏的数据还原的学习，能够把数据从高维空间降到低维空间加以表示，从而学习到数据的生成规律（分布）。举例来说，虽然一个圆柱形的罐子曲面上的每一个点都处于一个三维空间，但我们只要知道了它在曲面上的二维坐标就可表示它了。流形学习就是基于这个假设，认为大多数自然中的数据，例如图像、语音都是符合这个规律。而罚AE的方法，就是强制告诉AE，你只要几个维度就可以完美表示得了这些高维数据的规律。这来源于训练AE时的两方面力量：
#### 1 Reconstruction error。它要求AE的训练结果尽可能复原输入。
#### 2 Regularization。它要求AE的训练结果尽量少地对样本的无关变动敏感。
这两个R的合力，就使得AE既要能够尽量还原输入，又要对无关变动不敏感。而符合这个要求的最优解，就是在数据集中的流形上面。一句话，RAE就是让AE学习到流形的形状，而为了这个目的，罚是必须的，否则就会让AE完美通过所有点，这样的解是没有任何意义的。（从这个角度看来，降维的方法都是在做这件事。例如PCA就是把数据从高维降到了线性的直线或者平面、超平面上。而AE是降到了非线性的超平面上）

## 2 Denoising AE 与 Contractive AE

DAE的损失函数为：
<img src="http://www.forkosh.com/mathtex.cgi? L_{DAE}=||r(\tilde{x})-x||^2">
其中$\tilde{x}$是通过一个方差为$\sigma^2$的Corruption分布将原样本破坏以后得到的新样本。DAE就是要从这些样本来试图还原本来的样本，从而得到训练。换句话说，通过加入噪声，使得AE通过压缩噪声来达到正则化，学到了流形。

比起DAE，CAE采用的方式是直接对编码函数的一阶导数加罚：
$$L_{CAE}=||r(x)-x||^2+\lambda||\frac{df(x)}{dx}||^2$$
保证学习到的编码函数，对输入的变动反应尽量小，即“收缩”。因此，CAE一方面要保留一些样本变动的方向来保证拟合，另一方面又要丢掉一些不重要的方向来保证罚。因此和DAE一样，CAE也会选择流形作为最优解。

综上，CAE和DAE确实都是在用不同的方法做类似的事情。但实际上不止如此，DAE和CAE之间还有更深层次的联系。YB大神在一篇论文中给出了这种关系。将DAE的损失函数变形，就可以写成如下形式：
$$L_{DAE}=||r(x)-x||^2+\sigma^2||\frac{dr(x)}{dx}||^2$$
详细推导过程见论文。由此可见，DAE和CAE都可以看做给AE本身的倒数加罚，DAE对整体加罚，而CAE只对编码部分加罚。并且DAE的参数和CAE的参数也是对应的。

得到了这些，YB大神干脆提出了一个General的模型：Reconstruction CAE，即CAE的整体加罚版本：
$$L_{RCAE}=||r(x)-x||^2+\lambda||\frac{dr(x)}{dx}||^2$$
这样，CAE和DAE就可以看做一类模型一起讨论了。

## 3 密度轨点与两种MCMC

前面鬼扯了一大堆，都是理论概念的理解，真正厉害的地方要从这里开始。

我们都知道RBM是一种概率模型，即它的推断和估计都必须通过抽样。虽然这给模型的解释性造成了困难，但如果学习目标就是一个概率分布，RBM理论上是可以拟合任意分布的。而反观AE，它是确定性的模型，解释起来要方便得多，但如果我们没办法从学到的分布中取样。而YB大神的另一个重要结论就是，DAE学习到的，其实就是生成样本的对数概率密度的导数：

$$r(x)=x+\sigma^2\frac{dlog(x)}{dx}+o(\sigma^2) \rightarrow \frac{r(x)-x}{\sigma^2}=\frac{dlog(x)}{dx}$$

这也就是说，通过充分训练的DAE，我们可以精确估计对数概率密度的导数。也就是小标题说的”轨点“了（轨点是梯度的英文音译，感觉和绩点GPA一样……出国党伤不起，这都能联系起来）。这就和MCMC联系了起来。（YB大神还画了一个Vector Fields的图。训练的样本数据来自经典的Swiss Roll，训练的结果就好像attractor吸引子一样，无数向量都指向Swiss Roll，非常漂亮！以后有时间会补充进来……另外某人真是没审美，什么叫看起来好扎的赶脚，明明是有吸引力好不啦？） 

回忆经典的MH算法：根据细致平稳条件，假设我们能得到密度函数的比值$\frac{p(x^s)}{p(x)}$，再利用一个对称的预选分布$q(x^s|x)$，我们能利用MH算法取得$p(x)$的样本。换句话说，我们的任务就是设法得到$\frac{p(x^s)}{p(x)}$。下面详细讨论这一个问题：

$\frac{p(x^s)}{p(x)}$ 可以写成$e^{logp(x^s)-logp(x)}$，而 $logp(x^s)-logp(x)$ 进行一阶泰勒展开得到：
$$ logp(x^s)-logp(x)=\frac{dlog(x)}{dx}(x^s-x)+o(x^s-x)$$

根据前面的结论我们知道利用DAE可以计算导数。因此，DAE可以根据这个方法构造MH（或者用Hamilton的方法构造HMC）抽样。

那么问题来了：标题中的“两种“作何解释？我们知道MCMC的方法主要有两类，MH和Gibbs。因此这个第二种就是Gibbs了。具体的方法也很简单：

因为我们有：
$$P(x,\tilde{x})=P(x)C(\tilde{x}|x)$$
其中C分布是已知的破坏分布。而根据DAE的训练方法我们知道，DAE可以从$\tilde{x}$还原$x$，说明DAE估计的结果就是$E(x|\tilde{x})$，也就是样本分布的均值。如果我们知道样本的具体分布，就可以得到$P(x|\tilde{x})$。而如果假定正态分布，我们只要计算一下样本的reconstruction error得到分布的另一个参数方差，就可以得到整体的分布了。

得到了我们想要的一切，就可以进行Gibbs抽样：

#### 1 $\tilde{X}_t=C(\tilde{X}|X_t)$
#### 2 $X_{t+1}=P(X|\tilde{X}_t)$

经过一段时间burn in，分布达到平稳，我们就可以去收人头……不对，是收$X$的样本了。由于是Gibbs，抽样的速度会快很多。

## 4 总结

大致总结起来就两句话：自编码器加罚来得到流形。通过加入随机性来进行抽样。看完这些后感觉AE现在的解释力度（特征提取）和实用性（抽样）都已经很强大，但YB大神似乎没有满足，干脆提出了个Generative Stochastic Network，把随机性引入了DAE的任一节点，不明觉厉。不过话说回来，我看这些的目的可不只是睡前读物……对付高频金融的第二个难题，DAE，就就决定是你了！
