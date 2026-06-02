比特币交易Hash是对单笔交易内容生成的摘要信息，采用sha256算法。下面说明一笔交易含有哪些字段，字段代表什么，以及最后用这些字段值计算hash结果。

#### 交易含有字段

- 32位的 `nVersion` 
- 所有的UTXO输入 `inputs` 
- 所有的UTXO输出 `outputs` 
- 32位的 `nLockTime` 

#### 用创世区块举例：

- nVersion： `01000000` 
- inputs
  - count: `01` 
  - 1st input:
    - preout_hash: `0000000000000000000000000000000000000000000000000000000000000000` 
    - preout_n: `ffffffff` 
    - scriptSig_size: `4d` 
    - scriptSig: `04ffff001d0104455468652054696d65732030332f4a616e2f32303039204368616e63656c6c6f72206f6e206272696e6b206f66207365636f6e64206261696c6f757420666f722062616e6b73` 
    - sequence: `ffffffff` 
- outputs
  - count: `01` 
  - 1st output:
    - value: `00f2052a01000000` (hex(50*10^8) is 0000012a05f200 大小端转换
    - scriptPubkey_size: `43` 
    - scriptPubkey: `4104678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5fac` 
- nLockTime: `00000000` 

#### 拼接组成的数据块为:

01000000010000000000000000000000000000000000000000000000000000000000000000ffffffff4d04ffff001d0104455468652054696d65732030332f4a616e2f32303039204368616e63656c6c6f72206f6e206272696e6b206f66207365636f6e64206261696c6f757420666f722062616e6b73ffffffff0100f2052a01000000434104678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5fac00000000

#### 两次sha256哈希计算结果：

`3ba3edfd7a7b12b27ac72c3e67768f617fc81bc3888a51323a9fb8aa4b1e5e4a` 

#### 字节序列翻转，即为当前交易hash：

`4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b` 

```python
import hashlib
import binascii
plain = "01000000010000000000000000000000000000000000000000000000000000000000000000ffffffff4d04ffff001d0104455468652054696d65732030332f4a616e2f32303039204368616e63656c6c6f72206f6e206272696e6b206f66207365636f6e64206261696c6f757420666f722062616e6b73ffffffff0100f2052a01000000434104678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5fac00000000"
# binascii.unhexlify()：将十六进制字符串转换成二进制数据。
# hexdigest 用于将二进制表示的哈希值转换成十六进制表示的字符串。该方法返回值为一个字符串，长度为哈希算法产生哈希值的位数的两倍，因为每个字节（8 位）转换成了两个十六进制数字（每个数字 4 位）。
sha256_1 = hashlib.sha256(binascii.unhexlify(plain)).hexdigest()
sha256_2 = hashlib.sha256(binascii.unhexlify(sha256_1)).hexdigest()
print("两次计算sha256结果")
print(sha256_2, "\n")
# 3ba3edfd7a7b12b27ac72c3e67768f617fc81bc3888a51323a9fb8aa4b1e5e4a
txHash = binascii.hexlify(binascii.unhexlify(sha256_2)[::-1]).decode("utf8")
print("最终交易Hash结果")
print(txHash)
# 4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b
```
