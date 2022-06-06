```PYTHON
def merg(nums1, nums2):
    result = []
    while nums1 and nums2:
        if nums1[0] <= nums2[0]:
            result.append(nums1.pop(0))
        else:
            result.append(nums2.pop(0))
    while nums1:
        result.append(nums1.pop(0))
    while nums2:
        result.append(nums2.pop(0))
    return result

def mergeSort(nums):
    if(len(nums) < 2):
        return nums
    middle = len(nums)//2
    nums1 = nums[:middle]
    nums2 = nums[middle:]
    return merg(mergeSort(nums1),mergeSort(nums2))

if __name__ == '__main__':
    # 归并排序：每次将当前的list分成两组
    # 然后针对这两组list从头元素进行比较：从小到大排序
    # 利用递归merg(mergeSort(nums1),mergeSort(nums2)) 完成整租的怕排序
    # 稳定，时间复杂度：平均O(nlogn)、最坏O(nlogn)，最好O(nlogn);
    # 空间复杂度：O(n)
    nums = [22, 34, 3, 32,  82, 55, 89, 50, 37, 5, 64, 35, 9, 70]
    print(mergeSort(nums))
```
