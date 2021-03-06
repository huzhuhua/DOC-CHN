# 联邦建模过程分析
## 1.联邦建模过程信息交互
在纵向联邦学习的三方（arbiter，guest，host）架构中，涉及到数据相关的信息交互有五个环节：加密样本ID匹配对齐(RSA)、加密样本ID匹配对齐(RAW)、特征工程-分箱标签加密、纵向模型训练过程、横向模型训练过程。
- 加密样本ID匹配对齐(RSA)环节交互的是加密后的用户粒度的ID信息
【加密算法选择RSA和哈希，加密样本对齐后，guest和host只能得知对齐的样本ID，无法得知非对齐部分的对方样本ID信息】
- 加密样本ID匹配对齐(RAW)环节交互的是加密后的用户粒度的ID信息
【加密算法选择哈希，取交集的一方在过程中自方的ID没有出本地，提高和保障了安全性】
- 特征工程-分箱标签加密环节交互的是模型的样本个数和标签信息
【加密算法选择Paillier同态加密，双方之间不传输原始的业务数据，仅传输加密后的样本个数及标签】
- 纵向模型训练过程
【加密算法选择Paillier同态加密，由arbiter中立方生成秘钥对，将公钥发送给guest和host，私钥在arbiter方。信息交互中，guest和host将用公钥加密后的信息传输给arbiter，arbiter用私钥解密后，将聚合优化后的梯度分别传输给guest和host，guest和host更新模型参数后，继续模型训练。循环以上步骤，直至loss达到预期或者训练次数达到设置的最大值时完成模型训练。在训练的过程中，双方都不知道对方的数据结构，并且只能获得自己那一部分特征需要的参数。所以并没有直接传递数据相关的信息，它们间的通信也是安全的】
- 横向模型训练过程
【加密算法选择Paillier同态加密，双方原始数据信息不进行传输，仅传输模型的参数，arbiter获取不到两方的业务数据信息】
所有交互的信息均在加密情况下进行，解密的私钥在arbiter中立方，arbiter中立方仅获取guest和host双方的模型参数信息，获取不到双方的原始数据，保证信息交互的安全性。

## 全程三方示例流程图：
为了使大家更形象的理解整个信息交互流程，下面以比较通俗易懂的方式【详细的加密过程解析在后文会具体阐述】进行流程图展示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200819175731118.png)
## 2. 加密样本ID匹配对齐（RSA）
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020081815123983.png)

RSA+哈希【A方为GUEST,B方为HOST】：

第一步： B通过 RSA算法产生公私钥，将公钥（n,e）发给 A, A对于每条样本产生一个随机数ri,ri的e次方乘以A方的ID的哈希值【作为第一把锁】
第二步： A 将加密后的结果发给 B
第三步： B方将 A方的结果取d次幂【作为第二把锁，其中等式左边根据RSA原理省略了个%n，最后化为等式右边的ri】，将B方的ID先做一次哈希加密再做RSA加密最后再做一次哈希加密【其中取完d次幂后面还有个%n【由于是RSA原理故官方在图中未写明，其实是没错的哈】然后把加密后的 A, B方ID传给 A【之所以需要互传是因为双方需要统一为H(H(u))形式后取交集，相同的ID得到的H(H(u))结果一样】
第四步： A将获得的 A值除以前面产生的随机数ri后进行哈希加密与获得的B值取交集，得到交集的ID再传给B。【其中A并不需要对从B获得的值进行解密，由于A、B两方在传递时ID的顺序没变,所以传递后A、B方会知道特定的位置应该对应哪个ID】
选用哈希的原因：由于ID不一定是纯数字可能是字符串，需通过哈希将其转化为整数.
## 3.加密样本ID匹配对齐（RAW）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200818151314378.png)
【A.B方可以根据算法参数设置哪方为取交集方和传输方】

A：取交集方(ID不出库)  B:传输方（ID出库）
IDA:{X1、X2、X3、X4}和IDB:{X1、X2、X5、X6} 
第一步：B方将ID取哈希值【MD5,SHA256 等哈希方式】，将加密结果传给A
第二步：A方将己方ID取哈希值【MD5,SHA256 等哈希方式】，和B方的加密结果取交集，后将对齐结果发还给B
* 全程B方为取交集的一方在过程中自方的ID没有出本地，提高和保障了安全性。B方将取交后的结果{X1、X2}同步发给A方。
## 4. 联邦特征分箱标签加密
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200819154619933.png)
[A方为HOST,B方为GUEST]

* 特征分箱的目的是计算IV值，IV值是用来衡量这个特征对标签y的贡献程度，辅助于特征选择
>在纵向中A侧（含有X）,B侧（含X、Y）计算WOE和IV
由于A侧只有特征X,没有Y,计算WOE和IV得同时依赖X、Y，
A侧不对B侧暴露X，B侧不对A暴露Y,
最终只能让B侧获得所有特征的WOE&IV

第一步：B侧对标签Y通过同态加密方式加密【Paillier加密具有混淆项，即同样的数字0经过加密后结果不一样，因此不能通过密文推出原来的标签为0还是1】
第二步：A侧有自己特征的分箱信息，即知道每个分箱中的ID信息，通过B侧传来的加密标签Y后获得0、1的个数，再将每个分箱中0,1的个数发给B侧
第三步：B侧解密后知道0、1个数，进而求得WOE和IV值。再反馈给host方列名【guest方获取的host方列名为匿名的形式例如Host_0这种，Host依次获知哪些特征是需要保留的。】

