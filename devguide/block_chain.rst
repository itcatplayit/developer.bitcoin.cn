Block Chain
===========

区块链为比特币提供了一个公共账本，按照顺序和时间戳记录交易。该系统可以防止双花和更改历史交易。

Introduction
------------


比特币 `网络 <../devguide/p2p_network.html>`__ 中的每个全节点独立存储的区块链只包含该节点已经验证过的区块。当几个节点在它们的区块链中都包含了同一个区块，此时认为它们 :term:`达成一致 <Consensus>` 。节点之间为了保证一致性所遵循的验证规则别称为 :term:`一致性原则 <Consensus rules>` 。本节将讨论比特币核心使用的多种一致性原则。

.. figure:: /img/dev/en-blockchain-overview.svg
   :alt: 区块链纵览

   区块链纵览

上图示意了一个简单版本的区块链。每个 :term:`区块 <Block>` 中包含的一个或多个交易放置在区块的数据区。每笔交易副本被哈希，然后哈希值被配对，被哈希，再次被配对，再次被哈希直到只剩下最后一个哈希，该哈希称为默克尔树(Merkle tree)的 :term:`默克尔根(Merkle root) <Merkle root>` 。

默克尔根保存在区块的头部。每个区块头部还包含前一个区块的头hash，这样就将区块链接起来（前向链表）。这样可以保证交易在不修改后续节点的情况下无法修改当前节点的内容。

交易也是被链接在一起。比特币钱包给人的印象是1 satoshis（比特币最小单位，1 Satoshi等于0.00000001比特币）比特币从一个钱包到另一个钱包，但是实际上比特币是从一个交易到另一个交易。每个交易花费之前接收到的一个或多个交易的币，因此一个交易的输入是之前另一个交易的输出。

.. figure:: /img/dev/en-transaction-propagation.svg
   :alt: 交易传播图

   交易传播图

在将交易结果分送多个地址的情形下，一个交易可以产生多个输出，但是一个指定的交易输出只能在整条链中作为输入一次。后续任何的引用都会由于禁止双花导致失败。

输出和 :term:`交易标志符 (TXIDs) <Txid>` 绑定，它是交易的签名。

由于每个输出只能被花费一次，因此区块链中所有的输出可以分为两类： :term:`未花费的输出 (UTXOs) <UTXO>` 和已花费的输出。一个有效的交易，必须使用UTXOs作为输入。

除了基础币交易（后续详述）外，如果交易的输出值大于交易的输入值，这个交易会被拒绝；但是，如果交易的输入值大于交易的输出值，二者的差值则作为 :term:`交易费用 <Transaction fee>` 被挖出包含该交易区块的 :term:`矿工 <Mining>` 占有。举例来说，上图显示的每个交易的输出都比输入少10,000 satoshis，缺少的部分就是作为交易费用的部分。

Proof Of Work
-------------

区块链在 `网络 <../devguide/p2p_network.html>`__ 上被匿名节点协作保存，因此比特币要求每个区块都投入一定工作量才能生成，以此来确保不可靠节点想要修改历史块就不得不比那些只想往区块链上添加新块的诚实节点付出更多的努力。

链在一起的区块可以保证，如果不更新指定区块的所有后续区块，则无法更新该区块中的交易。因此，更新一个特定区块的代价随着新区块的增加而增加，同时放大了工作量证明的作用。

:term:`工作量证明 <Proof of work>` 利用了密码学哈希结果明显的随机特性。一个好的哈希算法将任意的数据转换成看起来随机的数字。如果输入数据的发生任何的变动，哈希过程重新执行，一个新的随机数就会产生，因此无法通过更新输入数据有目的的预测哈希结果。

为了证明节点为了生成区块做了一定的工作，节点必须产生出一个哈希结果不超出指定值的区块头部。比如，如果最大的可能的哈希值的个数为2256-1，可以证明节点平均尝试2次就可以产生一个哈希值小于2255的头部。

