# AI 之旅：跳伞

## 概述

像跳伞、蹦极、滑雪之类惊险刺激的运动是旅行乐趣中非常重要的一部分，而 AI 之旅中同样存在这样好玩刺激的项目，它就是 Magenta，一个探索机器学习在创作艺术和音乐过程中的作用的研究项目。也就是说，让 AI 涉足艺术领域，包括写歌、画画、写小说等，你可能会说了，AI 所谓的创作永远无法超越人类，是这样的，至少目前和可预见的未来是这样的，而 Magenta 的目的不是为了取代人类进行创作，而是探索艺术创作的过程，可以作为一种工具在人类创作艺术的过程中提供有价值的帮助  
Magenta 众多的模型中最出名的当属 Sketch-RNN 了，就像它的名字一样，它是一个可以自动生成素描（sketch）的循环神经网络（recurrent neural network）模型。如果你没听过 Sketch-RNN，那你一定玩过或者听过猜画小歌（Quick, Draw!）这款游戏吧，给你一个名字，你把它画出来，看 AI 能不能猜出你画的是什么，到目前为止，Quick, Draw! 数据集已经收集了 345 个类别共计 5000 万个矢量素描图，如果你玩过这个游戏，那么其中也有你的一份贡献  
![sketch_rnn](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/ai_parachuting_1.png)  
有趣的是谷歌的工程师小姐姐利用 Sketch-RNN 做了个魔法画板，能帮助你自动画猫  
![magic_skatchpad](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/ai_parachuting_2.gif)  
Sketch-RNN 目前只提供了 JavaScript 的实现，再加上模型和数据集比较大，我们就先不关心具体的实现细节了，我们更感兴趣的是名字叫 RNN 的这种神经网络究竟是什么，它为什么适合这种手绘图的生成，除了 RNN 还有什么比较有趣的神经网络

## 人工智能简史

### 传说

早在希腊神话中就出现了机器人和人造人，如火神赫菲斯托斯的黄金机器人女仆和塞浦路斯国王皮格马利翁的妻子加拉泰亚（被爱神赐予生命的雕像）  
中国古代也有类似的记载，在公元前 3 世纪的《列子》一书中曾记录了一个 “偃师献技” 的故事，讲述的是周穆王西巡归来的途中，还没有到达中国境内，遇到了自愿前来献技的工匠偃师，偃师第二天带来了一个像真人一样的歌舞艺人，能歌善舞，且心肝脾胃肾等虽然都是假的但一应俱全，连周穆王都感叹道 “人之巧乃可与造化者同功乎？”  
中世纪的时候，有传言说炼金术士可以将心智放入物品中  
到了 19 时机，关于人造人和思考机器的想法在小说中发展起来，如玛丽·雪莱的弗兰肯斯坦（有兴趣的可以看一下这个电影）  
虽然早在公元前 1000 年前中国、印度以及希腊的哲学家都开发了形式演绎的结构化方法，但是这么多世纪过去了，随着许多哲学家、心理学家、数学家等伟人的不断探索，才让人工智能或者说机械推理慢慢变为可能，如 17 世纪的莱布尼兹和笛卡尔等人探索了所有理性思想可以像代数或几何一样系统化的可能性，受到 1913 年罗素和怀特海所著《数学原理》的启发，David Hilbert 向二十世纪二三十年代的数学家们提出挑战：“所有的数学推理都可以形式化么？”，图灵机和 Lambda 演算等都给出了回答，这些回答令人惊讶的有两方面，一是它们证明了数学逻辑被实现时确实存在限制，二是这也表明在这些限制范围内任何形式的数学推理都可以机械化  

### 神经网络

