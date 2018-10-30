# apriori/FP-growth实现

### 背景

找出商品的关联商品，目的是为了挖掘出顾客在购买某一商品后，还会有较大意愿购买其他的商品，或者几个商品搭配销售，以此来做个关联销售，达到提高我们销售额的目的。

### 思路

Apriori算法的一个思想：如果一个商品不是频繁的，那么与它组成的所有商品集合都不是频繁的，根据这一思想，我们可以不用计算一些出现次数不是很多的商品的组合，从而减少了计算量。

第一步，寻找频繁项集：

通过遍历所有订单，找出其中出现的商品，形成候选集， 计算候选集的支持度，筛选出支持度高于阈值的候选集商品作为频繁一项集，以频繁一项集中的商品为基础，将里面的商品进行组合，并计算这些商品的支持度，取支持度高于阈值的商品作为频繁二项集，以此类推，找出所有的频繁项集。

第二步，寻找关联规则：

关联规则的制定用了和apriori相似的一个规则，即如果某条规则并不满足最小可信度要求，那么该规则的所有子集也不会满足最小可信度要求，如图所示：

  

![1540879360764](C:\Users\ADMINI~1\AppData\Local\Temp\1540879360764.png)

 

 

假设规则{0,1,2} ➞ {3}并不满足最小可信度要求，那么就知道任何左部为{0,1,2}子集的规则也不会满足最小可信度要求，可以利用关联规则的上述性质属性来减少需要测试的规则数目。

支持度：商品出现的次数

置信度：购买某一商品后购买另一商品的概率

### 实例：

~~~python
def apriori( dataSet, minSupport  ):
    # 构建初始候选项集C1
    C1 = createC1( dataSet )
    
    # 将dataSet集合化，以满足scanD的格式要求
    D = list(map( set, dataSet ))
    
    # 构建初始的频繁项集，即所有项集只有一个元素
    L1, suppData = scanD( D, C1, minSupport )
    L = [ L1 ]
    # 最初的L1中的每个项集含有一个元素，新生成的
    # 项集应该含有2个元素，所以 k=2
    k = 2
    
    while ( len( L[ k - 2 ] ) > 0 ):
        Ck = aprioriGen( L[ k - 2 ], k )
        Lk, supK = scanD( D, Ck, minSupport )
        
        # 将新的项集的支持度数据加入原来的总支持度字典中
        suppData.update( supK )
        
        # 将符合最小支持度要求的项集加入L
        L.append( Lk )
        
        # 新生成的项集中的元素个数应不断增加
        k += 1
    # 返回所有满足条件的频繁项集的列表，和所有候选项集的支持度信息
    return L, suppData
~~~

得出频繁项集以及候选集的支持度

运行代码得到：

频繁项集：

![img](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image001.png)

 

支持度信息：

![img](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image002.png)

根据支持度计算出商品关联的置信度，当置信度大于我们设定的阈值时，记录下规则：

运行代码：

~~~python
def calcConf( freqSet, H, supportData, brl, minConf):
    '''
    计算规则的可信度，返回满足最小可信度的规则。
    
    freqSet(frozenset):频繁项集
    H(frozenset):频繁项集中所有的元素
    supportData(dic):频繁项集中所有元素的支持度
    brl(tuple):满足可信度条件的关联规则
    minConf(float):最小可信度
    '''
    prunedH = []
    for conseq in H:
        conf = supportData[ freqSet ] / supportData[ freqSet - conseq ]
        if conf >= minConf:
            print (freqSet - conseq, '-->', conseq, 'conf:', conf)
            brl.append( ( freqSet - conseq, conseq, conf ) )
            prunedH.append( conseq )
    return prunedH

def rulesFromConseq(freqSet, H, supportData, brl, minConf):
    m = len(H[0])
    while (len(freqSet) > m): # 判断长度 > m，这时即可求H的可信度
        H = calcConf(freqSet, H, supportData, brl, minConf)
        if (len(H) > 1): # 判断求完可信度后是否还有可信度大于阈值的项用来生成下一层H
            H = aprioriGen(H, m + 1)
            m += 1
        else: # 不能继续生成下一层候选关联规则，提前退出循环
            break
~~~

得到规则和对应规则的置信度信息：

![img](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image001.png)

 

**遇到的问题**：

1.     运行一次的时间太长
2.     将指标数据调整后导致系统崩溃

**解决方法**：

1.       采用apriori算法的改良版本FP-growth算法发现频繁项集，大大的节约了时间。

2. FP-growth算法的支持度不宜调的太低，不然系统会因为计算量庞大而崩溃


 **FP** **算法原理**：

![1540879147193](C:\Users\ADMINI~1\AppData\Local\Temp\1540879147193.png)

FP算法引入一个数据结构，该数据结构分成三部分，第一部分是项头表，里面记录了所有频繁一项集出现的次数，并按次数降序排列；第二部分是节点链表，所有项头表里的频繁一项集都是一个节点链表的头，它依次指向FP树中该频繁一项集出现的位置，这样做主要是方便项头表和FP Tree之间的联系查找和更新。第三部分是FP树，FP树起始没有数据，建立FP树时我们一条条的读入排序后的数据集，插入FP树，插入时按照排序后的顺序，插入FP树中,如图所示：