在上面的例子中，节点几乎每个一次就可以成功一次。同时可以根据指定的 :term:`目标 <nBits>` 门槛，估计一次哈希尝试成功的概率。比特币假定门槛和成功概率是线性关系，门槛越低，平均需要的尝试次数越多。

只有当区块的哈希值满足一致性协议指定的 :term:`难度 <Difficulty>` 值时，该区块才会被加入到区块链中。每隔2,016个区块， `网络 <../devguide/p2p_network.html>`__ 利用存储在每个区块头中的时间戳计算产生这2,016各区块的时间间隔。该间隔的理想值为1,209,600秒(两周)。

- 如果生成2,016个区块花费的实际的间隔小于2周，则期望的难度值会按比例上升（最高300%），由此下一批2,016个区块按相同速率产生，则刚好花费2周的时间。

- 同理，如果实际的间隔大于2周，期望的难度值则会按比例下降（最低到75%）。

（注意：比特币核心实现中的一个一偏移错误导致使用仅2,01\ *5* 个块的时间戳每2,01\ *6* 个块更新一次，从而产生轻微的偏差。）

由于每个哈希头都要满足指定的难度值，而且每个区块都会链接它前面的区块，因此更新一个区块（平均来讲）需要付出从该区块创造到当前时刻区块链整体 `网络 <../devguide/p2p_network.html>`__ 算力的总和。因此只有你获得了了 `网络 <../devguide/p2p_network.html>`__ 的大部分算力，才能够可靠的进行51%攻击修改交易历史（但是，需要指出的是，即使少于50%的算力，仍然有很大可能性进行这种攻击）。

区块头部中提供了几个容易更新的字段，比如专门的nonce字段，因此获取新的哈希值并不一定要等待新的交易。同时，只需要对80字节的区块头进行哈希，因此在区块中包含大量的交易不会降低哈希的效率，增加新的交易只需要重算默克尔树。

Block Height And Forking
------------------------

所有成功挖到新块的矿工都可以把他们的新块添加到区块链中（假定这些区块都是有效的）。这些区块通过它们的 :term:`区块高度 <Block height>` ————当前区块到初始区块（区块0，或者说更有名的称为 :term:`创世块 <Genesis block>`）的区块个数 进行定位。例如，2016是第一个进行难度调整的区块。

.. figure:: /img/dev/en-blockchain-fork.svg
   :alt: 通常区块分叉与罕见区块分叉

   通常区块分叉与罕见区块分叉

由于多个矿工可能几乎同时挖到新区块，因此可能存在多个区块拥有相同区块高度。这种情况下就在区块链中产生了明显的 :term:`分叉 <Fork>` ，如上图所示。

当几个矿工同时生产出区块，每个节点独立的判断选择接受哪个，在没有其他考虑的情况下，节点通常选择接受他们看到的第一个区块。

最终，一个矿工生产出来了一个区块，它附在了几条并行区块分叉中的一条。这时这条区块就比其他区块更有优势。假设一个分叉只包含有效的区块，正常的节点通常会跟随难度最大的区块继续工作，抛弃其他分叉上的 :term:`失效块 <Stale block>`。（失效块也有时叫孤立块或孤儿区块，但这些术语也用于没有已知父块的真正孤立块。）

如果不同的矿工出于相反目的工作，例如一些矿工努力扩展区块链，而其他矿工则试图通过51%的攻击来修改交易历史，那么长期分叉是可能的。

由于可能存在多个分叉，因此区块高度不能作为区块的唯一标识。而是使用头部的哈希值（通常进行字节顺序反转，并用16进制表示）。

Transaction Data
----------------

Every block must include one or more transactions. The first one of these transactions must be a coinbase transaction, also called a generation transaction, which should collect and spend the block reward (comprised of a block subsidy and any transaction fees paid by transactions included in this block).

