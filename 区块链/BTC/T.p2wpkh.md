P2WPKH是指付给隔离见证（Segregated Witness）和public key hash（公钥哈希）的支付类型。

- 更高的安全性、
- 更短的交易大小
- 更低的交易费用

在P2WPKH中，支付输出（UTXO）被锁定到一个公钥哈希（一段字节序列）之上。与传统的P2PKH账户不同，这里的公钥哈希是由一个特殊的Segregated Witness地址计算得来。地址以“bc1”开头，例如“bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4”。生成该地址时，使用了P2WPKH地址格式和一个公钥哈希，该公钥哈希就是从钱包中的公钥截取的160位哈希值。

在进行P2WPKH交易时，发送方需要提供输入事务的签名，并在输出脚本中提供公钥哈希和Segregated Witness数据。这个数据可以是一个压缩格式的公钥，也可以是一个签名的纯文本脚本。

P2WPKH交易具有更高的安全性，因为Segregated Witness技术使得签名数据可以被客户端验证，而无需实际储存在链上。同时，交易大小更小，因此也可以更快地完成。

由于P2WPKH地址更为紧凑，而且质保攻击成本更高，因此一些钱包和交易所已经开始采用它，并通过Segregated Witness实现更低的交易费用和更好的隐私保护。
