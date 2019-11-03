---
title: Python sklearn 实现过采样和欠采样
categories:
  - 机器学习
  - Python

tags:
  - Python
abbrlink: 33760
date: 2019-10-10 01:29:56
---
Imblearn package study
# 1. 准备知识
Sparse input
For sparse input the data is converted to the Compressed Sparse
Rows representation (see scipy.sparse.csr_matrix) before being fed
to the sampler. To avoid unnecessary memory copies, it is
recommended to choose the CSR representation upstream.
## 1.1 Compressed Sparse Rows(CSR) 压缩稀疏的行
稀疏矩阵中存在许多0元素, 按矩阵A进行存储会占用很大的空间(内存).
CSR方法采取按行压缩的办法, 将原始的矩阵用三个数组进行表示:
```python
1. data = np.array([1, 2, 3, 4, 5, 6])
2. indices = np.array([0, 2, 2, 0, 1, 2])
3. indptr = np.array([0, 2, 3, 6])
```
data数组: 存储着矩阵A中所有的非零元素;
indices数组: data数组中的元素在矩阵A中的列索引
indptr数组: 存储着矩阵A中每行第一个非零元素在data数组中的索引.
```python
1. from scipy import sparse
2. mtx = sparse.csr_matrix((data,indices,indptr),shape=(3,3))
3. mtx.todense()
4. 
5. Out[27]:
6. matrix([[1, 0, 2],
7.         [0, 0, 3],
8.         [4, 5, 6]])
```
为什么会有针对不平衡数据的研究? 当我们的样本数据中, 正负样本的数据占比
极其不均衡的时候, 模型的效果就会偏向于多数类的结果. 具体的可参照官网利
用支持向量机进行可视化不同正负样本比例情况下的模型分类结果.
# 2. 过采样(Over-sampling)
## 2.1 实用性的例子
### 2.1.1 朴素随机过采样
针对不平衡数据, 最简单的一种方法就是生成少数类的样本, 这其中最基本的一
种方法就是: 从少数类的样本中进行随机采样来增加新的样
本, RandomOverSampler 函数就能实现上述的功能.
```python 
1. from sklearn.datasets import make_classification
2. from collections import Counter
3. X, y = make_classification(n_samples=5000, n_features=2, n_informative=2,
4.                            n_redundant=0, n_repeated=0, n_classes=3,
5.                            n_clusters_per_class=1,
6.                            weights=[0.01, 0.05, 0.94],
7.                            class_sep=0.8, random_state=0)
8. Counter(y)
9. Out[10]: Counter({0: 64, 1: 262, 2: 4674})
10. 
11. from imblearn.over_sampling import RandomOverSampler
12. 
13. ros = RandomOverSampler(random_state=0)
14. X_resampled, y_resampled = ros.fit_sample(X, y)
15. 
16. 
17. sorted(Counter(y_resampled).items())
18. Out[13]:
19. [(0, 4674), (1, 4674), (2, 4674)]
```
以上就是通过简单的随机采样少数类的样本, 使得每类样本的比例为1:1:1.
## 2.1.2 从随机过采样到SMOTE与ADASYN
相对于采样随机的方法进行过采样, 还有两种比较流行的采样少数类的方法:
(i) Synthetic Minority Oversampling Technique (SMOTE); (ii) Adaptive
Synthetic (ADASYN) .
SMOTE: 对于少数类样本a, 随机选择一个最近邻的样本b, 然后从a与b的连线上随
机选取一个点c作为新的少数类样本;
ADASYN: 关注的是在那些基于K最近邻分类器被错误分类的原始样本附近生成新
的少数类样本
```python 
1. from imblearn.over_sampling import SMOTE, ADASYN
2. 
3. X_resampled_smote, y_resampled_smote = SMOTE().fit_sample(X, y)
4. 
5. sorted(Counter(y_resampled_smote).items())
6. Out[29]:
7. [(0, 4674), (1, 4674), (2, 4674)]
8. 
9. X_resampled_adasyn, y_resampled_adasyn = ADASYN().fit_sample(X, y)
10. 
11. sorted(Counter(y_resampled_adasyn).items())
12. Out[30]:
13. [(0, 4674), (1, 4674), (2, 4674)]
```
### 2.1.3 SMOTE的变体
相对于基本的SMOTE算法, 关注的是所有的少数类样本, 这些情况可能会导致产生
次优的决策函数, 因此SMOTE就产生了一些变体: 这些方法关注在最优化决策函数
边界的一些少数类样本, 然后在最近邻类的相反方向生成样本.
SMOTE函数中的kind参数控制了选择哪种变体, (i) borderline1, (ii) borderline2,
(iii) svm:
```python 
1. from imblearn.over_sampling import SMOTE, ADASYN
2. X_resampled, y_resampled = SMOTE(kind='borderline1').fit_sample(X, y)
3. 
4. print sorted(Counter(y_resampled).items())
5. Out[31]:
6. [(0, 4674), (1, 4674), (2, 4674)]
```
### 2.1.4 数学公式
SMOTE算法与ADASYN都是基于同样的算法来合成新的少数类样本: 对于少数类样本
a, 从它的最近邻中选择一个样本b, 然后在两点的连线上随机生成一个新的少数
类样本, 不同的是对于少数类样本的选择.
原始的SMOTE: kind='regular' , 随机选取少数类的样本.
The borderline SMOTE: kind='borderline1' or kind='borderline2'
此时, 少数类的样本分为三类: (i) 噪音样本(noise), 该少数类的所有最近邻样本
都来自于不同于样本a的其他类别; (ii) 危险样本(in danger), 至少一半的最近邻
样本来自于同一类(不同于a的类别); (iii) 安全样本(safe), 所有的最近邻样本都来
自于同一个类.
这两种类型的SMOTE使用的是危险样本来生成新的样本数据, 对于 Borderline-
1 SMOTE, 最近邻中的随机样本b与该少数类样本a来自于不同的类; 不同的是, 对
于 Borderline-2 SMOTE , 随机样本b可以是属于任何一个类的样本;
SVM SMOTE: kind='svm', 使用支持向量机分类器产生支持向量然后再生成新的
少数类样本.
# 3. 下采样(Under-sampling)
## 3.1 原型生成(prototype generation)
给定数据集S, 原型生成算法将生成一个子集S’, 其中|S’| < |S|, 但是子集并非
来自于原始数据集. 意思就是说: 原型生成方法将减少数据集的样本数量, 剩下的
样本是由原始数据集生成的, 而不是直接来源于原始数据集.
ClusterCentroids函数实现了上述功能: 每一个类别的样本都会用K-Means算法的
中心点来进行合成, 而不是随机从原始样本进行抽取.
```python
1. from imblearn.under_sampling import ClusterCentroids
2. 
3. cc = ClusterCentroids(random_state=0)
4. X_resampled, y_resampled = cc.fit_sample(X, y)
5. 
6. print sorted(Counter(y_resampled).items())
7. Out[32]:
8. [(0, 64), (1, 64), (2, 64)]
```
ClusterCentroids函数提供了一种很高效的方法来减少样本的数量, 但需要注意
的是, 该方法要求原始数据集最好能聚类成簇. 此外, 中心点的数量应该设置好,
这样下采样的簇能很好地代表原始数据.
## 3.2 原型选择(prototype selection)
与原型生成不同的是, 原型选择算法是直接从原始数据集中进行抽取. 抽取的方
法大概可以分为两类: (i) 可控的下采样技术(the controlled under-sampling
techniques) ; (ii) the cleaning under-sampling techniques(不好翻译, 就放
原文, 清洗的下采样技术?). 第一类的方法可以由用户指定下采样抽取的子集中样
本的数量; 第二类方法则不接受这种用户的干预.
### 3.2.1 Controlled under-sampling techniques
RandomUnderSampler函数是一种快速并十分简单的方式来平衡各个类别的数据:
随机选取数据的子集.
```python
1. from imblearn.under_sampling import RandomUnderSampler
2. rus = RandomUnderSampler(random_state=0)
3. X_resampled, y_resampled = rus.fit_sample(X, y)
4. 
5. print sorted(Counter(y_resampled).items())
6. Out[33]:
7. [(0, 64), (1, 64), (2, 64)]
```
通过设置RandomUnderSampler中的replacement=True参数, 可以实现自助法
(boostrap)抽样.
```python
1. import numpy as np
2. 
3. np.vstack({tuple(row) for row in X_resampled}).shape
4. Out[34]:
5. (192L, 2L)
```
很明显, 使用默认参数的时候, 采用的是不重复采样;
```python
1. from imblearn.under_sampling import RandomUnderSampler
2. rus = RandomUnderSampler(random_state=0, replacement=True)
3. X_resampled, y_resampled = rus.fit_sample(X, y)
4. 
5. print sorted(Counter(y_resampled).items())
6. Out[33]:
7. [(0, 64), (1, 64), (2, 64)]
8. 
9. np.vstack({tuple(row) for row in X_resampled}).shape
10. Out[34]:
11. (181L, 2L)
```
NearMiss函数则添加了一些启发式(heuristic)的规则来选择样本, 通过设定
version参数来实现三种启发式的规则.
```python
1. from imblearn.under_sampling import NearMiss
2. nm1 = NearMiss(random_state=0, version=1)
3. X_resampled_nm1, y_resampled = nm1.fit_sample(X, y)
4. 
5. print sorted(Counter(y_resampled).items())
6. Out[35]:
7. [(0, 64), (1, 64), (2, 64)]
```
下面通过一个例子来说明这三个启发式的选择样本的规则, 首先我们假设正样本
是需要下采样的(多数类样本), 负样本是少数类的样本.
NearMiss-1: 选择离N个近邻的负样本的平均距离最小的正样本;
NearMiss-2: 选择离N个负样本最远的平均距离最小的正样本;
NearMiss-3: 是一个两段式的算法. 首先, 对于每一个负样本, 保留它们的M个近
邻样本; 接着, 那些到N个近邻样本平均距离最大的正样本将被选择.
### 3.2.2 Cleaning under-sampling techniques
### 3.2.2.1 Tomek’s links
TomekLinks : 样本x与样本y来自于不同的类别, 满足以下条件, 它们之间被称之为
TomekLinks; 不存在另外一个样本z, 使得d(x,z) < d(x,y) 或者 d(y,z) < d(x,y)成
立. 其中d(.)表示两个样本之间的距离, 也就是说两个样本之间互为近邻关系. 这
个时候, 样本x或样本y很有可能是噪声数据, 或者两个样本在边界的位置附近.
TomekLinks函数中的auto参数控制Tomek’s links中的哪些样本被剔除. 默认的
ratio='auto' 移除多数类的样本, 当ratio='all'时, 两个样本均被移除.
#### 3.2.2.2 Edited data set using nearest neighbours
EditedNearestNeighbours这种方法应用最近邻算法来编辑(edit)数据集, 找出那
些与邻居不太友好的样本然后移除. 对于每一个要进行下采样的样本, 那些不满
足一些准则的样本将会被移除; 他们的绝大多数(kind_sel='mode')或者全部
(kind_sel='all')的近邻样本都属于同一个类, 这些样本会被保留在数据集中.
```python
1. print sorted(Counter(y).items())
2. 
3. from imblearn.under_sampling import EditedNearestNeighbours
4. enn = EditedNearestNeighbours(random_state=0)
5. X_resampled, y_resampled = enn.fit_sample(X, y)
6. 
7. print sorted(Counter(y_resampled).items())
8. 
9. Out[36]:
10. [(0, 64), (1, 262), (2, 4674)]
11. [(0, 64), (1, 213), (2, 4568)]
```
在此基础上, 延伸出了RepeatedEditedNearestNeighbours算法, 重复基础的
EditedNearestNeighbours算法多次.
```python
1. from imblearn.under_sampling import RepeatedEditedNearestNeighbours
2. renn = RepeatedEditedNearestNeighbours(random_state=0)
3. X_resampled, y_resampled = renn.fit_sample(X, y)
4. 
5. print sorted(Counter(y_resampled).items())
6. Out[37]:
7. [(0, 64), (1, 208), (2, 4551)]
```
与RepeatedEditedNearestNeighbours算法不同的是, ALLKNN算法在进行每次迭代
的时候, 最近邻的数量都在增加.
```python
1. from imblearn.under_sampling import AllKNN
2. allknn = AllKNN(random_state=0)
3. X_resampled, y_resampled = allknn.fit_sample(X, y)
4. 
5. print sorted(Counter(y_resampled).items())
6. Out[38]:
7. [(0, 64), (1, 220), (2, 4601)]
```
#### 3.2.2.3 Condensed nearest neighbors and derived algorithms
CondensedNearestNeighbour 使用1近邻的方法来进行迭代, 来判断一个样本是应
该保留还是剔除, 具体的实现步骤如下:
1. 集合C: 所有的少数类样本;
2. 选择一个多数类样本(需要下采样)加入集合C, 其他的这类样本放入集合
S;
3. 使用集合S训练一个1-NN的分类器, 对集合S中的样本进行分类;
4. 将集合S中错分的样本加入集合C;
5. 重复上述过程, 直到没有样本再加入到集合C.
```python
1. from imblearn.under_sampling import CondensedNearestNeighbour
2. cnn = CondensedNearestNeighbour(random_state=0)
3. X_resampled, y_resampled = cnn.fit_sample(X, y)
4. 
5. print sorted(Counter(y_resampled).items())
6. Out[39]:
7. [(0, 64), (1, 24), (2, 115)]
```
显然, CondensedNearestNeighbour方法对噪音数据是很敏感的, 也容易加入噪音
数据到集合C中.
因此, OneSidedSelection 函数使用 TomekLinks 方法来剔除噪声数据(多数类样
本).
```python
1. from imblearn.under_sampling import OneSidedSelection
2. oss = OneSidedSelection(random_state=0)
3. X_resampled, y_resampled = oss.fit_sample(X, y)
4. 
5. print(sorted(Counter(y_resampled).items()))
6. Out[39]:
7. [(0, 64), (1, 174), (2, 4403)]
```
NeighbourhoodCleaningRule 算法主要关注如何清洗数据而不是筛选
(considering)他们. 因此, 该算法将使用
EditedNearestNeighbours和 3-NN分类器结果拒绝的样本之间的并集.
```python
1. from imblearn.under_sampling import NeighbourhoodCleaningRule
2. ncr = NeighbourhoodCleaningRule(random_state=0)
3. X_resampled, y_resampled = ncr.fit_sample(X, y)
4. 
5. print sorted(Counter(y_resampled).items())
6. Out[39]:
7. [(0, 64), (1, 234), (2, 4666)]
```
#### 3.2.2.4 Instance hardness threshold
InstanceHardnessThreshold是一种很特殊的方法, 是在数据上运用一种分类器,
然后将概率低于阈值的样本剔除掉.
```python
1. from sklearn.linear_model import LogisticRegression
2. from imblearn.under_sampling import InstanceHardnessThreshold
3. iht = InstanceHardnessThreshold(random_state=0,
4.                                 estimator=LogisticRegression())
5. X_resampled, y_resampled = iht.fit_sample(X, y)
6. 
7. print(sorted(Counter(y_resampled).items()))
8. Out[39]:
9. [(0, 64), (1, 64), (2, 64)]
```
# 4. 过采样与下采样的结合
在之前的SMOTE方法中, 当由边界的样本与其他样本进行过采样差值时, 很容易生
成一些噪音数据. 因此, 在过采样之后需要对样本进行清洗. 这样, 第三节中涉及
到的TomekLink 与 EditedNearestNeighbours方法就能实现上述的要求. 所以就
有了两种结合过采样与下采样的方法: (i) SMOTETomek and (ii) SMOTEENN.
```python
1. from imblearn.combine import SMOTEENN
2. smote_enn = SMOTEENN(random_state=0)
3. X_resampled, y_resampled = smote_enn.fit_sample(X, y)
4. 
5. print sorted(Counter(y_resampled).items())
6. Out[40]:
7. [(0, 4060), (1, 4381), (2, 3502)]
8. 
9. 
10. from imblearn.combine import SMOTETomek
11. smote_tomek = SMOTETomek(random_state=0)
12. X_resampled, y_resampled = smote_tomek.fit_sample(X, y)
13. 
14. print sorted(Counter(y_resampled).items())
15. Out[40]:
16. [(0, 4499), (1, 4566), (2, 4413)]
```
# 5. Ensemble的例子
## 5.1 例子
一个不均衡的数据集能够通过多个均衡的子集来实现均衡, imblearn.ensemble模
块能实现上述功能.
EasyEnsemble 通过对原始的数据集进行随机下采样实现对数据集进行集成.
```python
1. from imblearn.ensemble import EasyEnsemble
2. ee = EasyEnsemble(random_state=0, n_subsets=10)
3. X_resampled, y_resampled = ee.fit_sample(X, y)
4. 
5. print X_resampled.shape
6. print sorted(Counter(y_resampled[0]).items())
7. Out[40]:
8. (10L, 192L, 2L)
9. [(0, 64), (1, 64), (2, 64)]
```
EasyEnsemble 有两个很重要的参数: (i) n_subsets 控制的是子集的个数 and
(ii) replacement 决定是有放回还是无放回的随机采样.
与上述方法不同的是, BalanceCascade(级联平衡)的方法通过使用分类器
(estimator参数)来确保那些被错分类的样本在下一次进行子集选取的时候也能
被采样到. 同样, n_max_subset 参数控制子集的个数, 以及可以通过设置
bootstrap=True来使用bootstraping(自助法).
```python
1. from imblearn.ensemble import BalanceCascade
2. from sklearn.linear_model import LogisticRegression
3. bc = BalanceCascade(random_state=0,
4.                     estimator=LogisticRegression(random_state=0),
5.                     n_max_subset=4)
6. X_resampled, y_resampled = bc.fit_sample(X, y)
7. 
8. print X_resampled.shape
9. 
10. print sorted(Counter(y_resampled[0]).items())
11. Out[41]:
12. (4L, 192L, 2L)
13. [(0, 64), (1, 64), (2, 64)]
```
## 5.2 Chaining ensemble of samplers and
estimators
在集成分类器中, 装袋方法(Bagging)在不同的随机选取的数据集上建立了多个
估计量. 在scikit-learn中这个分类器叫做BaggingClassifier. 然而, 该分类器并
不允许对每个数据集进行均衡. 因此, 在对不均衡样本进行训练的时候, 分类器其
实是有偏的, 偏向于多数类.
```python
1. from sklearn.model_selection import train_test_split
2. from sklearn.metrics import confusion_matrix
3. from sklearn.ensemble import BaggingClassifier
4. from sklearn.tree import DecisionTreeClassifier
5. 
6. X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=0)
7. bc = BaggingClassifier(base_estimator=DecisionTreeClassifier(),
8.                        random_state=0)
9. bc.fit(X_train, y_train)
10. 
11. y_pred = bc.predict(X_test)
12. confusion_matrix(y_test, y_pred)
13. Out[35]:
14. array([[   0,    0,   12],
15.        [   0,    0,   59],
16.        [   0,    0, 1179]], dtype=int64)
```
BalancedBaggingClassifier 允许在训练每个基学习器之前对每个子集进行重抽
样. 简而言之, 该方法结合了EasyEnsemble采样器与分类器(如
BaggingClassifier)的结果.
```python
1. from imblearn.ensemble import BalancedBaggingClassifier
2. bbc = BalancedBaggingClassifier(base_estimator=DecisionTreeClassifier(),
3.                                 ratio='auto',
4.                                 replacement=False,
5.                                 random_state=0)
6. bbc.fit(X, y)
7. 
8. y_pred = bbc.predict(X_test)
9. confusion_matrix(y_test, y_pred)
10. Out[39]:
11. array([[  12,    0,    0],
12.        [   0,   55,    4],
13.        [  68,   53, 1058]], dtype=int64)
```
# 6. 数据载入
imblearn.datasets 包与sklearn.datasets 包形成了很好的互补. 该包主要有以
下两个功能: (i)提供一系列的不平衡数据集来实现测试; (ii) 提供一种工具将原始
的平衡数据转换为不平衡数据.
## 6.1 不平衡数据集
fetch_datasets 允许获取27个不均衡且二值化的数据集.
```python
1. from collections import Counter
2. from imblearn.datasets import fetch_datasets
3. ecoli = fetch_datasets()['ecoli']
4. ecoli.data.shape
5. 
6. print sorted(Counter(ecoli.target).items())
7. Out[40]:
8. [(-1, 301), (1, 35)]
```
## 6.2 生成不平衡数据
make_imbalance 方法可以使得原始的数据集变为不平衡的数据集, 主要是通过
ratio参数进行控制.
```python
1. from sklearn.datasets import load_iris
2. from imblearn.datasets import make_imbalance
3. iris = load_iris()
4. ratio = {0: 20, 1: 30, 2: 40}
5. X_imb, y_imb = make_imbalance(iris.data, iris.target, ratio=ratio)
6. 
7. sorted(Counter(y_imb).items())
8. Out[37]:
9. [(0, 20), (1, 30), (2, 40)]
10. 
11. #当类别不指定时, 所有的数据集均导入
12. ratio = {0: 10}
13. X_imb, y_imb = make_imbalance(iris.data, iris.target, ratio=ratio)
14. 
15. sorted(Counter(y_imb).items())
16. Out[38]:
17. [(0, 10), (1, 50), (2, 50)]
18. 
19. #同样亦可以传入自定义的比例函数
20. def ratio_multiplier(y):
21.     multiplier = {0: 0.5, 1: 0.7, 2: 0.95}
22.     target_stats = Counter(y)
23.     for key, value in target_stats.items():
24.         target_stats[key] = int(value * multiplier[key])
25.     return target_stats
26. X_imb, y_imb = make_imbalance(iris.data, iris.target,
27.                               ratio=ratio_multiplier)
28. 
29. sorted(Counter(y_imb).items())
30. Out[39]:
31. [(0, 25), (1, 35), (2, 47)]
```
以上就是在研究不平衡(不均衡)数据时所查询的一些资料, 内容更多的是来自于
Imblearn包的官方用户手册, 主要涉及一些下采样、过采样的方法与技术.
