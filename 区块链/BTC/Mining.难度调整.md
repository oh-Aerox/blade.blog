比特币网络通过动态地调整工作量证明（Proof of Work，PoW）的难度来保持块产出速率约为每10分钟一个块。这个机制旨在确保整个网络的稳定性，以及避免块产出速度过快或过慢。

具体来说，比特币的难度调整是根据前一个难度周期内的块产出速率进行计算的。难度调整的过程如下：

1. **计算周期：** 一个难度调整周期通常是2016个块，这大约等于两周的时间。比特币网络会监控前一个周期内实际产生的块数。 
2. **目标产出速率：** 比特币网络的目标是每10分钟产生一个块。如果前一个周期内的块产出速率大于10分钟一个块，说明网络的总算力增加，难度将会上调。反之，如果块产出速率低于10分钟一个块，说明网络的总算力减少，难度将会下调。 
3. **难度调整计算：** 难度调整计算涉及一个“调整因子”，它是实际块产出速率与目标块产出速率的比率。如果调整因子大于1，难度将上调；如果小于1，难度将下调。然后，新的难度将应用于下一个难度周期。 

通过这种机制，比特币网络可以自适应地调整难度，以确保每10分钟一个块的稳定块产出速率。这对于网络的安全性和稳定性非常重要，因为它确保了不断变化的算力和矿工兴趣不会导致产出速率过快或过慢。