The UTXO of a coinbase transaction has the special condition that it cannot be spent (used as an input) for at least 100 blocks. This temporarily prevents a miner from spending the transaction fees and block reward from a block that may later be determined to be stale (and therefore the coinbase transaction destroyed) after a block chain fork.

Blocks are not required to include any non-coinbase transactions, but miners almost always do include additional transactions in order to collect their transaction fees.

All transactions, including the coinbase transaction, are encoded into blocks in binary raw transaction format.

The raw transaction format is hashed to create the transaction identifier (txid). From these txids, the :term:`merkle tree <Merkle tree>` is constructed by pairing each txid with one other txid and then hashing them together. If there are an odd number of txids, the txid without a partner is hashed with a copy of itself.

The resulting hashes themselves are each paired with one other hash and hashed together. Any hash without a partner is hashed with itself. The process repeats until only one hash remains, the merkle root.

For example, if transactions were merely joined (not hashed), a five-transaction merkle tree would look like the following text diagram:

::

          ABCDEEEE .......Merkle root
         /        \
      ABCD        EEEE
     /    \      /
    AB    CD    EE .......E is paired with itself
   /  \  /  \  /
   A  B  C  D  E .........Transactions

As discussed in the Simplified Payment Verification (SPV) subsection, the merkle tree allows clients to verify for themselves that a transaction was included in a block by obtaining the merkle root from a block header and a list of the intermediate hashes from a full peer. The full peer does not need to be trusted: it is expensive to fake block headers and the intermediate hashes cannot be faked or the verification will fail.

For example, to verify transaction D was added to the block, an SPV client only needs a copy of the C, AB, and EEEE hashes in addition to the merkle root; the client doesn’t need to know anything about any of the other transactions. If the five transactions in this block were all at the maximum size, downloading the entire block would require over 500,000 bytes—but downloading three hashes plus the block header requires only 140 bytes.

Note: If identical txids are found within the same block, there is a possibility that the merkle tree may collide with a block with some or all duplicates removed due to how unbalanced merkle trees are implemented (duplicating the lone hash). Since it is impractical to have separate transactions with identical txids, this does not impose a burden on honest software, but must be checked if the invalid status of a block is to be cached; otherwise, a valid block with the duplicates eliminated could have the same merkle root and block hash, but be rejected by the cached invalid outcome, resulting in security bugs such as `CVE-2012-2459 <https://en.bitcoin.it/wiki/CVEs#CVE-2012-2459>`__.

Consensus Rule Changes
----------------------

To maintain consensus, all full nodes validate blocks using the same consensus rules. However, sometimes the consensus rules are changed to introduce new features or prevent `network <../devguide/p2p_network.html>`__ abuse. When the new rules are implemented, there will likely be a period of time when non-upgraded nodes follow the old rules and upgraded nodes follow the new rules, creating two possible ways consensus can break:

1. A block following the new consensus rules is accepted by upgraded nodes but rejected by non-upgraded nodes. For example, a new transaction feature is used within a block: upgraded nodes understand the feature and accept it, but non-upgraded nodes reject it because it violates the old rules.

2. A block violating the new consensus rules is rejected by upgraded nodes but accepted by non-upgraded nodes. For example, an abusive transaction feature is used within a block: upgraded nodes reject it because it violates the new rules, but non-upgraded nodes accept it because it follows the old rules.

In the first case, rejection by non-upgraded nodes, mining software which gets block chain data from those non-upgraded nodes refuses to build on the same chain as mining software getting data from upgraded nodes. This creates permanently divergent chains—one for non-upgraded nodes and one for upgraded nodes—called a :term:`hard fork <Hard fork>`.

.. figure:: /img/dev/en-hard-fork.svg
   :alt: Hard Fork

   Hard Fork