20 世纪开始人们就已经知道了我们神经元的大概结构  
![neuron](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/ai_parachuting_3.png)  
1943 年 Warren S. McCulloch 和 Walter Pitts 发表的论文 “A Logical Calculus of Ideas Immanent in Nervous Activity” 提出了第一个神经网络的数学模型，该模型的每个单元都是一个简单形式化的神经元（neuron），也被称为 McCulloch–Pitts 神经元，可以执行简单的逻辑功能，灵感来自上面所说的神经元结构，这个模型就像后来的冯诺依曼计算机之于计算机领域一样深刻影响着神经网络领域，现在人们对于神经网络的研究与实践基本都是以此为基础的  
![mp](hhttps://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/ai_parachuting_4.png)  
还记得我们在《AI 之旅：启程》中是怎么把线性模型扩展到网络模型的么？没错，我们往一些隐藏层的节点中添加了非线性的函数，这些非线性函数也被称为激活函数，这样的话，节点（神经元）就可以通过把激活函数应用于输入值的加权和来计算输出值了，事实证明，通过这种方式构造的神经网络模型可以解决几乎所有的线性和非线性问题  
1949 年，Donald O. Hebb 在他的著作 “The Organization of Behavior” 中提出了 Hebbian 学习理论，这个理论为创建模拟生物神经系统的生理过程的计算机模型开辟了道路，也就是说，从这时开始，计算机的工程师们意识到了通过动态调整各个权重的方式就可以训练出一个好的模型了  
1950 年图灵在具有里程碑意义的论文中提出了著名的 “图灵测试”，简单点说就是让人和机器进行对话，如果有超过 30% 的人都不知道跟自己对话的是人还是机器，那么就表明这个机器通过了测试，表明这个机器具有人工智能  
1951 年，受到 McCulloch 和 Pitts 的启发，年仅 24 岁的学生 Marvin Minsky（和 Dean Edmonds 一起） 建造了第一个神经网络机器 SNARC，他也成为了 AI 下一个 50 年的重要领导者和创新者之一。同年 Christopher Strachey 写的跳棋程序也标志着游戏 AI 应用的开始  
1955 年，Allen Newell 和 Herbert A. Simon 创造了 “Logic Theorist” 程序，该程序证明了《数学原理》中的前 52 个定理中的 38 个，西蒙说他们已经 “解决了古老的 心智/身体 问题，解释了物质组成的系统如何具有心智的属性”  

### AI 诞生

1956 年，Marvin Minsky, John McCarthy 和来自 IBM 的两位资深科学家 Claude Shannon 和 Nathan Rochester 组织召开了达特茅斯会议（Dartmouth Conference），会议提出了一个主张 “学习的每个方面或任何其它智能特征都可以如此精确地描述，以便可以使机器模拟它”，McCarthy 说服了与会者接受 “人工智能”（Artificial Intelligence）作为该领域的名字，因此这次会议也被广泛认为是 AI 的诞生之日  

### 黄金年代

1956 – 1974 年，是人工智能的黄金年代，对于人工智能理论和程序的成果让人们充满了乐观和热情，甚至有研究人员乐观地认为不到 20 年就可以建造一台完全智能的机器，各个政府、公司也投入了大量的资金和人力参与人工智能的研发。1958 年康奈尔航空实验室的 Frank Rosenblatt 发明了被叫做感知器（perceptron）的机器（程序），从现在来看，它其实就是一个简单的二元线性分类器的监督式机器学习算法，但是在当时可以识别图像再加上美国军方的参与也引起了很大的轰动和争议  

### 第一次 AI 寒冬

1974 – 1980 年，出现了第一次 AI 寒冬，AI 受到了批判和资金困难，面对遥遥无期的智能机器人们当初的乐观和热情受到了打击，再加上 Marvin Minsky 对感知机的毁灭性批判，导致神经网络（连接主义 connectionism）领域停工了几乎十年  

### 繁荣期

1980 – 1987 年，AI 迎来了繁荣期，专家系统（expert systems）的出现让知识称为主流人工智能研究的焦点，各个政府和公司也开始了人工智能资金的支持。1982 年 John Hopfield 证明了 Hopfield 神经网络可以以一种全新的方式学习和处理信息，几乎同时，Geoffrey Hinton 和 David Rumelhart 推广了一种训练神经网络的方法，即著名的反向传播算法（backpropagation），这两项发现帮助了连结主义领域的复兴  

### 第二次 AI 寒冬

1987 – 1993 年，出现了第二次 AI 寒冬，专家系统虽然有用但仅限于少数特殊情况，AI 资金被一次次削减，很多 AI 公司倒闭、破产或被收购  

### 冷静期

1993 – 2011 年，是 AI 的冷静期，计算机的工程师们开始更加谨慎地投入人工智能研究，将概率和决策论引入 AI，专注于解决数据挖掘、语音识别、医学诊断、搜索引擎等子问题，工程师们甚至有意避免使用人工智能术语  
2011 年至今，随着深度学习、大数据和简易人工智能的研究与应用，人工智能更关注于解决一些特定的问题。其中深度学习（特别是深度卷积神经网络和循环神经网络）的进步推动了图像和视频处理、文本分析、甚至语音识别的进步和研究，深度学习是机器学习的一个分支，它使用具有很多处理层的深度图来模拟数据中的高级抽象，根据通用近似定理（Universal approximation theorem），神经网络不需要太多深度就可以逼近任意连续函数  

## 主流的神经网络

### 卷积神经网络

卷积神经网络（convolutional neural network）一般简写成 CNN 或 ConvNet，是用来分析视觉图像最常用的一类深度神经网络，它受到生物过程的启发，因为神经元之间的模式类似于动物视觉皮层的组织，皮质神经元仅仅对视野的限制区域的刺激做出反应（这个区域也被称为 receptive field），不同神经元的 receptive field 部分重叠以便可以覆盖整个视野  
卷积是分析数学中的一种积分运算，在图像处理中可以进行特征值的处理，以便消除噪声和特征增强  
![cnn](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/ai_parachuting_5.png)  
CNN 的隐藏层通常由一系列卷积层组成，激活函数通常是 RELU 层，随后是附加的卷积，如池化层（pooling layer）、全连接层（fully connected layer）、正则化层（normalization layer），最后的卷积通常涉及反向传播以便更准确地最最终结果进行加权  
CNN 广泛应用于图像和视频识别、推荐系统、图像分类、医学图像分析、药物发现、自然语言处理以及游戏 AI（如 AlphaGo）中  

### 循环神经网络

循环神经网络（recurrent neural network）一般简写成 RNN，是节点之间的连接形成沿时间序列的有向图的一类神经网络，与前馈神经网络（feedforward neural networks）不同的是，RNN 可以通过内部状态来处理输入序列，这使得它适用于处理未分段、连续手写识别或语音识别等任务  
![rnn](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/ai_parachuting_6.png)  
其实 RNN 是概况指代具有类似一般结构的两大类网络：有限脉冲（finite impulse）和无限脉冲（infinite impulse），有限脉冲循环网络是有向无环图，可以被展开并用严格的前馈神经网络替代，而无限脉冲循环网络则是无法展开的有向循环图。两者都可以有额外的存储状态，存储可以由神经网络直接控制，如果包含时间延迟或者反馈循环，那么存储可以被其它网络或图替代。这些受控状态被称为门控状态或门控存储器，是 LSTM（long short-term memory）网络和门控循环单元的一部分  
LSTM 是 RNN 最关键最优魅力的一部分，有了它才会有效防止反向传播时梯度消失和梯度爆炸情况的发生  
![lstm](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/ai_parachuting_7.png)  
很多应用选择使用 LSTM RNN 栈并用 CTC（Connectionist Temporal Classification）算法训练他们以找到一个 RNN 权重矩阵（在给定相应输入序列的情况下最大化训练集中标签序列的可能性），CTC 实现了对齐和识别  
与之前基于隐式马尔科夫模型或类似概念的模型不同的是，LSTM 可以学习识别上下文敏感的语言  
值得一提的是，2014 年百度使用 CTC 训练的 RNN 打破了 Switchboard Hub5'00 语音识别基准，而不使用任何传统的语音处理方法

## 有趣的神经网络

### 权重不可知神经网络

根据我们之前的了解，我们需要花费大量的时间在特征工程和训练权重上，那有没有可能不需要这些前提条件甚至不需要训练一个模型就能执行任务呢？权重不可知神经网络（Weight Agnostic Neural Networks）给出了答案，当然可以  
![BipedalWalker champion network](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/ai_parachuting_8.png)  
![BipedalWalker-v2](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/ai_parachuting_9.gif)  
工程师还把 WANN 用在 MNIST 数字分类任务上，发现未经训练和权重调整，就达到了 82% 左右的准确率

## 总结

人工智能之路充满了高潮和低谷，也充满了赞许和争议，值得欣慰的是，人们终于冷静了下来，不再绞尽脑汁地去创造意识，而是将机器学习这一子领域重视起来并应用于生活中，让机器学习这一领域的研究成果改善世界，改善人们的生活方式  
机器学习有很多有趣的应用，如我们上面说的 Sketch-RNN 可以画画，还有些模型可以写歌，可以写小说，可以给图片上色，甚至可以根据一段音频学习发音，根据一张图片生成视频，只要我们把它用在正确的地方，它就能给我们带来乐趣和帮助

## 参考

- [Magenta](https://github.com/tensorflow/magenta)
- [Sketch-RNN](https://github.com/tensorflow/magenta/tree/master/magenta/models/sketch_rnn)
- [Quick, Draw! Dataset](https://github.com/googlecreativelab/quickdraw-dataset)
- [Magic Sketchpad](https://magic-sketchpad.glitch.me/)
- [Weight Agnostic Neural Networks](https://weightagnostic.github.io/)
- [History of artificial intelligence](https://en.wikipedia.org/wiki/History_of_artificial_intelligence#Perceptrons_and_the_dark_age_of_connectionism)
- [谷歌小姐姐搞出魔法画板：你随便画，补不齐算AI输](https://mp.weixin.qq.com/s/-3e5xhbz01FerZp8DcRV5g)
- [神经网络浅讲：从神经元到深度学习](https://www.cnblogs.com/subconscious/p/5058741.html)
