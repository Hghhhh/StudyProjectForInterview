#Leetcode72 最小编辑距离

给定两个单词 word1 和 word2，计算出将 word1 转换成 word2 所使用的最少操作数 。

你可以对一个单词进行如下三种操作：

插入一个字符
删除一个字符
替换一个字符
示例 1:

输入: word1 = "horse", word2 = "ros"
输出: 3
解释: 
horse -> rorse (将 'h' 替换为 'r')
rorse -> rose (删除 'r')
rose -> ros (删除 'e')
示例 2:

输入: word1 = "intention", word2 = "execution"
输出: 5
解释: 
intention -> inention (删除 't')
inention -> enention (将 'i' 替换为 'e')
enention -> exention (将 'n' 替换为 'x')
exention -> exection (将 'n' 替换为 'c')

exection -> execution (插入 'u')



思路是动态规划，`dp[i][j]`中的i，j表示把word1前i个字符变成word2前j个字符

显然`dp[i][0]=i`，`dp[0][j]=j`

![72](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/leetcode/72.png)



```java
public class 最小编辑距离 {

    public static  int minDistance(String strword1, String strword2) {
            int len1 = strword1.length();
            int len2 = strword2.length();
            char[] word1 = strword1.toCharArray();
            char[] word2 = strword2.toCharArray();
            int[][] dp = new int[len1+1][len2+1];
            for(int i=0;i<=len1;i++)dp[i][0] = i;
            for(int j=0;j<=len2;j++)dp[0][j] = j;
            for(int i=1;i<=len1;i++){
                for(int j=1;j<=len2;j++){
                    if(word1[i-1]==word2[j-1])dp[i][j] = dp[i-1][j-1];
                    else {
                        dp[i][j] = Math.min(dp[i-1][j-1],Math.min(dp[i-1][j],dp[i][j-1]))+1;
                    }
                }
            }
            return dp[len1][len2];
        }
}
```

