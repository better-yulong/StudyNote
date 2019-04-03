>PDF电子github地址（欢迎下载收藏；如若涉及版本也请联系删除）：https://github.com/better-yulong/StudyNote-Resource/blob/master/StudyNote-Resource/doc-pdf/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E4%B8%8E%E7%AE%97%E6%B3%95%E5%88%86%E6%9E%90_Java%E8%AF%AD%E8%A8%80%E6%8F%8F%E8%BF%B0(%E7%AC%AC2%E7%89%88).pdf

#### 一、本书讨论的内容
设有一组N个数而要确定其中第k个最大者，我们称之为选择问题（selection problem）。该问题，解决方法较多。
> 1. 排序（熟知）：将N个数存入数组，通过简单的排序算法(如冒泡排序）以递减顺序将数据排序，然后返回第k或k-1上的元素
> 2. 分组插入（稍后）:将前k个元读入数组并（以递减顺序）对其排序。之后，将剩余元素逐个读入；若读到的新元素小于数组中第k个元素则忽略，否则将新元素放入数组正确的位置同时将原数据中的