In the second case, rejection by upgraded nodes, it’s possible to keep the block chain from permanently diverging if upgraded nodes control a majority of the hash rate. That’s because, in this case, non-upgraded nodes will accept as valid all the same blocks as upgraded nodes, so the upgraded nodes can build a stronger chain that the non-upgraded nodes will accept as the best valid block chain. This is called a :term:`soft fork <Soft fork>`.

.. figure:: /img/dev/en-soft-fork.svg
   :alt: Soft Fork

   Soft Fork

Although a fork is an actual divergence in block chains, changes to the consensus rules are often described by their potential to create either a hard or soft fork. For example, “increasing the block size above 1 MB requires a hard fork.” In this example, an actual block chain fork is not required—but it is a possible outcome.

Consensus rule changes may be activated in various ways. During Bitcoin’s first two years, Satoshi Nakamoto performed several soft forks by just releasing the backwards-compatible change in a client that began immediately enforcing the new rule. Multiple soft forks such as `BIP30 <https://github.com/bitcoin/bips/blob/master/bip-0030.mediawiki>`__ have been activated via a flag day where the new rule began to be enforced at a preset time or block height. Such forks activated via a flag day are known as :term:`User Activated Soft Forks <UASF>` (UASF) as they are dependent on having sufficient users (nodes) to enforce the new rules after the flag day.

Later soft forks waited for a majority of hash rate (typically 75% or 95%) to signal their readiness for enforcing the new consensus rules. Once the signalling threshold has been passed, all nodes will begin enforcing the new rules. Such forks are known as :term:`Miner Activated Soft Forks <MASF>` (MASF) as they are dependent on miners for activation.

**Resources:** `BIP16 <https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki>`__, `BIP30 <https://github.com/bitcoin/bips/blob/master/bip-0030.mediawiki>`__, and `BIP34 <https://github.com/bitcoin/bips/blob/master/bip-0034.mediawiki>`__ were implemented as changes which might have lead to soft forks. `BIP50 <https://github.com/bitcoin/bips/blob/master/bip-0050.mediawiki>`__ describes both an accidental hard fork, resolved by temporary downgrading the capabilities of upgraded nodes, and an intentional hard fork when the temporary downgrade was removed. A document from Gavin Andresen outlines `how future rule changes may be implemented <https://gist.github.com/gavinandresen/2355445>`__.

Detecting Forks
---------------

Non-upgraded nodes may use and distribute incorrect information during both types of forks, creating several situations which could lead to financial loss. In particular, non-upgraded nodes may relay and accept transactions that are considered invalid by upgraded nodes and so will never become part of the universally-recognized best block chain. Non-upgraded nodes may also refuse to relay blocks or transactions which have already been added to the best block chain, or soon will be, and so provide incomplete information.

Bitcoin Core includes code that detects a hard fork by looking at block chain proof of work. If a non-upgraded node receives block chain headers demonstrating at least six blocks more proof of work than the best chain it considers valid, the node reports a warning in the `“getnetworkinfo” RPC <../reference/rpc/getnetworkinfo.html>`__ results and runs the ``-alertnotify`` command if set. This warns the operator that the non-upgraded node can’t switch to what is likely the best block chain.

Full nodes can also check block and transaction version numbers. If the block or transaction version numbers seen in several recent blocks are higher than the version numbers the node uses, it can assume it doesn’t use the current consensus rules. Bitcoin Core reports this situation through the `“getnetworkinfo” RPC <../reference/rpc/getnetworkinfo.html>`__ and ``-alertnotify`` command if set.

In either case, block and transaction data should not be relied upon if it comes from a node that apparently isn’t using the current consensus rules.

SPV clients which connect to full nodes can detect a likely hard fork by connecting to several full nodes and ensuring that they’re all on the same chain with the same block height, plus or minus several blocks to account for transmission delays and stale blocks. If there’s a divergence, the client can disconnect from nodes with weaker chains.

SPV clients should also monitor for block and :ref:`transaction version number <term-transaction-version-number>` increases to ensure they process received transactions and create new transactions using the current consensus rules.
