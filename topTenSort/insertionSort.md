```PYTHON
def insertionSort(nums):
    # 代码中，i 表示取列表中索引为 i 的数据进行插入排序(相当于“抓牌”)，
    # 每次发现抓到的牌小于已排序的最后一个，置换位置，然后已经排序的数量减一
    # 使用 currnet 标记待插入数据向前移动时的索引，
    # 直到不需再移动(相当于将新抓的牌插入到已有的牌中)，
    # 当列表中的所有数据都插入到了已排序序列中，列表排序完成。
    for i in range(0,len(nums)):
        currnet = i
        # 如果满足条件会移动置换当前元素，直到不满条件。
        while nums[currnet] < nums[currnet-1] and currnet-1>=0:
            nums[currnet],nums[currnet-1] = nums[currnet-1],nums[currnet]
            currnet -= 1
    return nums

if __name__ == '__main__':
    # 扑克牌
    # 插入排序：通过构建有序序列，对于未排序数据，
    # 在已排序序列中从后向前扫描，找到相应位置并插入。
    # 最好情况O(n),最坏情况O(n^2)
    # 空间复杂度O(1)
    nums = [22, 34, 3, 32, 82, 55, 89, 50, 37, 5, 64, 35, 9, 70]
    print(insertionSort(nums))
