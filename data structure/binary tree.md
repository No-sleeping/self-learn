# 前序

二叉树的前序遍历的枚举规则为：**根结点 —> 左子树 —> 右子树**，也就是给定一棵树，输出操作当前节点，然后枚举左子树(左子树依然按照根左右的顺序进行)，最后枚举右子树(右子树也按照根左右的顺序进行)，这样得到的一个枚举序列就是二叉树的前序遍历序列(也叫先序)。

![](https://img-blog.csdnimg.cn/img_convert/831d5c05f1d2dfc82b8abf407699a8f5.png)

```python
class Solution {
    List<Integer>value=new ArrayList();
    public List<Integer> preorderTraversal(TreeNode root) {
        qianxu(root);
        return value;
    }
    private void qianxu(TreeNode node) {
        if(node==null)
            return;
        value.add(node.val);
        qianxu(node.left);
        qianxu(node.right);
    }
}
```

# 中序

<br/>

![](https://img-blog.csdnimg.cn/img_convert/bc401f2b831d1b92f874296a04c00bd4.png)

```python
class Solution {
   public List<Integer> inorderTraversal(TreeNode root) {
	  List<Integer>value=new ArrayList<Integer>();
	  zhongxu(root,value);
	  return value;
  }
	 private void zhongxu(TreeNode root, List<Integer> value) {
		 if(root==null)
			 return;
		 zhongxu(root.left, value);
		 value.add(root.val);
		 zhongxu(root.right, value);
		
	}
}
```

# 后序

![](https://img-blog.csdnimg.cn/img_convert/9ac0f26dda230c75b87b518d2cf77edc.png)

<br/>

```python
class Solution {
    List<Integer>value=new ArrayList<>();
    public List<Integer> postorderTraversal(TreeNode root) {
        houxu(root);
        return value;
    }
    private void houxu(TreeNode root) {
        if(root==null)
            return;
        houxu(root.left);
        houxu(root.right);//右子树回来
        value.add(root.val);
    }
}
```

<br/>

---

```python
class Btree(object):
    def __init__(self,data=None,left=None,right=None):
        self.data = data
        self.left =left
        self.right = right

    # 前序遍历
    def preorder(self):
        if self.data is not None:
            print(self.data)
        if self.left is not None:
            self.left.preorder()
        if self.right is not None:
            self.right.preorder()

    # 中序遍历
    def inorder(self):
        if self.left is not None:
            self.left.preorder()
        if self.data is not None:
            print(self.data)
        if self.right is not None:
            self.right.preorder()

    # 后序遍历
    def postorder(self):
        if self.left is not None:
            self.left.preorder()
        if self.right is not None:
            self.right.preorder()
        if self.data is not None:
            print(self.data)


if __name__ == "__main__":
    right_tree = Btree(6)
    right_tree.left = Btree(2)
    right_tree.right = Btree(4)

    left_tree = Btree(5)
    left_tree.left = Btree(1)
    left_tree.right = Btree(3)

    tree = Btree(11)
    tree.left = left_tree
    tree.right = right_tree

    left_tree = Btree(7)
    left_tree.left = Btree(3)
    left_tree.right = Btree(4)

    right_tree = tree  # 增加新的变量
    tree = Btree(18)
    tree.left = left_tree
    tree.right = right_tree

    print('先序遍历为:')
    tree.preorder()
    print('中序遍历为:')
    tree.inorder()
    print('后序遍历为:')
    tree.postorder()
```

---

# 总结

  二叉树是很多重要算法及模型的基础，比如二叉搜索树（BST），哈夫曼树(Huffman Tree)，CART决策树等。本文先介绍了树的基本术语，二叉树的定义与性质及遍历、储存，然后笔者自己用Python实现了二叉树的上述方法，笔者代码的最大亮点在于实现了二叉树的可视化，这个功能是激动人心的。
  在Python中，已有别人实现好的二叉树的模块，它是**binarytree**模块，其官方文档的网址为：https://pypi.org/project/binarytree/。其使用的例子如下：

```
>>> from binarytree import tree
>>> tree(height=4)
Node(22)
>>> a= tree(height=4)
>>> a
Node(4)
>>> print(a)

        _________4____
       /              \
  ____1___             30______
 /        \           /        \
13        _15        2         _24__
  \      /   \      /         /     \
   26   21    29   6        _23      25
                           /        /
                          16       5

```
---

参考：

https://blog.csdn.net/qq_40693171/article/details/120425294  （一文彻底搞定二叉树的前序、中序、后序遍历(图解递归非递归)）

https://www.jianshu.com/p/9503238394df  (二叉树的Python实现)
