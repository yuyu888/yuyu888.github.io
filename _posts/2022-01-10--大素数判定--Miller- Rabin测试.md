---
layout: mypost
title: 大素数判定--Miller- Rabin测试
categories: [算法思想]
---

## 素数定义

一个数是素数（也叫质数），当且仅当它的约数只有两个——1和它本身。规定这两个约数不能相同，因此1不是素数

## 素数的性质

在研究RSA算法的时候， 关键问题都跟素数有关；包括各种Hash算法也是围绕素数转来转去，对于写代码的人来说，素数比 想像中的更重要；

一些好记的素数：4567, 124567, 3214567, 23456789, 55566677, 1234567894987654321, 11111111111111111111111 (23个1)

### 素数有很多神奇的性质:

### 素数的个数无限多（不存在最大的素数）
证明：反证法，假设存在最大的素数P，那么我们可以构造一个新的数2 * 3 * 5 * 7 * … * P + 1（所有的素数乘起来加1）。显然这个数不能被任一素数整除（所有素数除它都余1），这说明我们找到了一个更大的素数。

### 存在任意长的一段连续数，其中的所有数都是合数（相邻素数之间的间隔任意大）

证明：当0 < a <= n时，n!+a能被a整除。长度为n-1的数列n!+2, n!+3, n!+4, …, n!+n中，所有的数都是合数。这个结论对所有大于1的整数n都成立，而n可以取到任意大。

### 所有大于2的素数都可以唯一地表示成两个平方数之差。

证明：大于2的素数都是奇数。假设这个 数是2n+1。由于(n+1)<sup>2</sup>=n<sup>2</sup>+2n+1，(n+1)<sup>2</sup>和n<sup>2</sup>就是我们要找的两个平方数。下面证明这个方案是唯一的。如果素数p能表示成 a<sup>2</sup>-b<sup>2</sup>，则p=a<sup>2</sup>-b<sup>2</sup>=(a+b)(a-b)。由于p是素数，那么只可能a+b=p且a-b=1，这给出了a和b的唯一解。

### 当n为大于2的整数时，2<sup>n</sup>+1和2<sup>n</sup>-1两个数中，如果其中一个数是素数，那么另一个数一定是合数。

证明：2<sup>n</sup>不能被3整除。如果它被3除余1，那么2<sup>n</sup>-1就能被3整除；如果被3除余2，那么2<sup>n</sup>+1就能被3整除。总之，2<sup>n</sup>+1和2<sup>n</sup>-1中至少有一个是合数。

### 如果p是素数，a是小于p的正整数，a,p互质，那么a<sup>(p-1)</sup> mod p = 1。

这个就是 **费马小定理**， 本来想自己证明一下，但是知识储备不够又怕陷入循环论证， 记住就好

Euler对这个定理进行了推广，叫做 **Euler定理**。Euler一生的定理太多了，为了和其它的“Euler定理”区别开来，有些地 方叫做 **Fermat小定理的Euler推广**

**欧拉定理**：如果a, n互质， 则： a<sup>φ(N)</sup> % N =1 ； 如果N是质数，则 φ(N) = N-1

## Fermat素性测试

谈到Fermat小定理，数学历史上有很多误解。很长一段时间里，人们都认为Fermat小定理的逆命题是正确的，并且有人亲自验证了 a=2, p<300的所有情况。国外甚至流传着一种说法，认为中国在孔子时代就证明了这样的定理：如果n整除2<sup>(n-1)</sup>-1，则n就是素数。后来某个英 国学者进行考证后才发现那是因为他们翻译中国古文时出了错。

1819年有人发现了Fermat小定理逆命题的第一个反例：虽然2的340次方除以341余 1，但341 = 11 * 31。后来，人们又发现了561, 645, 1105等数都表明a=2时Fermat小定理的逆命题不成立。虽然这样的数不多，但不能忽视它们的存在。于是，人们把所有能整除2<sup>(n-1)</sup>-1的合 数n叫做伪素数(pseudoprime)，意思就是告诉人们这个素数是假的。

不满足2<sup>(n-1)</sup> mod n = 1的n一定不是素数；如果满足的话则多半是素数。这样，一个比试除法效率更高的素性判断方法出现了：制作一张伪素数表，记录某个范围内的所有伪素数，那么 所有满足2<sup>(n-1)</sup> mod n = 1且不在伪素数表中的n就是素数。之所以这种方法更快，是因为我们可以使用二分法快速计算2<sup>(n-1)</sup> mod n 的值，这在计算机的帮助下变得非常容易；在计算机中也可以用二分查找有序数列、Hash表开散列、构建Trie树等方法使得查找伪素数表效率更高。
有 人自然会关心这样一个问题：伪素数的个数到底有多少？换句话说，如果我只计算2<sup>(n-1)</sup> mod n的值，事先不准备伪素数表，那么素性判断出错的概率有多少？研究这个问题是很有价值的，毕竟我们是OIer，不可能背一个长度上千的常量数组带上考场。 统计表明，在前10亿个自然数中共有50847534个素数，而满足2<sup>(n-1)</sup> mod n = 1的合数n有5597个。这样算下来，算法出错的可能性约为0.00011。这个概率太高了，如果想免去建立伪素数表的工作，我们需要改进素性判断的算 法。

