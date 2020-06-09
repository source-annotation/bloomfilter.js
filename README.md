# bloomfilter.js 阅读版

源码阅读版 (带着问题去读源码) 。 

[Document](README-en.md) 

# Sched   

- [x] 为什么需要 k 个哈希函数 ?  
- [x] 为什么说 bloom filter 能减少空间？ 
- [x] 为什么常用 fnv hash 用来做哈希映射(concurrent-map也是) ?  
- [x] 如何用一次哈希模拟出  k 个 hash function 的效果?  
- [x] Bloom filter 有哪些出名的实践(业务上比如缓存？ 数据库上比如levelDb, bittable?)?   
- [x] 什么时候用 bloom filter ?   
- [x] k 为多少最合适？  


# Q&A

## #0 为什么会出现这个项目 ?  

这个项目看起来就是使用了 fnv 来做哈希映射的 bloom filter 。bloom filter 使用的哈希函数越多速度就越慢，但使用的哈希函数过少又会导致误判率增加。

这个项目看起来就是换了种哈希算法提升了速度(bloomfilter 从 md5 换成 murmurhash 速度提升了 800%)。  

## #1 `bitmap` vs `bloom filter`

bloom filter : 原生的 bloom filter 只支持 add / get，被魔改过的 cuckoo filter，counting filter 支持 delete。  
bitmap : key 仅支持 int 型。 

## #2 为什么常用 fnv hash 用来做哈希映射(concurrent-map也是) ? 
对我来说， fnv 出镜率还蛮高的： 

1. golang 内置的 4 种 hash function 之一 `$GOSRC/hash/`。  
2. concurrent-map 这个 golang 项目里 fnv 被用来做 key => shard 位置的映射。   
3. bloom filter 的各个版本实现里也看到好多人用 fnv_1a 来取代默认的哈希函数(比如本项目)。  
 
用哈希函数来做映射的话，我们会一般会选用 non-cryptographic hash function，但即便这样可选的也有很种 :  murmurhash, fnv, crc32 ...等等。 由于仅仅是做映射，所以需求很简单： 

1. 低冲突率  
2. 运算速度块  
3. 分布均匀 
4. ...(欢迎补充)

只是很难有一个哈希算法可以满足所有要求，冲突率低的慢，而快的就冲突率高，不然就是分布不均。 

综合下来 : murmurhash 很优秀，在一致性哈希算法里出镜率挺高的。 不过 fnv 也不赖，虽然不能说 fnv 是最好的，但目前来说确实是主流的之一，具体用什么，各有取舍吧。

## #3 为什么需要 k 个哈希函数  

降低哈希冲突概率。 

假设你只有一个哈希函数 hash1， `hash1("hello") => 0000 1000`， `hash1("bloom") => 0000 1000` 这样就发生哈希冲突了。  

假设你有2个哈希函数 hash1 和 hash2 : 

```
hash1("hello") => 0000 1000  
hash2("hello") => 0100 0000 

hash1("bloom") => 0000 1000 
hash2("bloom") => 0001 0000
``` 
那么此时 hello 映射到 bit array 中就是 `0100 1000`，bloom 映射到 bit array 就是 `0001 1000`，除非 hash1 和 hash2 就这么巧 2 个 key 两次哈希都映射到同一个位上了，否则 hello 和 bloom 就不可能会发生冲突。 

这样就能有效降低冲突率，但是面对巨大的数据量，还是有冲突的可能性。 所以这里的 k 取多少取决于你对准确率的要求， k 越多越不易发生冲突，但运算速度就越慢。反之 k 越少越容易发生冲突，但运算速度就越快。

## #4 如何用一次哈希模拟出  k 个 hash function 的效果 ?  

该 repo 参考了 :  

* [Producing n hash functions by hashing only once](http://willwhim.wpengine.com/2011/09/03/producing-n-hash-functions-by-hashing-only-once/)   
* [Less Hashing, Same Performance:
   Building a Better Bloom Filter](http://citeseer.ist.psu.edu/viewdoc/download;jsessionid=4060353E67A356EF9528D2C57C064F5A?doi=10.1.1.152.579&rep=rep1&type=pdf)  
 
具体看不太懂，暂时不看了。

## #5 Bloom filter 有哪些出名的实践 ? 

* Google Bigtable : 减少磁盘 I/O
* LevelDB     
* 爬虫 : 判断一个 url 是否被已爬过  
* 缓存 : 在缓存 key 数量非常多时，用来挡在缓存前面, 先判断缓存 key 是否存在。
* ...(欢迎补充) 

## #6 什么时候用 bloom filter ?   

个人看法：

经常看到用于应对缓存危机，当缓存key非常多时，确实可以挡在缓存前。 业务量不大到一定程度，用不上布隆过滤器，不必为了用而用，哈希表完全够用。  其余的参考 #5。          

## #7 为什么说 bloom filter 能减少空间？  

bloom filter 妙在用了 k 个哈希函数。假设你只有 1 个哈希函数，那么为了避免哈希冲突，你可能会把 bit array 的位的数量设置的非常大，占用空间就大了。  

用了 k 个哈希函数，哈希冲突概率减少了，你就不必设定那么大的 bit array 了。  

## #8 k 为多少最合适？   

见 [Bloom Filters by Example](https://llimllib.github.io/bloomfilter-tutorial/zh_CN/) 里的 **"应该使用多少个哈希函数?"** 。 
# 参考 

> [Bloom Filters by Example](https://llimllib.github.io/bloomfilter-tutorial/zh_CN/) : 一个很不错的教程.   
> [布隆过滤器(Bloom Filter)详解——基于多hash的概率查找思想](https://www.cnblogs.com/bonelee/p/6215176.html)  
> [Bloom Filters](https://www.jasondavies.com/bloomfilter/)