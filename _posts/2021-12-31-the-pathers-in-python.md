---
title:  "The Patchers in Python"
layout: post
---

Mock在单元测试中随处可见。在Python的单元测试中，常见的有unittest和pytest。unittest对应的Mock库是unittest.mock，pytest对应的是pytest-mock。这两个Mock库中都存在Patchers，Patchers与Mock密不可分


## 什么是Patch

Pytest-mock本质上与unittest.mock是相同的API，因此这里我们先从unittest.mock说起

### A simple example
<p align="center">
    <img src="https://lh6.googleusercontent.com/Gm-L7UhOQmRbqnlAeKwRV7Vgx8SAdM8gAFJ2nEJREbkL3nmuNsB9YROIYEIqR2G8-GEB-wXXbVnxYO1NsP4Z0zokzvNQ0TnqkygVaJVrtuBKiNCIfsEl5Tq1Hub-Lq9omrUaV8W3" alt="图1. Example code" width="350"/>
</p>

<center>图1. Example code</center>

当测试目标是`BubbleTeaShop`的`make_milk_tea`方法，但又不想实际调用`Kitchen`的`make_milk_tea`方法时，就可以使用下图中的`@patch()`方法对class进行patch，并使用`instance.make_milk_tea.return_value`指定了方法的返回值

<p align="center">
    <img src="https://lh3.googleusercontent.com/RkBu9Zvaaqw7U1gnx4Y6WhCLWMElTzJ5BKaS8gX9KeuwekO1iVtE4MEeeSm_IB1aSmZW3y2UqRBLrrzW7CE_3XoaRXpuyyN--WKXq7N_PIw0B73F4RFKEMXlBTqx7eR33TC5nAfh" alt="图2. 使用patch进行mock" width="350"/>
</p>

<center>图2. 使用patch进行mock</center>

上例`bubbleteashop.py`中的`Kitchen`被`@patch`所替换，`@patch`所返回的是一个`unittst.mock`中的`MagicMock`对象

<p align="center">
    <img src="https://lh4.googleusercontent.com/UwP7-utvazXw4iKuIenLGvOGPMQLudQnixrsiCRbYOdFbBYwQdTludRJC4m9sKcOxQHP7pKG4KgWWRT6xes8tPBYPdNhFa_-c23SM2GXr6PvtOzhwiLSTRrQxA0BLVdu6tD5cFaO" alt="图3. 被MagicMock替换" width="350"/>
</p>
<center>图3. 被MagicMock替换</center>

## Where to Patch

---

在官方文档中讲述Patch使用之前提到

