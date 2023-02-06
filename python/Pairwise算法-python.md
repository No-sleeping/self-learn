# Pairwise算法

对于以下测试场景：

- 浏览器：M，O，P
- 操作平台：W（windows），L（linux），i（ios）
- 语言：C（chinese），E（english）

对于上述测试场景，可以通过笛卡尔积设计18条两两组合的测试用例：

```
1，M W C
2，M W E
3，M L C
4，M L E
5，M I C
6，M I E
7，O W C
8，O W E
9，O L C
10，O L E
11，O I C
12，O I E
13，P W C
14，P W E
15，P L C
16，P L E
17，P I C
18，P I E
```

对于第18条用例P I E来说，两两组合是PI ，PE ，IE，PI在17号，PE在16号，IE在12号出现过，所以第18条用例可以过滤掉。按照这个算法继续过滤，最终剩下9条用例：

```
1，M W C
4，M L E
6，M I E
7，O W E
9，O L C
11，O I C
14，P W E
15，P L C
17，P I C
```

用例减少了50%！而且维度越多越明显，当有10个维度的时候4*4*4*4*3*3*3*2*2*2=55296个测试case，pairwise为24个，是原始测试用例规模的0.04%。

<br/>

根据数学统计分析，73%的缺陷（单因子是35%，双因子是38%）是由单因子或2个因子相互作用产生的。19%的缺陷是由3个因子相互作用产生的。也就是说，**大多数的bug都是条件的两两组合造成的**。

Pairwise算法是L. L. Thurstone在1927年首先提出来的，他是美国的一位心理统计学家。Pairwise算法基于两两组合，过滤出性价比高的用例集。它的思路是：**如果某一组用例的两两组合结果，在其他组合中均出现，就删除该组用例**，从而精简用例。

<br/>

# python 实践

```python
from allpairspy import AllPairs
from collections import OrderedDict


def is_valid_combination(row):
    n = len(row)
    # 设置过滤条件
    if n > 2:
        if "and" == row[0] and 11 == row[1]:
            return False
    return True


def genCase():
    par = OrderedDict({
    "name" : ["and", "qwe", "rew"],
    "age" : [11, 3,9],
    "gender": ["male", "female"],
    "qwe": [1,3,54]
    })
    for i, pairs in enumerate(AllPairs(par, filter_func=is_valid_combination)):
        print("用例编号{:2d}: {}".format(i, pairs))
     
        
if __name__ == '__main__':
    genCase()
```

```python
# 未添加过滤函数
用例编号 0: Pairs(name='and', age=11, gender='male', qwe=1)
用例编号 1: Pairs(name='qwe', age=3, gender='female', qwe=1)
用例编号 2: Pairs(name='rew', age=9, gender='female', qwe=3)
用例编号 3: Pairs(name='rew', age=3, gender='male', qwe=54)
用例编号 4: Pairs(name='qwe', age=11, gender='male', qwe=3)
用例编号 5: Pairs(name='and', age=9, gender='female', qwe=54)
用例编号 6: Pairs(name='and', age=3, gender='female', qwe=3)
用例编号 7: Pairs(name='qwe', age=9, gender='male', qwe=54)
用例编号 8: Pairs(name='rew', age=11, gender='female', qwe=54)
用例编号 9: Pairs(name='rew', age=9, gender='female', qwe=1)

# 添加过滤函数
用例编号 0: Pairs(name='qwe', age=11, gender='male', qwe=1)
用例编号 1: Pairs(name='rew', age=3, gender='female', qwe=1)
用例编号 2: Pairs(name='rew', age=9, gender='male', qwe=3)
用例编号 3: Pairs(name='qwe', age=9, gender='female', qwe=54)
用例编号 4: Pairs(name='qwe', age=3, gender='male', qwe=54)
用例编号 5: Pairs(name='rew', age=11, gender='female', qwe=54)
用例编号 6: Pairs(name='qwe', age=11, gender='female', qwe=3)
用例编号 7: Pairs(name='qwe', age=3, gender='female', qwe=3)
用例编号 8: Pairs(name='qwe', age=9, gender='female', qwe=1)

```

<br/>

# 原理（为什么不是x * y  *z）

https://www.softwaretestinghelp.com/what-is-pairwise-testing/

<br/>

---

<br/>

参考：

https://github.com/thombashi/allpairspy

https://pypi.org/project/allpairspy/#basic-usage

https://cloud.tencent.com/developer/article/2063872

https://cloud.tencent.com/developer/article/1899632