最简单的想法就是，我们刚才只考虑了a=2的情况。对于式子a<sup>(n-1)</sup> mod n，取不同的a可能导致不同的结果。一个合数可能在a=2时通过了测试，但a=3时的计算结果却排除了素数的可能。于是，人们扩展了伪素数的定义，称满足 a<sup>(n-1)</sup> mod n = 1的合数n叫做以a为底的伪素数(pseudoprime to base a)。前10亿个自然数中同时以2和3为底的伪素数只有1272个，这个数目不到刚才的1/4。这告诉我们如果同时验证a=2和a=3两种情况，算法出错 的概率降到了0.000025。容易想到，选择用来测试的a越多，算法越准确。通常我们的做法是，随机选择若干个小于待测数的正整数作为底数a进行若干次 测试，只要有一次没有通过测试就立即把这个数扔回合数的世界。这就是Fermat素性测试。

## Carmichael数

人们在用费马小定理做素性测试的时候自然会想，如果考虑了所有小于n的底数a，出错的概率是否就可以降到0呢？没想 到的是，居然就有这样的合数，它可以通过所有a的测试（这个说法不准确）。Carmichael第一个发现这样极端的伪素数，他 把它们称作Carmichael数。你一定会以为这样的数一定很大。错。第一个Carmichael数小得惊人，仅仅是一个三位数，561。前10亿个自 然数中Carmichael数也有600个之多。Carmichael数的存在说明，我们还需要继续加强素性判断的算法。

## Miller-Rabin素性测试

Miller和Rabin两个人的工作让Fermat素性测试迈出了革命性的一步，建立了传说中的Miller-Rabin素性测试算法。 新的测试基于下面的定理：如果p是素数，x是小于p的正整数，且x<sup>2</sup> mod p = 1，那么要么x=1，要么x=p-1。这是显然的，因为x<sup>2</sup> mod p = 1相当于p能整除x<sup>2</sup>-1，也即p能整除(x+1)(x-1)。由于p是素数，那么只可能是x-1能被p整除(此时x=1)或x+1能被p整除(此时 x=p-1)。

我们下面来演示一下上面的定理如何应用在Fermat素性测试上。前面说过341可以通过以2为底的Fermat测试，因 为2<sup>340</sup> mod 341=1。如果341真是素数的话，那么2<sup>170</sup> mod 341只可能是1或340；当算得2<sup>170</sup> mod 341确实等于1时，我们可以继续查看2<sup>85</sup>除以341的结果。我们发现，2<sup>85</sup> mod 341=32，这一结果摘掉了341头上的素数皇冠，面具后面真实的嘴脸显现了出来，想假扮素数和我的素MM交往的企图暴露了出来。

这就 是Miller-Rabin素性测试的方法。不断地提取指数n-1中的因子2，把n-1表示成d * 2<sup>r</sup>（其中d是一个奇数）。那么我们需要计算的东西就 变成了a的d * 2<sup>r</sup>次方除以n的余数。于是，a<sup>(d * 2<sup>r-1</sup>)</sup>要么等于1，要么等于n-1。如果a<sup>(d * 2<sup>r-1</sup>)</sup>等于1，定理继续适用于a<sup>(d * 2<sup>r-2</sup>)</sup>，这样不断开方开下去，直到对于某个i满足a<sup>(d * 2<sup>i</sup>)</sup> mod n = n-1或者最后指数中的2用完了得到的a<sup>d</sup> mod n=1或n-1。  

这样，Fermat小定理加强为如下形式：  
尽可能提取因子2， 把n-1表示成d * 2<sup>r</sup>，如果n是一个素数，那么或者a<sup>d</sup> mod n=1，或者存在某个i使得a<sup>(d * 2<sup>i</sup>)</sup> mod n=n-1 ( 0 <= i < r ) （注意i可以等于0，这就把a<sup>d</sup> mod n=n-1的情况统一到后面去了）

Miller-Rabin 素性测试同样是不确定算法，我们把可以通过以a为底的Miller-Rabin测试的合数称作以a为底的强伪素数(strong pseudoprime)。第一个以2为底的强伪素数为2047。第一个以2和3为底的强伪素数则大到1 373 653。

对于大数的素性判断，目前Miller-Rabin算法应用最广泛。一般底数仍然是随机选取，但当待测数不太大时，选择测试底数就有一些技巧了。比如，如果 被测数小于4 759 123 141，那么只需要测试三个底数2, 7和61就足够了。当然，你测试的越多，正确的范围肯定也越大。如果你每次都用前7个素数(2, 3, 5, 7, 11, 13和17)进行测试，所有不超过341 550 071 728 320的数都是正确的。如果选用2, 3, 7, 61和24251作为底数，那么10<sup>16</sup>内唯一的强伪素数为46 856 248 255 981。这样的一些结论使得Miller-Rabin算法在OI中非常实用。通常认为，Miller-Rabin素性测试的正确率可以令人接受，随机选取 k个底数进行测试算法的失误率大概为4<sup>(-k)</sup>。


转自：http://www.matrix67.com/blog/archives/234
