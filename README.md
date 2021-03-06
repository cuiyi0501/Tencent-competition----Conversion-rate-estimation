# Tencent-competition----Conversion-rate-estimation
## 目标
以移动app广告为研究对象，根据给定广告、用户和上下文情况信息，预测app广告点击后被激活的概率。评估方式是交叉熵。
## 实验数据介绍
![dataset](https://github.com/Maggione/Tencent-competition----Conversion-rate-estimation/raw/master/dataset.jpeg) 
特别说明：
由于转化需要一定时间，训练集最后几天的数据的标签可能不准确，但是测试集的标签是准确的。
这里提供的数据是缩减版，真实数据集大小是：train.csv（37912917）  test.csv（3321749）
## 代码说明
特征提取<br>
> cd pre_script <br>
> sh pre.sh <br>

模型训练<br>
> python lgb.28-29.py

实验结果：单模型测试集B榜最终得分是0.102092，排名35<br>
## 实验流程
### 数据清洗
#### 重复点击数据
初赛时时间信息不够精确，导致训练数据中出现“除了label不同，其他信息均相同”的情况，复赛虽然精确了时间，仍然存在少部分这种情况。<br> 
注意到测试集中也存在重复点击数据，直接删除训练集中的重复样本显然不可取。统计发现大部分的转化存在于重复点击数据中。对于这部分数据的挖掘后面特征提取细讲。<br> 
#### 30号的label不准确
由于转化需要一定的时间，据统计转化时间大部分为1天内，导致30号样本的标不准确问题比较严重。直接弃用30号数据也是不行的，因为在31号中存在appID仅在30号数据中出现的情况。<br> 
于是应该找出30号数据中不可信的部分的数据，有两种方法：a. 删除转化时间长的数据；b. 删除30号数据中转化率远低于以往转化率的数据。我采用的是第二种。<br> 
### 数据集划分
初赛数据较小，故全部用上；复赛数据量大，只用了最后三天的训练做训练。找到和测试集效果一致的验证集尤为重要，我以30号为验证集，实验效果与测试集一致。<br> 
（1）注意到这个比赛的数据是有时间顺序，应该严格按照时间顺序切分训练集和验证集。<br> 
（2）采用的随机切分，交叉验证的方式。实验发现效果不错，而且由于多组实验结果去平均得到最基本的融合效果，实验效果更佳。不过，仅适用于本次比赛数据。<br> 
### 实验模型
本次比赛只使用了GBDT模型，初赛时使用的是xgboost模型，决赛的时候由于数据量太大，于是使用LightGBM，速度明显提高。<br> 
### 特征工程
#### 基础特征
广告类ID，上下文ID，app类ID，用户ID，时间（这是一个重要特征）<br> 
本次比赛的数据基本上都是ID特征，做one-hot处理数据量太大，可以用hash处理。<br> 
还有一些连续的特征，除了gbdt模型，应该注意进行均值归一化操作，同时可以进行分箱操作的特征工程。<br> 
#### 历史统计特征
注意到这个比赛的数据是有时间顺序，所以在做统计的时候应该严格考虑时间顺序，防止出现数据泄漏的情况。不然，会导致验证集的效果和测试集效果不一致的情况。<br> 
考虑时间顺序统计方法有三种：<br> 
a. 固定窗口统计特征：比如一直取第17-23天的数据进行统计，之后的数据进行训练。统一基数，更稳定一些，时效性差。<br> 
b. 滑动窗口统计特征：取该天前7天的数据进行统计。考虑时效性。<br> 
c. 变化窗口统计特征：取该天之前的所有数据进行统计。时效性较弱，数据量大，覆盖更全面。<br> 
本次比赛，根据特征的时效性需求，选择使用b或c，比如上下文特征的时效性可能相对要求较低等，具体看实验结果反馈。<br> 
历史统计特征主要分为三类：点击量、转化数、转化率<br> 
点击量和转化数：由于采用滑窗或者变化窗都会导致特征基数变化，所以我们采用比率形式进行统计，即feature_a点击量／窗口内总点击量。<br> 
转化率的平滑问题：极端情况下，广告只被点击1此并转化了，则其转化率计算为1，这是不合理的。平滑方式采用了贝叶斯平滑方法。<br> 
#### 组合特征
采用笛儿卡积，把两个或多个特征进行组合成为一个新的特征。尽管xgboost能够挖掘组合特征，但是人为引入先验信息可以使模型效果更好。<br> 
筛选特征的方法：<br> 
a. 可以使用xgboost进行特征重要性排序，特征的全局重要度通过计算特征在单颗树中的重要度的平均值来衡量。<br> 
b. 可以计算特征的方差，如果方差大的话，可能特征更有用。<br> 
#### 重复点击数据
如上文提到的，本次比赛的数据有一个特点就是出现了大量重复数据，并且测试集也出现了这种情况。同时注意到数据是按时间顺序排列的。对于重复点击数据的挖掘有三个方面：<br> 
a. 标记区分重复数据和不重复数据<br> 
b. 标记重复数据的次序<br> 
c. 重复数据间的时间差，和第一条数据和最后一条数据的时间差<br> 
同时观察数据，发现重复数据的点击和appID已经positionID有关系，于是对其进行特征组合。注意区分重不重复很重要，因为第一个点击并不知道后面是否重复，其实这里有一些信息泄漏，但是测试集中这部分泄漏的信息也是可以得到的，所以必须是当天内是否重复。<br> 
