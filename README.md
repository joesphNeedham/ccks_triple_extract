### 数据集下载网址：http://lic2019.ccf.org.cn/kg 。

&emsp;&emsp;本文将会介绍笔者在2019语言与智能技术竞赛的三元组抽取比赛方面的一次尝试。由于该比赛早已结束，笔者当时也没有参加这个比赛，因此没有测评成绩，我们也只能拿到训练集和验证集。但是，这并不耽误我们在这方面做实验。

### 比赛介绍
&emsp;&emsp;该比赛的网址为：[http://lic2019.ccf.org.cn/kg](http://lic2019.ccf.org.cn/kg) ，该比赛主要是从给定的句子中提取三元组，给定schema约束集合及句子sent，其中schema定义了关系P以及其对应的主体S和客体O的类别，例如（S_TYPE:人物，P:妻子，O_TYPE:人物）、（S_TYPE:公司，P:创始人，O_TYPE:人物）等。比如下面的例子：

```
{
  "text": "九玄珠是在纵横中文网连载的一部小说，作者是龙马",
  "spo_list": [
    ["九玄珠", "连载网站", "纵横中文网"],
    ["九玄珠", "作者", "龙马"]
  ]
}
```
该比赛一共提供了20多万标注质量很高的三元组，其中17万训练集，2万验证集和2万测试集，实体关系（schema）50个。

&emsp;&emsp;在具体介绍笔者的思路和实战前，先介绍下本次任务的处理思路：
![本次人物的总体思路](https://img-blog.csdnimg.cn/20200315105912758.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2pjbGlhbjkx,size_16,color_FFFFFF,t_70)
首先是对拿到的数据进行数据分析，包括统计每个句子的长度及三元组数量，每种关系的数量分布情况。接着，对数据单独走序列标注模型和关系分析模型。最后在提取三元组的时候，用Pipeline模型，先用序列标注模型预测句子中的实体，再对实体（加上句子）走关系分类模型，预测实体的关系，最后形成有效的三元组。

&emsp;&emsp;接下来笔者将逐一介绍，项目结构图如下：
![项目结构图](https://img-blog.csdnimg.cn/20200315112107112.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2pjbGlhbjkx,size_16,color_FFFFFF,t_70)

### 数据分析
&emsp;&emsp;我们能拿到的只有训练集和验证集，没有测试集。我们对训练集做数据分析，训练集数据文件为train_data.json。

&emsp;&emsp;数据分析会统计训练集中每个句子的长度及三元组数量，还有关系的分布图，代码为data_analysis/train_data_analysis.py 。

输出结果如下：
```
             spo_num    text_length
count  173108.000000  173108.000000
mean        2.103993      54.057190
std         1.569331      31.498245
min         0.000000       5.000000
25%         1.000000      32.000000
50%         2.000000      45.000000
75%         2.000000      68.000000
max        25.000000     300.000000
```
句子的平均长度为54，最大长度为300；每句话中的三元组数量的平均值为2.1，最大值为25。
&emsp;&emsp;每句话中的三元组数量的分布图如下：![每句话中的三元组数量分布图](https://img-blog.csdnimg.cn/20200315112630155.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2pjbGlhbjkx,size_16,color_FFFFFF,t_70)
&emsp;&emsp;关系数量的分布图如下：
![关系数量分布图](https://img-blog.csdnimg.cn/20200315113529945.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2pjbGlhbjkx,size_16,color_FFFFFF,t_70)
### 序列标注模型
&emsp;&emsp;我们将句子中的主体和客体作为实体，分别标注为SUBJ和OBJ，标注体系采用BIO。一个简单的标注例子如下：
```
如 O
何 O
演 O
好 O
自 O
己 O
的 O
角 O
色 O
， O
请 O
读 O
《 O
演 O
员 O
自 O
我 O
修 O
养 O
》 O
《 O
喜 B-SUBJ
剧 I-SUBJ
之 I-SUBJ
王 I-SUBJ
》 O
周 B-OBJ
星 I-OBJ
驰 I-OBJ
崛 O
起 O
于 O
穷 O
困 O
潦 O
倒 O
之 O
中 O
的 O
独 O
门 O
秘 O
笈 O
```
&emsp;&emsp;序列标注的模型采用ALBERT+Bi-LSTM+CRF，结构图如下：
![序列标注模型结构图](https://img-blog.csdnimg.cn/20200315113946554.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2pjbGlhbjkx,size_16,color_FFFFFF,t_70)
模型方面的代码不再具体给出，有兴趣的同学可以参考文章[NLP（二十五）实现ALBERT+Bi-LSTM+CRF模型](https://blog.csdn.net/jclian91/article/details/104826655)，也可以参考文章最后给出的Github项目网址。

&emsp;&emsp;模型设置文本最大长度为128，利用ALBERT做特征提取，在自己的电脑上用CPU训练5个epoch，结果如下：
```
_________________________________________________________________
Train on 173109 samples, validate on 21639 samples
Epoch 1/5
173109/173109 [==============================] - 3275s 19ms/step - loss: 0.1269 - crf_viterbi_accuracy: 0.9417 - val_loss: 0.0251 - val_crf_viterbi_accuracy: 0.9613
Epoch 2/5
173109/173109 [==============================] - 3252s 19ms/step - loss: -0.0192 - crf_viterbi_accuracy: 0.9623 - val_loss: -0.0612 - val_crf_viterbi_accuracy: 0.9638
Epoch 3/5
173109/173109 [==============================] - 3445s 20ms/step - loss: -0.1040 - crf_viterbi_accuracy: 0.9644 - val_loss: -0.1450 - val_crf_viterbi_accuracy: 0.9649
Epoch 4/5
173109/173109 [==============================] - 3363s 19ms/step - loss: -0.1869 - crf_viterbi_accuracy: 0.9655 - val_loss: -0.2269 - val_crf_viterbi_accuracy: 0.9652
Epoch 5/5
173109/173109 [==============================] - 3266s 19ms/step - loss: -0.2694 - crf_viterbi_accuracy: 0.9662 - val_loss: -0.3088 - val_crf_viterbi_accuracy: 0.9651
           precision    recall  f1-score   support

      OBJ     0.9591    0.8870    0.9216     40844
     SUBJ     0.9621    0.9202    0.9407     25252

micro avg     0.9603    0.8997    0.9290     66096
macro avg     0.9602    0.8997    0.9289     66096
```
利用seqeval模块做评估，在验证集上的F1值约为93%。

### 关系分类模型
&emsp;&emsp;需要对关系做一下说明，因为笔者会对句子（sent）中的主体（S）和客体（O）组合起来，加上句子，形成训练数据。举个例子，在句子`历史评价李氏朝鲜的创立并非太祖大王李成桂一人之功﹐其五子李芳远功不可没`，三元组为`[{"predicate": "父亲", "object_type": "人物", "subject_type": "人物", "object": "李成桂", "subject": "李芳远"}, {"predicate": "国籍", "object_type": "国家", "subject_type": "人物", "object": "朝鲜", "subject": "李成桂"}]}`，在这句话中主体有李成桂，李芳远，客体有李成桂和朝鲜，关系有父亲（关系类型：2）和国籍（关系类型：22）。按照笔者的思路，这句话应组成4个关系分类样本，如下：

```
2 李芳远$李成桂$历史评价李氏朝鲜的创立并非太祖大王###一人之功﹐其五子###功不可没
0 李芳远$朝鲜$历史评价李氏##的创立并非太祖大王李成桂一人之功﹐其五子###功不可没
0 李成桂$李成桂$历史评价李氏朝鲜的创立并非太祖大王###一人之功﹐其五子李芳远功不可没
22 李成桂$朝鲜$历史评价李氏##的创立并非太祖大王###一人之功﹐其五子李芳远功不可没
```
因此，就会出现关系0（表示“未知”），这样我们在提取三元组的时候就可以略过这条关系，形成真正有用的三元组。

&emsp;&emsp;因此，关系一共为51个（加上未知关系：0）。关系分类模型采用ALBERT+Bi-GRU+ATT，结构图如下：
![关系分类模型图](https://img-blog.csdnimg.cn/20200315115506974.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2pjbGlhbjkx,size_16,color_FFFFFF,t_70)

&emsp;&emsp;模型方面的代码不再具体给出，有兴趣的同学可以参考文章[NLP（二十一）人物关系抽取的一次实战](https://blog.csdn.net/jclian91/article/details/104380371)，也可以参考文章最后给出的Github项目网址。

&emsp;&emsp;模型设置文本最大长度为128，利用ALBERT做特征提取，在自己的电脑上用CPU训练30个epoch（实际上，由于有early stopping机制，训练不到30个eopch），在验证集上的评估结果如下：

```
				precision    recall  f1-score   support

          未知       0.84      0.77      0.80      5057
          祖籍       0.89      0.77      0.82       181
          父亲       0.82      0.81      0.82       609
        总部地点       0.95      0.95      0.95       310
         出生地       0.96      0.94      0.95      2330
           目       1.00      1.00      1.00      1271
          面积       0.84      0.94      0.89        79
          简称       0.99      0.99      0.99       138
        上映时间       0.95      0.97      0.96       463
          妻子       0.88      0.85      0.87       680
        所属专辑       0.97      0.97      0.97      1282
        注册资本       1.00      1.00      1.00        63
          首都       0.92      0.94      0.93        47
          导演       0.92      0.93      0.92      2603
           字       0.96      0.94      0.95       339
          身高       0.99      0.97      0.98       393
        出品公司       0.94      0.96      0.95       851
        修业年限       1.00      1.00      1.00         2
        出生日期       0.99      0.99      0.99      2892
         制片人       0.76      0.74      0.75       127
          母亲       0.76      0.82      0.79       425
          编剧       0.78      0.79      0.79       771
          国籍       0.89      0.96      0.92      1621
          海拔       1.00      1.00      1.00        43
        连载网站       0.98      1.00      0.99      1658
          丈夫       0.85      0.88      0.87       678
          朝代       0.87      0.91      0.89       419
          民族       0.98      0.99      0.99      1434
           号       0.92      0.99      0.95       197
         出版社       0.98      0.99      0.98      2272
         主持人       0.89      0.83      0.86       200
        专业代码       1.00      1.00      1.00         3
          歌手       0.88      0.92      0.90      2857
          作词       0.81      0.83      0.82       884
          主角       0.92      0.59      0.72        39
         董事长       0.93      0.83      0.88        47
        毕业院校       0.99      0.99      0.99      1433
        占地面积       0.89      0.82      0.85        61
        官方语言       1.00      1.00      1.00        15
        邮政编码       1.00      1.00      1.00         4
        人口数量       1.00      0.98      0.99        45
        所在城市       0.95      0.95      0.95        77
          作者       0.97      0.97      0.97      4359
        成立日期       0.99      0.99      0.99      1608
          作曲       0.78      0.71      0.75       849
          气候       0.99      1.00      1.00       103
          嘉宾       0.77      0.78      0.77       158
          主演       0.94      0.96      0.95      7383
         改编自       0.88      0.90      0.89        71
         创始人       0.85      0.91      0.88        75

    accuracy                           0.93     49506
   macro avg       0.92      0.91      0.92     49506
weighted avg       0.93      0.93      0.93     49506
```
### 三元组提取

&emsp;&emsp;最后一部分，也是本次比赛的最终目标，就是三元组提取。

&emsp;&emsp;三元组提取采用Pipeline模式，先用序列标注模型预测句子中的实体，然后再用关系分类模型判断实体关系的类别，过滤掉关系为未知的情形，就是我们想要提取的三元组了。

&emsp;&emsp;三元组提取的代码为triple_extract/triple_extractor.py 。

&emsp;&emsp;运行三元组提取脚本，代码为triple_extract/spo.py 。

&emsp;&emsp;我们在网上找几条样本进行测试，测试的结果如下：

>
>

### 总结
&emsp;&emsp;本文标题为限定领域的三元组抽取的一次尝试，之所以取名为限定领域，是因为该任务的实体关系是确定，一共为50种关系。

&emsp;&emsp;当然，上述方法还存在着诸多不足，参考苏建林的文章[基于DGCNN和概率图的轻量级信息抽取模型](https://spaces.ac.cn/archives/6671)，我们发现不足之处如下：

- 主体和客体的标注策略有问题，因为句子中有时候主体和客体会重叠在一起；
- 新引入了一类关系：未知，是否有办法避免引入；
- 其他（暂时未想到）


&emsp;&emsp;从比赛的角度将，本文的办法效果未知，应该会比联合模型的效果差一些。但是，这是作为笔者自己的模型，算法是一种尝试，之所以采用这种方法，是因为笔者一开始是从开放领域的三元组抽取入手的，而这种方法方便扩展至开放领域。关于开放领域的三元组抽取，笔者稍后就会写文章介绍，敬请期待。


### 参考网址

1. NLP（二十五）实现ALBERT+Bi-LSTM+CRF模型：https://blog.csdn.net/jclian91/article/details/104826655 
2. NLP（二十一）人物关系抽取的一次实战： https://blog.csdn.net/jclian91/article/details/104380371
3. 基于DGCNN和概率图的轻量级信息抽取模型：https://spaces.ac.cn/archives/6671