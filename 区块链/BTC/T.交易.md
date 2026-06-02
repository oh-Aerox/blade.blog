比特币协议规定，申报交易的时候，除了交易金额，转出比特币的一方还必须提供以下数据。
上一笔交易的 Hash（你从哪里得到这些比特币）
本次交易双方的地址
支付方的公钥
支付方的私钥生成的数字签名

验证这笔交易是否属实，需要三步。
第一步，找到上一笔交易，确认支付方的比特币来源。
第二步，算出支付方公钥的指纹，确认与支付方的地址一致，从而保证公钥属实。
第三步，使用公钥去解开数字签名，保证私钥属实。

经过上面三步，就可以认定这笔交易是真实的。

# 输出
交易输出包含两部分：

- 一定量的比特币，面值为“聪”（satoshis） ，是最小的比特币单位；
- 确定花费输出所需条件的加密难题（cryptographic puzzle）这个加密难题也被称为锁定脚本(locking script), 见证脚本(witness script), 或脚本公钥 (scriptPubKey)。
# 输入

- 一个交易 ID，引用包含正在使用的 UTXO 的交易
- 一个输出索引（vout），用于标识来自该交易的哪个 UTXO 被引用（第一个为零）
- 一个 scriptSig（解锁脚本），满足放置在 UTXO 上的条件，解锁它用于支出
- 一个序列号

[https://www.cnblogs.com/studyzy/p/how-bitcoin-sign-a-tx.html](https://www.cnblogs.com/studyzy/p/how-bitcoin-sign-a-tx.html)

