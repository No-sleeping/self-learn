```python
# 快速排序使用分治法（Divide and conquer）
# 策略来把一个串行（list）分为两个子串行（sub-lists）。
# 1、从数列中挑出一个元素，称为 "基准"（pivot）;
# 2、 重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面（相同的数可以到任一边）。
# 在这个分区退出之后，该基准就处于数列的中间位置。这个称为分区（partition）操作；
# 3、递归地（recursive）把小于基准值元素的子数列和大于基准值元素的子数列排序；

# 时间复杂度：平均O(nlogn)、最坏O(n^{2})，最好O(nlogn);
# 空间复杂度：O(logn)

def run(arr,left,right):
    index, pivot = left, right
    for i in range(left,right):
        if arr[i] < arr[pivot]:
            arr[i], arr[index] = arr[index], arr[i]
            index += 1
    arr[index], arr[right] = arr[right], arr[index]
    return index


def quickSort(arr,left,right):
    if left >= right:
        return
    piv = run(arr,left,right)
    quickSort(arr,left,piv-1)
    quickSort(arr,piv+1,right)
    
    
 if __name__ == "__main__":
    # arr = [random.randint(1,100) for _ in range(10)]
    arr = [36, 81, 17, 80,23,55,24,12,53,30]
    quickSort(arr,0,len(arr)-1)
    print(arr)
```

![](https://www.runoob.com/wp-content/uploads/2019/03/quickSort.gif)
