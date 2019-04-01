Morris遍历二叉树算法

morris遍历的时候每个节点最多被访问2次，时间复杂度O（n）,空间复杂度为O（1）
思路是：每个节点去找它的中序变量的前一个节点，即它的左子树的最右节点，如果最右节点的右子树等于当前结点，说明是第二次遍历了，把这个它的右子树指向空，向右走；否则把最右节点的右子树指向当前结点，向左走，走到null之后向右走



**复杂度分析：**

空间复杂度：O(1)，因为只用了两个辅助指针。

时间复杂度：O(n)。证明时间复杂度为O(n)，最大的疑惑在于寻找中序遍历下二叉树中所有节点的前驱节点的时间复杂度是多少，即以下两行代码：

```java
 while (prev->right != NULL && prev->right != cur)
     prev = prev->right;
```

直觉上，认为它的复杂度是O(nlgn)，因为找单个节点的前驱节点与树的高度有关。但事实上，寻找所有节点的前驱节点只需要O(n)时间。n个节点的二叉树中一共有n-1条边，整个过程中每条边最多只走2次，一次是为了定位到某个节点，另一次是为了寻找上面某个节点的前驱节点，如下图所示，其中红色是为了定位到某个节点，黑色线是为了找到前驱节点。所以复杂度为O(n)。

![img](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/morris.jpg)

```java
//前序遍历
    public static void preList(Node head){
        if(head==null)return;
        Node cur1 = head;
        Node cur2 = null;
        while(cur1!=null){
            cur2 = cur1.left;
            if(cur2!=null){
                while(cur2.right!=null&&cur2.right!=cur1){
                    cur2 = cur2.right;
                }
                if(cur2.right==null){
                    System.out.print(cur1.value+" ");
                    cur2.right = cur1;
                    cur1 = cur1.left;
                    continue;
                }else if(cur2.right==cur1){
                    cur2.right = null;
                }
            }else{
                System.out.print(cur1.value+" ");
            }
            cur1 = cur1.right;
        }
        System.out.println();
    }

    //中序遍历
    public static void midList(Node head){
        if(head==null)return;
        Node cur1 = head;
        Node cur2 = null;
        while(cur1!=null){
            cur2 = cur1.left;
            if(cur2!=null){
                while(cur2.right!=null&&cur2.right!=cur1){
                    cur2 = cur2.right;
                }
                if(cur2.right==null){

                    cur2.right = cur1;
                    cur1 = cur1.left;
                    continue;
                }else if(cur2.right==cur1){
                    cur2.right = null;
                }
            }
            System.out.print(cur1.value+" ");
            cur1 = cur1.right;
        }
        System.out.println();
    }


    //后序遍历，这个比较有意思，就是在morris的基础上，在第二次到这个节点的时候，
    // 说明这个节点的左边完成了，把这个节点左子树打印,打印方法是一直把右边的指针逆序过来，逆转后打印，然后再逆序回去
    public static void afterList(Node head){
        if(head==null)return;
        Node cur1 = head;
        Node cur2 = null;
        while(cur1!=null){
            cur2 = cur1.left;
            if(cur2!=null){
                while(cur2.right!=null&&cur2.right!=cur1){
                    cur2 = cur2.right;
                }
                if(cur2.right==null){

                    cur2.right = cur1;
                    cur1 = cur1.left;
                    continue;
                }else if(cur2.right==cur1){

                    cur2.right = null;
                    printReverse(cur1.left);//逆序打印
                }
            }
            cur1 = cur1.right;
        }
        printReverse(head);//逆序打印
        System.out.println();
    }

    public static Node reverse(Node cur1){
        Node pre = null;
        Node next = null;
        while(cur1!=null){
            next = cur1.right;
            cur1.right = pre;
            pre = cur1;
            cur1 = next;
        }
        return pre;
    }
    public static void printReverse(Node cur1) {
        //逆转
        Node cur = reverse(cur1);
        Node head = cur;
        while(cur!=null){
            System.out.print(cur.value+" ");
            cur = cur.right;
        }
        //逆转回去
        reverse(head);
    }
```

