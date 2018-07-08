# 0x02Block

区块是组成区块的基本单位，我们可以通过bitcoin-cli命令查看一个区块的基本信息。

![bitcli-cli获取区块信息](https://upload-images.jianshu.io/upload_images/830585-b2f5228798c3ebf1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来我们就在源代码找一下区块的定义，由于我们并不知道区块定义在哪。我们试着全局搜一下block.h(或block.cpp)：

![搜索block.h](https://upload-images.jianshu.io/upload_images/830585-cbe9657e40c0cea9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

进去发现还真被我找到了，其实我们在上一篇的bitcoin源码结构的目录结构里已经说过./private目录下是区块类和交易类的实现。接下来，就让我们一窥block的究竟。

# 源码初窥
### CBlockHeader
```C++
/** Nodes collect new transactions into a block, hash them into a hash tree,
 * and scan through nonce values to make the block's hash satisfy proof-of-work
 * requirements.  When they solve the proof-of-work, they broadcast the block
 * to everyone and the block is added to the block chain.  The first transaction
 * in the block is a special one that creates a new coin owned by the creator
 * of the block.
 *
 **网络中的节点不断收集新的交易打包到区块中，所有的交易会通过两两哈希的方式形成一个Merkle树
 * 打包的过程就是要完成工作量证明的要求，当节点解出了当前的随机数时，
 * 它就把当前的区块广播到其他所有节点，并且加到区块链上。
 * 区块中的第一笔交易称之为CoinBase交易，是产生的新币，奖励给区块的产生者  
 * 
 * add by chaors 20180419
 */

class CBlockHeader
{
public:
    // header
    int32_t nVersion;       //版本
    uint256 hashPrevBlock;  //上一个区块的hash
    uint256 hashMerkleRoot; //包含交易信息的Merkle树根
    uint32_t nTime;         //时间戳
    uint32_t nBits;         //工作量证明(POW)的难度
    uint32_t nNonce;        //要找的符合POW的随机数

    CBlockHeader()          //构造函数初始化成员变量
    {
        SetNull();          
    }

    ADD_SERIALIZE_METHODS;  //通过封装的模板实现类的序列化

    template <typename Stream, typename Operation>
    inline void SerializationOp(Stream& s, Operation ser_action) {
        READWRITE(this->nVersion);
        READWRITE(hashPrevBlock);
        READWRITE(hashMerkleRoot);
        READWRITE(nTime);
        READWRITE(nBits);
        READWRITE(nNonce);
    }

    void SetNull()          //初始化成员变量
    {
        nVersion = 0;
        hashPrevBlock.SetNull();
        hashMerkleRoot.SetNull();
        nTime = 0;
        nBits = 0;
        nNonce = 0;
    }

    bool IsNull() const
    {
        return (nBits == 0);     //难度为0说明区块还未创建，区块头为空
    }

    uint256 GetHash() const;     //获取哈希

    int64_t GetBlockTime() const //获取区块时间
    {
        return (int64_t)nTime;
    }
};
```

### CBlock 
```C++
class CBlock : public CBlockHeader         //继承自CBlockHeader，拥有其所有成员变量
{
public:
    // network and disk
    std::vector<CTransactionRef> vtx;      //所有交易的容器

    // memory only
    mutable bool fChecked;                 //交易是否验证

    CBlock()
    {
        SetNull();
    }

    CBlock(const CBlockHeader &header)
    {
        SetNull();
        *((CBlockHeader*)this) = header;
    }

    ADD_SERIALIZE_METHODS;

    template <typename Stream, typename Operation>
    inline void SerializationOp(Stream& s, Operation ser_action) {
        READWRITE(*(CBlockHeader*)this);
        READWRITE(vtx);
    }

    void SetNull()
    {
        CBlockHeader::SetNull();
        vtx.clear();
        fChecked = false;
    }

    CBlockHeader GetBlockHeader() const
    {
        CBlockHeader block;
        block.nVersion       = nVersion;
        block.hashPrevBlock  = hashPrevBlock;
        block.hashMerkleRoot = hashMerkleRoot;
        block.nTime          = nTime;
        block.nBits          = nBits;
        block.nNonce         = nNonce;
        return block;
    }

    std::string ToString() const;
};
```

### CBlockLocator
```C++

/** Describes a place in the block chain to another node such that if the
 * other node doesn't have the same branch, it can find a recent common trunk.
 * The further back it is, the further before the fork it may be.
 *
 **描述区块链中在其他节点的一个位置，
 *如果其他节点没有相同的分支，它可以找到一个最近的中继(最近的相同块)。
 *更进一步地讲，它可能是分叉前的一个位置
 */
struct CBlockLocator
{
    std::vector<uint256> vHave;

    CBlockLocator() {}

    explicit CBlockLocator(const std::vector<uint256>& vHaveIn) : vHave(vHaveIn) {}

    ADD_SERIALIZE_METHODS;

    template <typename Stream, typename Operation>
    inline void SerializationOp(Stream& s, Operation ser_action) {
        int nVersion = s.GetVersion();
        if (!(s.GetType() & SER_GETHASH))
            READWRITE(nVersion);
        READWRITE(vHave);
    }

    void SetNull()
    {
        vHave.clear();
    }

    bool IsNull() const
    {
        return vHave.empty();
    }
};
```

###. cpp函数
```C++
uint256 CBlockHeader::GetHash() const
{
    return SerializeHash(*this);        //生成256位的哈希值
}

std::string CBlock::ToString() const    //区块对象格式化字符串输出
{
    std::stringstream s;
    s << strprintf("CBlock(hash=%s, ver=0x%08x, hashPrevBlock=%s, hashMerkleRoot=%s, nTime=%u, nBits=%08x, nNonce=%u, vtx=%u)\n",
        GetHash().ToString(),
        nVersion,
        hashPrevBlock.ToString(),
        hashMerkleRoot.ToString(),
        nTime, nBits, nNonce,
        vtx.size());
    for (const auto& tx : vtx) {
        s << "  " << tx->ToString() << "\n";
    }
    return s.str();
}
```

# 区块结构分析

区块是通过前后链接构成区块链的一种容器数据结构。它由描述区块主要信息的区块头和包含若干交易数据的区块共同组成。区块头是80字节，而平均每个交易至少是250字节，而且平均每个区块至少包含超过500个交易。所以，一个区块是比区块头大好多的数据体。这也是比特币验证交易是否存在采用Merkcle树的原因。

### 整体结构

数据项 | 大小(Byte) | 描述
:-: | :-: | :-:
Block Size | 4 | 区块大小
Block Header | 80 | 区块头信息大小
Transactions | m*n(n>=250)| 所有交易的列表
Transactions Counter | 1-9 | 交易数额

比特币的区块大小目前被严格限制在1MB以内。4字节的区块大小字段不包含在此内!

### 区块头
数据项 | 大小(Byte) | 存储方式 | 描述
:-: | :-: | :-: | :-:
Version | 4 | 小端 | [区块版本](https://bitcoin.org/en/developer-reference#block-versions)，规定了区块遵守的验证规则
Previous Block Hash | 32 | 内部字节顺序 | 上一个区块哈希值(SHA256 (SHA256(Block Header)))
Merkle Root | 32 | 内部字节顺序 | [Merkle](https://bitcoin.org/en/developer-reference#merkle-trees)树根，包含了所有交易的哈希
Timestamp | 4 | 小端 | 区块产生时间戳，大于前11个区块时间戳的平均值，全节点会拒绝时间戳超出自己2小时的区块
nBitS | 4 | 小端 | 工作量证明(POW)的目标难度值，当前区块难度值需要经过[Target nBits编码](https://bitcoin.org/en/developer-reference#merkle-trees)才能转化为目标哈希值
Nonce | 4 | 小端 | 用于POW的一个随机数，随着算力增大可能会导致Nonce位数不够 协议规定时间戳和CoinbaseTransaction信息可修改用于扩展Nonce位数

### 区块标识符

 - __BlockHash__ 区块哈希值，是通过SHA256算法对区块头信息进行哈希得到的，这个值必须满足POW的DifficultyTarget，该区块才被认为有效。同时，也是区块的唯一标识符，可以通过通过bitcoin-cli根据BlockHash查询区块信息(文章开头我们就使用过)

- __BlockHeight__ 区块高度，是用来标示区块在区块链中的位置。创世区块高度为0，每一个加在后面的区块，区块高度递增1。可以通过bitcoin-cli根据高度查询区块哈希值(文章开头我们就使用过)

##### 注：BlockHeight并不是唯一标识符，当区块链出现临时分叉时，会有两个区块的高度值相同的情况。

### 创世区块

区块链上第一个区块被称为创世区块，它是所有区块的共同祖先。我们可以查看下比特币的创世区块：

![创世区块](https://upload-images.jianshu.io/upload_images/830585-c55b9849ed27f2cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

比特币创始人聪哥在创世区块包含了一个隐藏的信息。在其Coinbase交易的输入中包含这样一句话“The Times 03/Jan/2009 Chancellor on brink of second bailout forbanks.”这句话是泰晤士报当天的头版文章标题，聪哥这样做的目的不得而知。但是，这样一条非交易信息可以轻而易举地插入比特币，这个现象值得深思。如此，就不难理解前不久曝光的"不法分子利用比特币存储儿童色情内容"新闻，当然这种存储可能远比聪哥的那句话要更复杂一点。

# 思考

- 1. 我们查看的区块信息中，以下不在源码结构中的字段有什么含义？为什么他们不在源码定义的区块结构中？
      confirmations
      strippedsize
      weight
      mediantime
      bits
      chainwork

- 2.为什么区块的哈希没有定义在区块头内部？

##### 注：以上问题可能我也没答案，可以留言我们一起交流共同进步。