> **Note:** The key is to do the patching in the right namespace. See the section [where to patch](https://docs.python.org/3/library/unittest.mock.html#id6).
> 

> The basic principle is that you patch where an object is looked up, which is not necessarily the same place as where it is defined.
> 

即在object被查找（被使用）的地方进行patch，而非定义的地方。并且文档也提到一个前提：object是通过`from import`导入的

以上例来讲，我们通过`from src.kitchen import Kitchen`的方式导入`Kitchen`,

所以patch的是`src.bubbleteashop.Kitchen`，而非`src.kictchen.Kitchen`

原因都与python的赋值机制直接相关

![图4. 引用[Ned Batchelder - Python Names and Values](https://nedbatchelder.com/text/names1.html)](https://lh6.googleusercontent.com/unT9nJaXYLL-KUfYxRdFMa9G2gj-zJ3768PPelwT5JuR8LJtA0GmCq06BI8k994gCHSnEuzk0ES1QjyGHVCVNdMlb16sHux3ggqoWlQnoN60PG_SL_04xqiZAx_tQaSM3DZe3LNn)

<center>图4. 引用[Ned Batchelder - Python Names and Values](https://nedbatchelder.com/text/names1.html)</center>

Ned Batchelder在Pycon 2015中分享时提到，Python 中的变量是引用值的名称。如果分配第二个名称，这些名称不会相互引用，它们都引用相同的值（x与y都指向了23）。如果然后再次分配其中一个名称（x=12），则另一个名称不受影响（y仍然是23）

在之前的例子中，`from src.kitchen import Kitchen`就等同于先导入`src.kitchen`，再进行赋值`kitchen = src.kitchen.Kitchen`，kitchen如同y，src.kitchen.Kitchen如同x。

如果以`src.bubbleteashop.Kitchen`进行patch时，`bubbleteashop.py`的`Kitchen`会被替换为MagicMock，这正是测试中所希望达到的目标

如果以`src.kictchen.Kitchen`进行patch时，`kitchen.py`的`Kitchen`会被替换为MagicMock，而`bubbleteashop.py`的`Kitchen`不受影响，并没有被mock

<p align="center">
    <img src="https://lh4.googleusercontent.com/EXFoa7diKDyOHA_STBhVjraKCSXXtrRzdmbJzXnZ7O02ryEcGDietib5OJSLcPDBd355YNjcmPQM-8lcLDuhsepZq9LWEPZfBDq3LKGY9wTNpdbyAW3wgSTiPSdVm3O4EHPGCnPG" alt="图5. “from import”导入" width="500px"/>
</p>
<center>图5. “from import”导入</center>

如果我们以`import src.kitchen`进行导入呢？

<p align="center">
    <img src="https://lh4.googleusercontent.com/LphYx2h5_Uz7GeZOERPfKSNowBf1Db7Eu1-cO6Cx3t98k6RhjP2Q7zcWSNMoOB_82Jtwv-ekKnx1uN-XzFjNp201PWKWhzkFBloF83spaUzMx6JZucM_JbMyymrgLDWHIF_AsMsr" alt="图6. “import src.kitchen” 示例" width="350" />
</p>

<center>图6. “import src.kitchen” 示例</center>

`import src.kitchen`等同于在`bubbleteashop.py`中使`src.kitchen`指代整个kitchen模块，访问`src.kitchen.make_milk_tea`会进入到kitchen中,找到`make_milk_tea`，并执行它。

如果以`src.kictchen.Kitchen`进行patch时，`kitchen.py`的`Kitchen`会被替换为MagicMock，因为`bubbleteashop.py`的`src.kitchen`指代整个kitchen模块，所以mock同样会影响到`bubbleteashop.py`

<p align="center">
    <img src="https://lh3.googleusercontent.com/E3ZPM-EYew590CNR3YTTlb0rXMLVxiAHCWIBS3WmPIIMxt4FLW3Sr6twkJnDPfYxQtQr2kY-hFmOgAesPz3TuNK10C4SkK6KLYUKayK4kn_DguFi6rm28AhzUwluL0wIDjjBMjVS" alt="图7. “import”导入" width="500px"/>
</p>

<center>图7. “import”导入</center>

## Patch scope

---

Patch主要有三种方式，每一种方式的scope不同，使用场景也不同

<p align="center">
    <img src="https://lh5.googleusercontent.com/WuM3NunSa9NEbQIj-Qpr4hfTvyPER2XBpkdGTvAlXgKGw-EAMKcD0_N-pv9ofwQMdSF_hBkfxl4bGGv5GCfaHt8mV_Lu4tV__qGmit74le041GlCMR9A__z3o-6MzmUwtNpLONP-" alt="图8. Patch scope" width="350px"/>
</p>

<center>图8. Patch scope</center>

### Context manager

使用with的声明进行patch，patch的范围仅限于with语句的语句体

<p align="center">
    <img src="https://lh3.googleusercontent.com/ZFqU9ElTW4kOUL_NNy5A4mpCwOSCoMB7Mod-CnpKPm2f3jfsgd0uSPbR7kMlSSfw0ftd9_SJMWK9zK-tKI768A7MPcWnXLkLGGVnJkP-OkDzJzGs9RgSqyiZFA9J5pXJWXxBnTeX" alt="图9. Context manager scope" width="350px"/>
</p>
<center>图9. Context manager scope</center>

### Function decorator

如同上例中创建在测试方法上的`@patch(‘src.bubbleteashop.Kitchen’)`就是一个function decorator，仅作用于当前测试方法

<p align="center">
    <img src="https://lh6.googleusercontent.com/p48dN6LRn02rIquL669Mq58UBVzuodYeLhrBpBWrhcO48AzJSH4SIKua-MgNpOySFALrzZqgQFu8N-LViCsfdkq9X0U40VU19I38Xfjj-FQo6nNlkYfW3If2rvvRqHhzFTLezgQD" alt="图10. Function decorator scope" width="350px"/>
</p>

<center>图10. Function decorator scope</center>

### Class decorator

同样采用`@pathch()`的方法进行创建，不过是创建在class上，作用于这个test class的每一个方法，适用于多个测试方法共享同样的patcher

<p align="center">
    <img src="https://lh4.googleusercontent.com/mSdXz-v01Yh7iLkf4n92dqtxBjeO-7IWdGAcOJ-ZPjJkIRQCL5bOwAQ_eMZuBL9h9ECUSJB5R-ODPyjl8J4oQOwT-8ISirdUU-YnefHE1EfG6XbWebVisp6LNL-r56FVx5nsJfPQ" alt="图11. Class decorator scope" width="350px"/>
</p>

<center>图11. Class decorator scope</center>
