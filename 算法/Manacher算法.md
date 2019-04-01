求字符串中的最小回文字符串

暴力的做法是：从第一个元素每次比较当前元素左右两边，得到回文串。

时间复杂度是O（n^2）

```java
 //暴力解法
    public static int baoli(String s){
        char[] charArr = manacherString(s);
        int max =  Integer.MIN_VALUE;
        for(int i=0;i<charArr.length;i++){
            int j = i-1;
            int k = i+1;
            int count = 1;
            while(j>=0&&k<charArr.length&&charArr[k]==charArr[j]){
                j--;
                k++;
                count+=2;
            }
            max = max>=count?max:count;
        }
        return max/2;
    }
```



Manacher的思路：
先了解几个概念：

hwbj[]: 回文半径数组：记录每个元素回文半径

r: 最右位置：记录回文半径时访问到的最右的回文半径，初始值为-1

mid: 该最右回文半径对应的下标

遍历每个元素的时候分为以下2种大情况：

（1）如果当前位置cur>r，则暴力递归求出hwbj[cur]

（2）如果当前位置cur<=r，再分为下面三种情况：
​	1）如果cur对应于mid的对称点（2*i-cur）的hwbj大于r-cur，那么hwbj[cur]=r-cur

​	2）如果cur对应于mid的对称点`（2*i-cur）`的hwbj小于r-cur，那么`hwbj[cur]=hwbj[2*i-cur]`

​	3）如果cur对应于mid的对称点`（2*i-cur）`的hwbj等于r-cur，那么从r之后的位置暴力递归求cur的hwbj

```JAVA
 /*
    Manacher解法
     */
    public static int manacher(String s){
        char[] charArr = manacherString(s);
        int[] hwbj = new int[charArr.length];
        int r = -1;
        int mid = 0;
        int max =0;
        for(int i=0;i<charArr.length;i++){
            //如果i<=r,那么hwbj可以加速求得，否则就是0，后面暴力去求
            hwbj[i] = i<=r?Math.min(hwbj[2*mid-i],r-i):0;
            int j = i-hwbj[i]-1;
            int k = i+hwbj[i]+1;
            while(j>=0&&k<charArr.length&&charArr[k]==charArr[j]){
                j--;
                k++;
                hwbj[i]++;
            }
            //更新r的值和mid的值
            if(k-1>r){
                r = k-1;
                mid = i;
            }
            max = max<hwbj[i]?hwbj[i]:max;
        }
        return max;
    }

//给字符串最前面加上和所有字符后面加上'#'，目的是暴力递归的时候不用处理奇偶的情况，求得回文串的长度后直接除以2即可
    public static char[] manacherString(String str) {
        char[] charArr = str.toCharArray();
        char[] res = new char[str.length() * 2 + 1];
        int index = 0;
        for (int i = 0; i != res.length; i++) {
            res[i] = (i & 1) == 0 ? '#' : charArr[index++];
        }
        return res;
    }
```



时间复杂度分析：
其他的不用管，r的位置一直在增加，所以时间复杂度是O(n)



相关问题：

问题描述

　　吉哥又想出了一个新的完美队形游戏！
　　假设有n个人按顺序站在他的面前，他们的身高分别是h [1]，h [2] ... h [n]，吉哥希望从中挑出一些人，让这些人形成一个新的队形，新的队形若满足以下三点要求，则就是新的完美队形：

　　1，挑出的人保持原队形的相对顺序不变，且必须都是在原队形中连续的; 
　　2，左右对称，假设有m个人形成新的队形，则第1个人和第m个人身高相同，第2个人和第m-1个人身高相同，依此类推，当然如果m是奇数，中间那个人可以任意; 
　　3，从左到中间那个人，身高需保证不下降，如果用H表示新队形的高度，则H [1] <= H [2] <= H [3] .... <= H [mid]。

　　现在吉哥想知道：最多能选出多少人组成新的完美队形呢？



思路：这是一个类似求最大回文串长度的问题，不过加多了第三个条件，我们在暴力求的时候记录下前一个值判断一下大小即可

```java
class 完美队形{
    public static void main(String[] args){
        System.out.println(manacher(new int[]{51,44,51,52,51,44,51}));
    }
    //还是加一些特殊元素进去处理奇偶的情况
    public static int[] addChar(int[] s){
        int[] newArr = new int[s.length*2+1];
        int i=0,j=0;
        boolean flag = false;
        while(j<newArr.length){
            if(!flag){
                newArr[j++] = -1;
            }else{
                newArr[j++] = s[i++];
            }
            flag = !flag;
        }
        return newArr;
    }
    //Manacher
    public static int manacher(int[] num){
        if(num==null)return 0;
        int[] arr = addChar(num);
        int max = 0;
        int mid = 0;
        int r = -1;
        int[] hwbj = new int[arr.length];
        for(int i=0;i<hwbj.length;i++){
            hwbj[i] = i<=r?Math.min(hwbj[2*mid-i],r-i):0;
            int left = i-hwbj[i]-1;
            int right = i+hwbj[i]+1;
            //pre记录一下前面的值
            int pre = arr[i];
            while(left>-1&&right<hwbj.length&&arr[left]==arr[right]){
                //如果前面是填充的值，直接判断就行    
                if(arr[left]==-1){
                        left--;
                        right++;
                        hwbj[i]++;
                    }
                //否则判断下是否非递减的
                 else{
                        if(pre==-1)pre = arr[left];
                        if(arr[left]<=pre){
                            pre = arr[left];
                            left--;
                            right++;
                            hwbj[i]++;
                        }
                        else{
                            break;
                        }
                    }
            }
            if(right-1>r){
                r = right-1;
                mid = i;
            }
            max = max<hwbj[i]?hwbj[i]:max;
        }
        return max;
    }
}
```