## 5.纵向模型训练过程梯度传递
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200818151403995.png)

官方书籍对线性回归进行了阐述（《联邦学习》77页到80页），故在此对逻辑回归进行解释：


#### 逻辑回归
总体上，加密训练过程从分发公钥到更新模型分为四步。
【1,2步包含ID，3.4步不包含】

**通过打印日志和源码以及论文A Quasi-Newton Method Based Vertical Federated 获知：**
>之所以要将损失函数通过泰勒展开式展开是由于加密算法目前支持加法和数乘同态，对Log不支持，故采用泰勒展开，式子变为只有加法和乘法

推导过程中需要用的公式如下：
|  符号   | 含义 |
|  ----  | ----  |
|![在这里插入图片描述](https://img-blog.csdnimg.cn/20200818151937806.png#pic_center) |加密输出
  |![在这里插入图片描述](https://img-blog.csdnimg.cn/20200818152139932.png#pic_center)|![在这里插入图片描述](https://img-blog.csdnimg.cn/20200818152050715.png#pic_center)
| ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200818152156141.png#pic_center)|![在这里插入图片描述](https://img-blog.csdnimg.cn/20200818152245501.png#pic_center)
|loss|![在这里插入图片描述](https://img-blog.csdnimg.cn/20200818152406760.png#pic_center)
|梯度g|![在这里插入图片描述](https://img-blog.csdnimg.cn/20200818152453442.png#pic_center)
|残差d|![在这里插入图片描述](https://img-blog.csdnimg.cn/20200818152618300.png#pic_center)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200819154823654.png)


* 第一步A方发给B方的信息以一个array的形式发送，其中包括明文ID,一个ID对应一个中间聚合值。
* 第二步B方发给A方的残差也是以一个array的形式发送，其中包括明文ID,一个ID对应一个残差值。
* 第三步B方发给C方的损失以一个累加损失值的形式发送（对整份样本求得的一个累加损失值）
【C方优化方式参考FATE源码federatedml\optim\optimizer.py 】
以下展示的是nesterov_momentum_sgd优化方法：
```bash
class _NesterovMomentumSGDOpimizer(_Optimizer):
    def __init__(self, learning_rate, alpha, penalty, decay, decay_sqrt):
        super().__init__(learning_rate, alpha, penalty, decay, decay_sqrt)
        self.nesterov_momentum_coeff = 0.9
        self.opt_m = None

    def apply_gradients(self, grad):
        learning_rate = self.decay_learning_rate()

        if self.opt_m is None:
            self.opt_m = np.zeros_like(grad)
        v = self.nesterov_momentum_coeff * self.opt_m - learning_rate * grad
        delta_grad = self.nesterov_momentum_coeff * self.opt_m - (1 + self.nesterov_momentum_coeff) * v
        self.opt_m = v
        LOGGER.debug('In nesterov_momentum, opt_m: {}, v: {}, delta_grad: {}'.format(
            self.opt_m, v, delta_grad
        ))
        return delta_grad
```

>在整个模型训练过程中交互的信息主要为模型训练的中间结果信息：梯度（gradient）和损耗（loss）。交互的梯度和loss是模型参数维度的信息。加密算法选择Paillier同态加密。由arbiter中立方生成秘钥对，将公钥发送给guest和host，私钥在arbiter方。信息交互中，guest和host将用公钥加密后的信息传输给arbiter，arbiter用私钥解密后，将聚合优化后的梯度分别传输给guest和host，guest和host更新模型参数后，继续模型训练。循环以上步骤，直至loss达到预期或者训练次数达到设置的最大值时完成模型训练。在训练的过程中，A 和 B 都不知道对方的数据结构，并且只能获得自己那一部分特征需要的参数。所以 A 和 B 之间并没有直接传递数据相关的信息，它们间的通信也是安全的。
### 6.横向模型训练过程
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200818152932869.png)

横向训练分为梯度平均和模型平均两种，Fate源码中用的为模型平均

#### 梯度平均：
第一步：各参与方在本地计算模型梯度，对梯度信息进行加密后发送给聚合服务器（arbiter）
第二步：聚合服务器进行安全聚合（secure aggregation）操作，将梯度进行加权平均计算
第三步：聚合服务器将聚合后的结果发给各参与方
第四步：各参与方根据获得的聚合结果进行解密并更新自方模型参数
上述步骤将会持续迭代进行，直到损失函数收敛或者达到允许的迭代次数上限
#### 模型平均：
第一步：各参与方在本地计算模型参数和损失以及己方行数，对信息进行加密后发送给聚合服务器（arbiter）
第二步：聚合服务器进行安全聚合（secure aggregation）操作，将模型参数运用两方行数进行加权平均计算
第三步：聚合服务器将聚合后的模型参数结果发给各参与方
第四步：各参与方根据获得的聚合结果进行解密后更新自方模型参数
上述步骤将会持续迭代进行，直到损失函数收敛或者达到允许的迭代次数上限


