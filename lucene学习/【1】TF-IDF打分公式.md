lucene打分机制是采用空间向量模型的，即计算Query与Doc这两个向量的cos值

![image-20210401110510503](https://raw.githubusercontent.com/ppb2/note/main/imgs/%E7%A9%BA%E9%97%B4%E5%90%91%E9%87%8F%E6%A8%A1%E5%9E%8BVSM.png)

我们把一个文档中的词（term）的权重看作一个向量
Document = {term1,term2,...,termN}
Document Vector= {weight1,weight2,...,weightN}

同样我们把Query也用作向量表示
Query = {term1,term2,...,termN}
Query Vector= {weight1,weight2,...,weightN}

Document向量Vd=<W(t1,d),W(t2,d),...,W(tn,d)>
Query向量Vq=<W(t1,q),W(t2,q),...,W(tn,q)>

![image-20210401141708966](https://raw.githubusercontent.com/ppb2/note/main/imgs/%E4%BD%99%E5%BC%A6%E5%85%AC%E5%BC%8F.png)

Vq*Vd = w(t1, q)*w(t1, d) + w(t2, q)*w(t2, d) + ...... + w(tn ,q)*w(tn, d)

w 代表 weight，计算公式一般为 tf*idf。
Vq*Vd = tf(t1, q)*idf(t1, q)*tf(t1, d)*idf(t1, d) + tf(t2, q)*idf(t2, q)*tf(t2, d)*idf(t2, d) + ...... + tf(tn ,q)*idf(tn, q)*tf(tn, d)*idf(tn, d)

+ 由于是点积，则此处的 t1, t2, ......, tn 只有查询语句和文档的并集有非零值，只在查询语句出现的或只在文档中出现的 Term 的项的值为零。
+ 在查询的时候，很少有人会在查询语句中输入同样的词，因而可以假设 tf(t, q)都为 1
+ idf 是指 Term 在多少篇文档中出现过，其中也包括查询语句这篇小文档，因而 idf(t, q)和 idf(t, d)其实是一样的，是索引中的文档总数加一，当索引中的文档总数足够大的时候，查询语句这篇小文档可以忽略，因而可以假设 idf(t, q) = idf(t, d) = idf(t)

Vq*Vd = tf(t1, d) * idf(t1) * idf(t1) + tf(t2, d) * idf(t2) * idf(t2) + ...... + tf(tn, d) * idf(tn) * idf(tn)

![image-20210401142035404](https://raw.githubusercontent.com/ppb2/note/main/imgs/tf-idf%E6%8E%A8%E5%AF%BC.png)

由上面的讨论，查询语句中 tf 都为 1，idf 都忽略查询语句这篇小文档，得到如下公式

![image-20220323112845041](https://raw.githubusercontent.com/ppb2/note/main/imgs/image-20220323112845041.png)

此时得到的打分公式如下

![image-20220323112938030](https://raw.githubusercontent.com/ppb2/note/main/imgs/image-20220323112938030.png)

近一步推导

![image-20220323113037079](https://raw.githubusercontent.com/ppb2/note/main/imgs/image-20220323113037079.png)

如果按照标准的余弦计算公式，完全消除文档长度的影响，则又对长文档不公平(毕竟 它是包含了更多的信息)，偏向于首先返回短小的文档的，这样在实际应用中使得搜索结果 很难看。

![image-20220323113555672](https://raw.githubusercontent.com/ppb2/note/main/imgs/image-20220323113555672.png)

![image-20220323113641484](https://raw.githubusercontent.com/ppb2/note/main/imgs/image-20220323113641484.png)



### lucene的实际打分公式

![image-20220323150019148](https://raw.githubusercontent.com/ppb2/note/main/imgs/image-20220323150019148.png)

- Document boost：此值越大，说明此文档越重要。 
- Field boost：此域越大，说明此域越重要。  
- lengthNorm(field) = (1.0 / Math.sqrt(numTerms))：一个域中包含的 Term 总数越多，也即 文档越长，此值越小，文档越短，此值越大。  
- 其中第三个参数可以在自己的 Similarity 中影响打分，下面会论述。 
- 当 然 ， 也 可 以 在 添 加 Field 的 时 候 ， 设 置 Field.Index.ANALYZED_NO_NORMS 或 Field.Index.NOT_ANALYZED_NO_NORMS，完全不用 norm，来节约空间。没有 norms 意味着 索引阶段禁用了文档 boost 和域的 boost 及长度标准化。好处在于节省内存，不用在搜索阶 段为索引中的每篇文档的每个域都占用一个字节来保存 norms 信息了。但是对 norms 信息 的禁用是必须全部域都禁用的，一旦有一个域不禁用，则其他禁用的域也会存放默认的 norms 值。因为为了加快 norms 的搜索速度，Lucene 是根据文档号乘以每篇文档的 norms 信息所占用的大小来计算偏移量的，中间少一篇文档，偏移量将无法计算。也即 norms 信 息要么都保存，要么都不保存。
- queryNormal：这是按照向量空间模型，对 query 向量的归一化。此值并不影响排序，而仅仅使得不同的 query 之间的分数可以比较。
- coord：一次搜索可能包含多个搜索词，而一篇文档中也可能包含多个搜索词，此项表示，当一篇文 档中包含的搜索词越多，则此文档则打分越高。