以下转自：[https://zhuanlan.zhihu.com/p/32739785](https://zhuanlan.zhihu.com/p/32739785)

# 目标值

区块头中的Bits字段，标识了当前区块头Hash之后要小于等于的**目标值（target）**。注意，区块头SHA256的结果，有256位，而Bits，只有32位（4字节）。Bits通过如下运算，得到target。以[区块277316](https://link.zhihu.com/?target=https%3A//blockchain.info/block/0000000000000001b6b9a13b095e96db41c4a928b97ef2d944a9b31b2cc7bdc4)为例
Bits值为0x1903a30c（十进制为419668748），这个值被存为系数/指数格式，前2位十六进制数字为幂，接下来得6位为系数。在这个区块里，0x19为幂（exponent ），而0x03a30c为系数（coefficient），计算公式为
target = coefficient * 256^(exponent – 3)
所以这个区块的target值为

```
target = 0x03a30c * 256^(0x19 - 3)
       = 238,348 * 256^22
       = 22,829,202,948,393,929,850,749,706,076,701,368,331,072,452,018,388,575,715,328
```

十六进制为0x0000000000000003a30c00000000000000000000000000000000000000000000
也就是说高度为277316的有效区块的头信息哈希值，要小于等于这个目标值。高度277136区块的Hash值实际为0x0000000000000001b6b9a13b095e96db41c4a928b97ef2d944a9b31b2cc7bdc4

# 难度

从中本聪的[创世区块](https://link.zhihu.com/?target=https%3A//blockchain.info/block/000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f)中可以看到，Bits为0x1d00ffff（十进制为486604799）时，难度（difficulty）为1（注意，**区块头中并没有存储difficulty的字段**）
根据Bits，算出target为0x00ffff * 256^26，即0x00000000ffff0000000000000000000000000000000000000000000000000000。也就是说，为了构建下一个合法区块，需要不断对块头做SHA256运算，直到找到一个区块头的Hash结果，前32位的值均为0（小于target）
SHA256运算的结果被认为是[一致的随机序列](https://link.zhihu.com/?target=https%3A//crypto.stackexchange.com/questions/12822/are-the-sha-family-hash-outputs-practically-random)，可以说，SHA256的结果中的某一位的值，为0或为1的概率相同。所以做一次计算，满足上述条件（前32位的值均为0）的概率为1 / (2^32)，也就是平均要做2^32次运算，才能找到这个值
为了方便理解，我们把**为了使区块头的SHA256结果小于某个目标值（target），平均要尝试的计算次数**，定义为**难度（difficulty）**，1 difficulty ≈ 2^32次 = 4294967296次 ≈ 4.2 * 10^9次 ≈ 4G次运算
以[区块501509](https://link.zhihu.com/?target=https%3A//blockchain.info/block/0000000000000000006c5532f4fd9ee03e07f94df165c556b89c495e97680147)为例，Bits值为0x18009645（十进制为402691653），target为0x009645 * 256 ^ 21。创世区块的target为0x00ffff * 256 ^ 26，难度为1，所以得到区块501509的难度为

```
(0x00ffff * 256 ^ 26）/ (0x009645 * 256 ^ 21)
= 65535/38469 * (256^5)
= 1.703579505575918 * 2^40
= 1873105475221.611
≈ 1.87 * 10^12
```

也就是1.87T（1T = 10^12）

综上，可以得出公式
difficulty_当前 = target_创世区块 / target_当前

其中，target_创世区块为定值0x00000000ffff0000000000000000000000000000000000000000000000000000，target_当前根据当前区块头中的Bits字段算出

同时，还能得出另一个结论：
出块时间(单位：秒) ≈ difficulty_当前 * 2^32 / 全网算力

当前比特币全网难度 1.93 T，全网算力 15.32 EH/s（1秒计算15.32E 次Hash），算出来的出块时间约为

```
1.93T * 2^32 / 15.32EH/s
= 1.93 * 10^12 * 4.295 * 10^9 / 15.32 * 10^18
= 8289.35 * 10^18 / 15.32 * 10^18
≈ 541 秒
```

总结一下

- 难度（difficulty）反映了矿工找到下一个有效区块的难易程度，难度随 区块头目标Hash值（target）的变动而变动，target值越小，难度越大
- 在难度（difficulty）不变的情况下，全网算力越大，出块时间理论上越快

# 难度调整

如前所述，**区块头目标Hash值（target）决定了难度（difficulty），进而影响比特币的出块时间**。根据设计，比特币要保证平均每10分钟的区块生成速度，这是比特币新币发行和交易完成的基础，需要在长期内保持相对稳定。
挖矿设备不断更新、淘汰，矿工不断加入、流失，导致全网算力保持实时变化，但整体来看在不断提高。为了保证比特币的平均出块时间稳定在十分钟，需要定期调整难度。
难度调整逻辑被写在代码中，在每个全节点中独立自动发生。每产生2016个区块，网络中的所有全节点都会调整难度。难度的调整公式是由产生最新2016个区块的花费时长与20160分钟（两周，即这些区块以10分钟一个的速率产生所期望花费的时长）比较得出的。难度是根据实际时长与期望时长的**比值**进行相应调整的（或变难或变易），简单来说，如果网络发现区块产生速率比10分钟要快时会增加难度。如果发现比10分钟慢时则降低难度。
这个**逻辑**可以简单表示为

```
Difficulty_新 = Difficulty_原 * ( 20160分钟 / 产生2016个区块的实际花费时长 )
```

从[代码](https://link.zhihu.com/?target=https%3A//github.com/bitcoin/bitcoin/blob/45173fa6fca9537abb0a0554f731d14b9f89c456/src/pow.cpp%23L49)能看到，为了防止难度变的太快，每个周期在调整的时候，如果需要调整的幅度超过4倍，也只会按4倍（难度调成原来的4倍或1/4）调整

```
unsigned int CalculateNextWorkRequired(const CBlockIndex* pindexLast, int64_t nFirstBlockTime, const Consensus::Params& params)
{
    if (params.fPowNoRetargeting)
        return pindexLast->nBits;

    // Limit adjustment step
    int64_t nActualTimespan = pindexLast->GetBlockTime() - nFirstBlockTime;
    if (nActualTimespan < params.nPowTargetTimespan/4)      // <==== 看这里
        nActualTimespan = params.nPowTargetTimespan/4;
    if (nActualTimespan > params.nPowTargetTimespan*4)      // <==== 看这里
        nActualTimespan = params.nPowTargetTimespan*4;

    // Retarget
    const arith_uint256 bnPowLimit = UintToArith256(params.powLimit);
    arith_uint256 bnNew;
    bnNew.SetCompact(pindexLast->nBits);
    bnNew *= nActualTimespan;
    bnNew /= params.nPowTargetTimespan;

    if (bnNew > bnPowLimit)
        bnNew = bnPowLimit;

    return bnNew.GetCompact();
}
```

**请再次注意，难度的调整，是通过调整区块头中的Bits字段来实现的，区块头中并没有直接存储全网难度（difficulty）的字段**

# 总结

到这里，相信你已经知道文章开头问题的答案。简单的说，比特币的全网难度（difficulty）

- 反映了矿工找到下一个有效区块的难易程度，难度随 区块头目标Hash值（target）的变化而变化，target值越小，难度越大
- 区块头目标Hash值（target）由Bits字段的值计算得出
- 定义 Bits值为0x1d00ffff时的难度（difficulty）为1
- 从创世区块起，每产生2016个区块，网络中的所有全节点都会自动调整难度，以保证比特币约十分钟的出块速度