|      |                                                              |
| ---- | ------------------------------------------------------------ |
|      | ![img](C:\Users\ADMINI~1\AppData\Local\Temp\1540879124999.png) |

在起始的FP树中插入ACEBF数集，则对每个节点的计数+1。

 

插入ACG数据集时，A,C,G的节点树各+1，以此方法将所有数据集插入到FP树，FP树的建立就完成了。

有了FP树后就可以根据FP树找出频繁项集，我们从每条链路出发，如上图所示，关于F的频繁二项集就是{A:1,F:1}, {C:1,F:1}, {E:1,F:1}, {B:1,D:1},频繁三项集为{A:1,C:1,F:1},……以此类推。

 

### 实例：

~~~python
def fpGrowth(dataSet, minSup):
    initSet = createInitSet(dataSet)
    myFPtree, myHeaderTab = createTree(initSet, minSup)
    freqItems = []
    mineTree(myFPtree, myHeaderTab, minSup, set([]), freqItems)
    return freqItems
~~~

得到频繁项集：

![img](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image001.png)

运行代码：

~~~python
def scanD( D, Ck):
    '''
    计算Ck中的项集在数据集合D(记录或者transactions)中的支持度,
    返回满足最小支持度的项集的集合，和所有项集支持度信息的字典。
    '''
    ssCnt = {}
    for tid in D:
        # 对于每一条transaction
        for can in Ck:
            # 对于每一个候选项集can，检查是否是transaction的一部分
            # 即该候选can是否得到transaction的支持
            if can.issubset( tid ):
                ssCnt[ can ] = ssCnt.get( can, 0) + 1
    numItems = float( len( D ) )
    supportData = {}
    for key in ssCnt:
        # 每个项集的支持度
        support = ssCnt[ key ] / numItems
        supportData[ key ] = support
    return supportData

suppData = scanD( D, freqItemss)
~~~

得到支持度信息：

![img](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image001.png)

运行如下代码得到关联规则结果：

~~~python
def calcConf( freqSet, H, supportData, brl, minConf):
    '''
    计算规则的可信度，返回满足最小可信度的规则。
    
    freqSet(frozenset):频繁项集
    H(frozenset):频繁项集中所有的元素
    supportData(dic):频繁项集中所有元素的支持度
    brl(tuple):满足可信度条件的关联规则
    minConf(float):最小可信度
    '''
    prunedH = []
    for conseq in H:
        conf = supportData[ freqSet ] / supportData[ freqSet - conseq ]
        if conf >= minConf:
#            print (freqSet - conseq, '-->', conseq, 'conf:', conf)
            brl.append( ( freqSet - conseq, conseq, conf ) )
            prunedH.append( conseq )
    return prunedH

def rulesFromConseq(freqSet, H, supportData, brl, minConf):
    m = len(H[0])
    while (len(freqSet) > m): # 判断长度 > m，这时即可求H的可信度
        H = calcConf(freqSet, H, supportData, brl, minConf)
        if (len(H) > 1): # 判断求完可信度后是否还有可信度大于阈值的项用来生成下一层H
            H = aprioriGen(H, m + 1)
            m += 1
        else: # 不能继续生成下一层候选关联规则，提前退出循环
            break

def generateRules(L, supportData, minConf):
    bigRuleList = []
    for freqSet in L:
        H1 = [frozenset([item]) for item in freqSet]
        rulesFromConseq(freqSet, H1, supportData, bigRuleList, minConf)
    return bigRuleList
L1 = []
for i in freqItemss:
    if len(i)>1:
        L1.append(i)

rules = generateRules( L1,suppData, minConf = 0.001 )

~~~

得到的结果如下：

![img](file:///C:\Users\ADMINI~1\AppData\Local\Temp\msohtmlclip1\01\clip_image001.png)

 

 

 

 

## Apriori算法和FP算法的异同：



相同点：

1.       同样是通过先寻找频繁项集，然后挖掘规则的方法，寻找频繁项集的思路也一样，通过商品出现的频率来计算支持度，从而得到频繁项集

2.       得到的结果一样

不同点：

1.       寻找频繁项集的方法不同，apriori通过传统的遍历数据集的方式寻找频繁项集，当数据量大时，这个方法计算所需要的时间就会大幅增加（因为需要反复遍历数据库）;FP算法通过构建FP树的方法发现频繁项集，该算法只需遍历两次数据集，第一次计算各商品出现的频率，第二次构建FP树。

2.       代码运行时长不同，原因在于以上提到的他们寻找频繁项集所采用的方法不同。例中所取参数完全一样（支持度为0.001，置信度为0.1），apriori算法运行时间为1216秒，FP算法运行时间为146秒，当数据量变大时，该差异会更加显著。