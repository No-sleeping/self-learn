```PYTHON
#首先在未排序序列中找到最小（大）元素，存放到排序序列的起始位置。
#再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。
#重复第二步，直到所有元素均排序完毕。
def selection_sort(arr):
    n = len(arr)
    for i in range(n-1):
        # 记录当前最小元素下标
        minIndex = i
        # 从当前元素之后 找到是更小的元素
        for j in range(i+1,n):
            if arr[j] < arr[minIndex]:
                minIndex = j
        # 如果找到了，minIndex变化了，将 i 和最小数进行交换
        if minIndex != i:
            arr[i], arr[minIndex] = arr[minIndex], arr[i]
    return arr
```
