# sim-hash算法

sim-hash是谷歌推出的一款局部敏感哈希算法（LSH），主要用于海量文本查重。

谷歌的应用场景：搜索引擎每天需要获取大量的网页，这样网页中有很多是近重复的，比如镜像网站、内容复制、嵌入广告、少量修改的网页，试想假如我们搜索出来的网页里面包含大量重复信息，我们需要翻多少页才能看到更多不一样的内容呢，因此谷歌需要对获取的网页与它原有的数以亿计的信息库进行比对去重。

## 算法原理

对于web文档，我们考虑将其通过算法抽象成一串固定位数的哈希串，称为这个文档的指纹（fingerprint），接下来我们通过对各个文档的指纹进行比对，计算距离，比如计算哈希串之间的海明距离，将距离小于某个值的文档视为相似文档即可。

sim-hash算法大概就是基于这种思想，它将文档抽象为一串01串，算法的实现步骤如下：

1. 选择sim-hash串的位数，比如说64bit；
2. 分词，读取文档，将文档中的内容分词；
3. 哈希，使用hash函数计算得到的各个分词的hash串，hash串长度和sim-hash串的长度一致；
4. 加权，把各个分词的hash串按照权重加权处理，如果第i位为1，则+w，否则-w；
5. 合并，对各个分词的hash串相加，得到一个串；
6. 降维，对合并得到的串，如果第i位大于0，则sim-hash串第i位为1，则为0。

最后得到一个sim-hash串即为文档的指纹。

整个过程可以参考下图：

![原理如下](http://dl.iteye.com/upload/attachment/437426/baf42378-e625-35d2-9a89-471524a355d8.jpg)

## 算法效率提高

对于海量的文档，虽然得到了它们的指纹，但是还要计算它们之间的相似度，比如64位的哈希串，如何在海量文档中找到与其海明距离在3以内的文档呢？有两种常规思路：

1. 第一种是查找待查询文本的64位串的所有3位以内变化的组合，大约需要四万多次的查询；
2. 第二种是预先计算出这64位串的所有3位以内变化的组合，需要四万多倍的原始空间。

显然这两种都不好时间和空间复杂度太高，如果我们要找距离为3以内的相似文档，我们将这64位串均分为4块，根据鸽笼原理，匹配的相似文档中必有一块完全相同。

![鸽笼原理](http://dl.iteye.com/upload/attachment/437559/689719df-54b7-318c-bc90-e289f84344b9.jpg)

## 代码实现

根据上面的原理分析，我实现了如下代码：

```java
public class SimHashUtil {

    private static int hashbits = 64;

    public static String simHash(String tokens) throws IOException {
        // 定义特征向量/数组
        int[] v = new int[hashbits];
        // 1、中文分词，分词器采用 IKAnalyzer3.2.8
        StringReader reader = new StringReader(tokens);
        // 当为true时，分词器进行最大词长切分
        IKSegmentation ik = new IKSegmentation(reader, true);
        Lexeme lexeme = null;
        String word = null;
        String temp = null;
        while ((lexeme = ik.next()) != null) {
            word = lexeme.getLexemeText();
            // 2、将每一个分词hash为一组固定长度的数列.比如 64bit 的一个整数.
            BigInteger t = hash(word);
            for (int i = 0; i < hashbits; i++) {
                BigInteger bitmask = new BigInteger("1").shiftLeft(i);
                // 3、建立一个长度为64的整数数组(假设要生成64位的数字指纹,也可以是其它数字),
                // 对每一个分词hash后的数列进行判断,如果是1000...1,那么数组的第一位和末尾一位加1,
                // 中间的62位减一,也就是说,逢1加1,逢0减1.一直到把所有的分词hash数列全部判断完毕.
                if (t.and(bitmask).signum() != 0) {
                    // 这里是计算整个文档的所有特征的向量和
                    // 这里实际使用中需要 +- 权重，比如词频，而不是简单的 +1/-1，
                    v[i] += 1;
                } else {
                    v[i] -= 1;
                }
            }
        }

        StringBuffer strSimHash = new StringBuffer();
        for (int i = 0; i < hashbits; i++) {
            // 4、最后对数组进行判断,大于0的记为1,小于等于0的记为0,得到一个 64bit 的数字指纹/签名.
            if (v[i] >= 0) {
                strSimHash.append("1");
            }else{
                strSimHash.append("0");
            }
        }
        return strSimHash.toString();
    }

    private static BigInteger hash(String source) {
        if (source == null || source.length() == 0) {
            return new BigInteger("0");
        } else {
            char[] sourceArray = source.toCharArray();
            BigInteger x = BigInteger.valueOf(((long) sourceArray[0]) << 7);
            BigInteger m = new BigInteger("1000003");
            BigInteger mask = new BigInteger("2").pow(hashbits).subtract(new BigInteger("1"));
            for (char item : sourceArray) {
                BigInteger temp = BigInteger.valueOf((long) item);
                x = x.multiply(m).xor(temp).and(mask);
            }
            x = x.xor(new BigInteger(String.valueOf(source.length())));
            if (x.equals(new BigInteger("-1"))) {
                x = new BigInteger("-2");
            }
            return x;
        }
    }

    public static int getDistance(String str1, String str2) {
        int distance;
        if (str1.length() != str2.length()) {
            distance = -1;
        } else {
            distance = 0;
            for (int i = 0; i < str1.length(); i++) {
                if (str1.charAt(i) != str2.charAt(i)) {
                    distance++;
                }
            }
        }
        return distance;
    }

    /**
     * 将hashbits位的指纹分为（distance+1）块
     */
    public static List<BigInteger> subByDistance(String strSimHash, int distance) {
        // 分成几组来检查
        int numEach = hashbits / (distance + 1);
        List<BigInteger> characters = new ArrayList<>();

        StringBuffer buffer = new StringBuffer();

        int k = 0;
        for (int i = 0; i < strSimHash.length(); i++) {
            // 当且仅当设置了指定的位时，返回 true
            boolean sr = strSimHash.charAt(i) == '1';

            if (sr) {
                buffer.append("1");
            } else {
                buffer.append("0");
            }

            if ((i + 1) % numEach == 0) {
                // 将二进制转为BigInteger
                BigInteger eachValue = new BigInteger(buffer.toString(), 2);
                buffer.delete(0, buffer.length());
                characters.add(eachValue);
            }
        }

        return characters;
    }

}
```









