### 阶段性报告

#### 关于之前的过拟合问题
##### 对比了双输入情况下基于CNN的三篇论文：
* （2014年）DeepReID: Deep Filter Pairing Neural Network for Person Re-Identification
* （2015年）An Improved Deep Learning Architecture for Person Re-Identification
* （2016年）PersonNet: Person Re-identification with Deep Convolutional Neural Networks
##### 论文对比分析：
由于三篇论文在网络结构上，依次借鉴，并无太大差异，尤其16年论文几乎就是完全参考了15年CVPR的论文，因此我们仔细的阅读了论文之后，发现，在训练策略上，三篇论文虽然依次借鉴，但实际上有了一些区别，我们在试图解决过拟合问题的时候，发现这些微小的区别可能会产生巨大的影响，下面是对于数据处理和训练策略上三篇论文的对比分析。由于论文里都采用过CUHK03，因此以CUHK03为例，里面包含1360个人，每个人都用两个摄像头去拍摄，每个摄像头基本上拍摄了5张照片，所以彼此建立连接共有25组正数据。
##### 数据处理方式和训练策略对比分析
* 14年Deep ReID
1.  采用对正数据做随机平移的方式进行augmentation，使每个人的图片增加至原来的5倍，相当于**正数据扩大了25倍**
2.   **最开始用1：1的正负数据比去训练网络，然后逐步增加到1：5**
3.  后期选择那些网络表现不好的负数据与正数据一同送进网络对网络作fine-tune
* 15年CNN
1. augmentation同14年
2. 直接选取**正负数据比为1：2**，训练整个网络
3. 然后将负数据中分类不好的挑选出来，与相同数量的正数据一同放入网络中，对**最后一层**即500个神经元的全连接层重新训练
* 16年PersonNet
1. 采取与前两年相同的augmentation策略
2. 对于负数据，此论文采取了**在线提取策略**，在正数据训练的同时，负数据源源不断的重新采样放进去，这样将会把全部负数据放入网络中训练，并且，**正负数据维持1：1的关系。**

##### 科研日志：
* 在对比了三年来的处理方式以后，我们发现，我们的过拟合问题产生于做augmentation时，不是对每个照片，而是对每对正数据做的augmentation，因而正数据扩大的倍数其实不是25倍，而是5倍，此时将其按照1：2的比例放入网络，当迭代次数上升到一定数目后，网络便开始过拟合。同时，由于在数据处理上，**如果提前做好正数据和负数据，那么可能会需要存入几百G的数据，此时一旦需要更改，就会变得很麻烦。**
* 同时，由于16年PersonNet论文里采用的**在线提取负数据**的方式，引起了注意，实际上，这也是我们觉得这篇论文唯一值得借鉴的地方，因为其网络结构基本上就是抄袭15年的论文，然后换了一种优化器而已。我们对数据重新做了处理，然后修改了送入网络的数据迭代器，使其对正数据做实时的augmentation，也就是随机平移，**对正负样本同时采取在线的随机抽样。**在一定的理论分析过后，我们认为，这种方式应该是目前为止，ReID领域里对于正负样本比例悬殊问题的最佳解决办法。
* 但是实验效果欠佳，网络的效果始终上不去，CUHK03上的rank1还不到30%，只有20%多，market1501的rank1更是只有17%。
* 于是，我们打算将模型做分布式处理，让其在不同的单卡上做不同参数的训练，然后6张卡同时开始对CUHK03训练，每一轮150万对图片输入，每一轮结束后计算cmc曲线并输出，然后保存weights到本地，共计20轮，20轮也是论文里所说的迭代总数。**我们将随机平移改为概率行为，分别以0，0.2，0.4，0.6，0.8.，1的概率对正数据的两个图片做随机平移，负数据不动，正负数据均为在线随机采样。**
* 目前，6个不同参数的模型正在训练，但是已经初露端倪
##### 关于随机平移的概率的实验现象（还在训练，但是已经能看出效果）
1. 当随机平移概率为`1`时：
 ![Alt text](./0_0.png)
2. 当随机平移概率为`0.8`时：
![Alt text](./0_2.png)
3. 当随机平移概率为`0.6`时：
![Alt text](./0_4.png)
4. 当随机平移概率为`0.4`时：
![Alt text](./0_6.png)
5. 当随机平移概率为`0.2`时：
![Alt text](./0_8.png)
6. 当随机平移概率为`0`时：
![Alt text](./1_0.png)

##### 关于实验现象的猜测：
当随机平移概率为1时，由于全部的正数据已经都被修改，在用原始数据集去测试网络的时候，由于数据分布发生了改变，因此，实验效果一定是欠佳的。但是，当随机平移概率小于1时，平移的概率越高，网络的rank1上升越缓慢，但是会继续上升。

**我们预估，随机平移的概率在小于1的情况下：**
1. 概率越高，网络训练效果越好。
2. 概率越高，峰值出现的越晚。

**基于以上推测，从我们的数据上来看，基本吻合：**
1. 随机平移概率为0时是极限情况，网络很快便进入了rank1是45%的情况，但是，继续训练，网络的rank1下滑至39%。
2. 随机平移概率为0.2时，rank1分别为：32，36，`41`，37。
3. 随机平移概率为0.4时，rank1分别为：42，`48`，44，38。
4. 随机平移概率为0.6时，rank1分别为：35，41，41，`46`。
5. 随机平移概率为0.8时，rank1分别为：34，36，38，39，`43`。
6. 随机平移概率为1.0时，rank1分别为：23，25，`27`，23，21。

`以上实验结果很清晰表明，当随机平移概率在0.6和0.8的时候，网络的rank1可能还会继续上升。`

#### 关于双输入CNN的ReID模型的训练策略的假设：
1. 我们认为，对于正负样本，均采用在线抽样，并且控制样本比例为1：1的时候，是目前为止，解决ReID领域正负样本比例悬殊的最有效方法。
2. 同时，用随机平移的方式是可以提升训练效果的，但是随机平移的概率不能为1，由于可能出现越接近于1的时候，训练效果越好的情况，我们大胆猜测，这个概率的极限值可能用如下公式计算：

$$ p=1-\dfrac{negative}{positive} $$

3. 后续我们会对fine-tune的效果进行实验。
4. 基于以上两点，双输入的ReID模型可能存在最佳训练策略，这个训练策略如果从理论上得到证明和分析及实验，可能会成为领域内的规则。