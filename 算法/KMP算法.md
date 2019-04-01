有两个字符串，A,B，问A是不是B的子字符串

暴力做法：每次B向后移动一个字符，和A比较，时间复杂度是O（n*m）

而在KMP算法中，对于每一个模式串我们会事先计算出模式串的内部匹配信息，在匹配失败时最大的移动模式串，以减少匹配次数。KMP算法根据模式串的匹配信息，加速了比较的过程。

得到模式串的内部消息的复杂度是O（m），目标串匹配的时间复杂度是O（n），所以总的时间复杂度是O（n+m）。

问题1：有两个字符串，A,B，问A是不是B的子字符串

```java
//得到模式串的内部匹配信息：next[]数组
public static int[] getNext(char[] pattern){
    if(pattern.length==1)return new int[]{-1};
    int[] next = new int[pattern.length];
    //这个数组记录了当前下标相等的最大前缀和最大后缀的长度，其实就是前缀的下一个位置
    next[0] = -1;
    next[1] = 0;
    int i=2,cn = 0;
    //i是遍历的下标
    //cn是当前最大匹配的长度
    while(i<pattern.length){
        if(pattern[i-1]==next[cn]){
            next[i++] = ++cn;
        }else if(cn!=0){
            cn = next[cn];
        }else{
            i++;
        }
    }
    return next;
}

//比较两个字符串
public static boolean kmp(String s,String pattern){
    if(pattern==null){
        if(s!=null)return false;
        else return true;
    }else if(s==null){
        if(pattern!=null)return false;
        else return true;
    }
    int i = 0;
    int j = 0;
    char[] ss = s.toCharArray();
    char[] pp = pattern.toCharArray();
    //拿到next数组
    int[] next = getNext(pp);
    while(i<s.length()&&j<pattern.length()){
        if(ss[i]==pp[j]){
            i++;
            j++;
        }else if(j!=0){
            //如果不相等，不用从头开始，而是从最长前缀后面继续比较
            j = next[j];
        }else{
            //如果不能再往前找了，而且不相等，只能i往后移动
            i++;
        }
    }
    return j==pattern.length();
}
```

问题2：有两个字符串A,B，问A在B中出现的次数，比如ABA，在ABABABA中出现了3次

````java
public static int[] getNext(char[] cc){
        int i = 2;
        int cn = 0;
        int[] next = new int[cc.length+1];
        while(i<=cc.length){
            if(cc[i-1]==cc[cn]){
                next[i++] = ++cn;

            }else if(cn!=0){
                cn = next[cn];
            }else{
                i++;
            }
        }
        return next;
    }

    public static int count(String a,String b){
        char[] aa = a.toCharArray();
        char[] bb = b.toCharArray();
        int[] next = getNext(aa);
        int i=0;
        int j=0;
        int count = 0;
        while(i<b.length()){
            if(bb[i]==aa[j]){
                i++;
                j++;
            }
            else if(j!=0){
                j = next[j];
            }
            else{
                i++;
            }
            //如果A已经匹配成功，count++，然后j回到前一个，接着和B比对
            if(j==aa.length){
                count++;
                j = next[j];
            }
        }
        return count;
    }
````

问题3:判断一个字符串是否是另一个字符串的重复次组成的，比如abababab是由ab重复组成的

```java
/**
 * 思路是n的位置的n%(n-next(n))是否为0，如果是的话那n-next(n)到n的位置就是那个重复的子字符串
 */
class PowerString{
    public  static void main(String[] args){
        String ss = "abcabcabcabc";
        System.out.println(getPattern(ss));
    }

    public static int[] getNext(char[] cc){
        if(cc.length==1)return new int[]{-1};
        int[] next = new int[cc.length+1];
        next[0] = -1;
        next[1] = 0;
        int i=2;
        int cn = 0;
        while(i<=cc.length){
            if(cc[i-1]==cc[cn]){
                next[i++] = ++cn;
            }else if(cn!=0){
             cn = next[cn];
            }else{
                i++;
            }
        }
        return next;
    }

    public static  String getPattern(String s){
        if(s==null||s.length()<=1){
            return s;
        }
        int[] next = getNext(s.toCharArray());
        if(s.length()%(s.length()-next[s.length()])==0){
            return s.substring(next[s.length()]);
        }else{
            return null;
        }
    }
}
```