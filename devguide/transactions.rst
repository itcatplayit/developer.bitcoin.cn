交易
============

交易使得用户消费聪。每个交易有多个部分组成，既包括简单直接的付款，也包括复杂交易。

介绍
------------

这一部分将会讨论交易的每个部分，并演示如何将它们组合成一个完整的交易。

为了简单起见，本节假设生成交易不存在。生成交易只能被比特币矿工创建，并且它们对下面的规则存在许多例外情况。这里建议读者在区块链一章了解生成交易的一些细节，本章将不在交易规则后面单独注明对生成交易的例外情况。

.. figure:: /img/dev/en-tx-overview.svg
   :alt: 一个交易的部分图

   一个交易的部分图

上图展示了比特币交易的主要部分。每笔交易至少有一个输入和一个输出。每个 :term:`输入 <Input>` 会花费上个输出产生的比特币。每个 :term:`输出 <Output>` 都作为交易未花输出（UTXO）直到被后面作为输入花费掉。当你的比特币钱包告诉你，你还有10000聪的比特币，它实际上说的是你拥有10000聪等着一个或多个UTXO。

每个交易的前缀为4字节的 :ref:`交易版本号 <term-transaction-version-number>`，告诉比特币用户和矿工，采取什么样的策略验证它。这种设计可以让开发者在不影响旧区块的前提下为未来交易创建新的一致性规则。

.. figure:: /img/dev/en-tx-overview-spending.svg
   :alt: 花费输出

   花费输出

一个输出有一个基于其在交易中位置的隐含索引号码，第一个输出的索引是0。同时，输出还有一定数量的比特币，这些比特币会支付给可以满足公钥脚本指定条件的人。任何满足该公钥脚本条件的人可以消费这些比特币。

一个输入使用交易标识（txid）和 :ref:`输出索引 <term-output-index>` 号（常被称为“vout”，代指输出向量output vector）来区分一个将要使用的输出。同时，它还有一个签名脚本，可以用来提供满足输出公钥脚本的条件的参数。（序列号和锁定时间是相关的，将在后续章节介绍）

下面的图片通过展示Alice给Bob一些币然后Bob把币花掉的两笔交易过程，反映出这些特性具体是如何被使用的。Alice和Bob都将使用最常见的Pay-To-Public-Key-Hash (P2PKH)的交易方式。:term:`P2PKH <P2PKH address>` 使Alice将比特币支付给一个地址，然后Bob再使用简单的加密 :ref:`钥对 <term-key-pair>` 继续将这些比特币花掉。

.. figure:: /img/dev/en-creating-p2pkh-output.svg
   :alt: 创建P2PKH公钥哈希以接收款项

   创建P2PKH公钥哈希以接收款项

Bob在Alice新建第一交易前，首先要生成一个私有/公共的 :ref:`键值对 <term-key-pair>`。比特币使用椭圆曲线数字签名算法（ `ECDSA <https://en.wikipedia.org/wiki/Elliptic_Curve_DSA>`__ )带有 `secp256k1<http://www.secg.org/sec2-v2.pdf>`__ 曲线； `secp256k1 <http://www.secg.org/sec2-v2.pdf>`__ :term:`键值对 <Private key>`是一个256bit的随机数。这个数据的拷贝被确定性地转换为一个 `secp256k1 <http://www.secg.org/sec2-v2.pdf>`__ :term:`公钥 <Public key>`。由于这个转换过程是确定可重复的，因此不需要在本地保存公钥。

然后对公钥（pubkey）进行加密哈希。这个公钥哈希稍后也可以可靠地重复，因此也不需要存储。哈希缩短并混淆了公钥，使手动抄写变得更容易，并提供了针对意外问题的安全性，这些问题可能允许在以后的某个时候从公钥数据重建私钥。

Bob将公钥哈希提供给Alice。公钥哈希几乎总是以比特币 :term:`addresses <Address>` 的形式发送，比特币地址是基于58的编码字符串，包含地址版本号、哈希和错误检测校验和，以捕捉拼写错误。地址可以通过任何介质传输，包括阻止发送者与接收者通信的单向介质，还可以进一步编码为另一种格式，例如包含 :ref:`比特币:” URI <term-bitcoin-uri>` 的二维码。

一旦Alice有了地址并将其解码回标准哈希，她就可以创建第一个交易。她创建了一个标准的P2PKH交易输出，其中包含指令，如果任何人能够证明他们控制了与Bob的哈希公钥相对应的私钥，则允许他们使用该输出。这些指令被称为 :term:`公钥脚本（pubkey script或scriptPubKey） <Pubkey script>` 。

Alice广播交易并将其添加到块链中。 `网络 <../devguide/p2p_network.html>`__将其归类为未消费交易输出（UTXO），Bob的钱包软件将其显示为可消费余额。

一段时间后，当Bob决定使用UTXO时，他必须创建一个引用由其哈希创建的交易Alice的输入，称为交易标识符（txid），以及她使用的特定输出的索引号（:ref:`输出索引 <term-output-index>`）。然后，他必须创建一个签名脚本——一组满足Alice在前一个输出的公钥脚本中设置的条件的数据参数。签名脚本也称为scriptSigs。

公钥脚本和签名脚本将 `secp256k1 <http://www.secg.org/sec2-v2.pdf>`__ 公钥和签名与条件逻辑相结合，创建了一个可编程的授权机制。

.. figure:: /img/dev/en-unlocking-p2pkh-output.svg
   :alt: 解锁P2PKH产出用于支出

   解锁P2PKH产出用于支出

对于P2PKH样式的输出，Bob的签名脚本将包含以下两段数据：

1.他的完整（未哈希）公钥，因此公钥脚本可以检查其哈希值是否与Alice提供的公钥哈希值相同。

2.通过使用 `ECDSA <https://en.wikipedia.org/wiki/Elliptic_Curve_DSA>`__ 密码公式将某些交易数据（如下所述）与Bob的私钥组合而成的 `secp256k1 <http://www.secg.org/sec2-v2.pdf>`__ :term:`签名 <Signature>`。这允许公钥脚本验证Bob是否拥有创建公钥的私钥。

Bob的 `secp256k1 <http://www.secg.org/sec2-v2.pdf>`__ 签名不仅证明Bob控制了他的私钥；它还使交易中的非签名脚本部分不可篡改，这样Bob就可以通过 `对等网络 <../devguide/p2p_network.html>`__ 安全地进行广播。

.. figure:: /img/dev/en-signing-output-to-spend.svg
   :alt: 支出产出时一些事项签名

   支出产出时一些事项签名

如上图所示，Bob签署的数据包括上一笔交易的txid和 :ref:`输出索引 <term-output-index>`、上一笔输出的公钥脚本、Bob创建的将让下一个接收者花费这笔交易输出的公钥脚本，以及花费给下一个收件人的比特币金额。本质上，除了任何签名脚本之外，整个交易都是签名的，这些脚本包含完整的公钥和 `secp256k1 <http://www.secg.org/sec2-v2.pdf>`__ 签名。

在签名脚本中放入签名和公钥后，Bob通过  `对等网络 <../devguide/p2p_network.html>`__ 向比特币矿工广播交易。在进一步广播事务或尝试将其包括在新的事务块中之前，每个对等方和矿工独立验证事务。

P2PKH Script Validation
-----------------------

The validation procedure requires evaluation of the signature script and pubkey script. In a P2PKH output, the pubkey script is:

::

   OP_DUP OP_HASH160 <PubkeyHash> OP_EQUALVERIFY OP_CHECKSIG

The spender’s signature script is evaluated and prefixed to the beginning of the script. In a P2PKH transaction, the signature script contains an `secp256k1 <http://www.secg.org/sec2-v2.pdf>`__ signature (sig) and full public key (pubkey), creating the following concatenation:

::

   <Sig> <PubKey> OP_DUP OP_HASH160 <PubkeyHash> OP_EQUALVERIFY OP_CHECKSIG

The script language is a `Forth-like <https://en.wikipedia.org/wiki/Forth_%28programming_language%29>`__ stack-based language deliberately designed to be stateless and not Turing complete. Statelessness ensures that once a transaction is added to the block chain, there is no condition which renders it permanently unspendable. Turing-incompleteness (specifically, a lack of loops or gotos) makes the script language less flexible and more predictable, greatly simplifying the security model.

To test whether the transaction is valid, signature script and pubkey script operations are executed one item at a time, starting with Bob’s signature script and continuing to the end of Alice’s pubkey script. The figure below shows the evaluation of a standard P2PKH pubkey script; below the figure is a description of the process.

.. figure:: /img/dev/en-p2pkh-stack.svg
   :alt: P2PKH Stack Evaluation

   P2PKH Stack Evaluation

-  The signature (from Bob’s signature script) is added (pushed) to an empty stack. Because it’s just data, nothing is done except adding it to the stack. The public key (also from the signature script) is pushed on top of the signature.

-  From Alice’s pubkey script, the :ref:`“OP_DUP” <term-op-dup>` operation is executed. :ref:`“OP_DUP” <term-op-dup>` pushes onto the stack a copy of the data currently at the top of it—in this case creating a copy of the public key Bob provided.

-  The operation executed next, :ref:`“OP_HASH160” <term-op-hash160>`, pushes onto the stack a hash of the data currently on top of it—in this case, Bob’s public key. This creates a hash of Bob’s public key.

-  Alice’s pubkey script then pushes the pubkey hash that Bob gave her for the first transaction. At this point, there should be two copies of Bob’s pubkey hash at the top of the stack.

-  Now it gets interesting: Alice’s pubkey script executes :ref:`“OP_EQUALVERIFY” <term-op-equalverify>`. :ref:`“OP_EQUALVERIFY” <term-op-equalverify>` is equivalent to executing :ref:`“OP_EQUAL” <term-op-equal>` followed by :ref:`“OP_VERIFY” <term-op-verify>` (not shown).

   :ref:`“OP_EQUAL” <term-op-equal>` (not shown) checks the two values at the top of the stack; in this case, it checks whether the pubkey hash generated from the full public key Bob provided equals the pubkey hash Alice provided when she created transaction #1. :ref:`“OP_EQUAL” <term-op-equal>` pops (removes from the top of the stack) the two values it compared, and replaces them with the result of that comparison: zero (*false*) or one (*true*).

   :ref:`“OP_VERIFY” <term-op-verify>` (not shown) checks the value at the top of the stack. If the value is *false* it immediately terminates evaluation and the transaction validation fails. Otherwise it pops the *true* value off the stack.

-  Finally, Alice’s pubkey script executes :ref:`“OP_CHECKSIG” <term-op-checksig>`, which checks the signature Bob provided against the now-authenticated public key he also provided. If the signature matches the public key and was generated using all of the data required to be signed, :ref:`“OP_CHECKSIG” <term-op-checksig>` pushes the value *true* onto the top of the stack.

If *false* is not at the top of the stack after the pubkey script has been evaluated, the transaction is valid (provided there are no other problems with it).

P2SH Scripts
------------

Pubkey scripts are created by spenders who have little interest what that script does. Receivers do care about the script conditions and, if they want, they can ask spenders to use a particular pubkey script. Unfortunately, custom pubkey scripts are less convenient than short Bitcoin addresses and there was no standard way to communicate them between programs prior to widespread implementation of the now deprecated `BIP70 <https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki>`__ Payment Protocol discussed later.

To solve these problems, pay-to-script-hash (:term:`P2SH <P2SH address>`) transactions were created in 2012 to let a spender create a pubkey script containing a hash of a second script, the :term:`redeem script <Redeem script>`.

The basic P2SH workflow, illustrated below, looks almost identical to the P2PKH workflow. Bob creates a redeem script with whatever script he wants, hashes the redeem script, and provides the redeem script hash to Alice. Alice creates a P2SH-style output containing Bob’s redeem script hash.

.. figure:: /img/dev/en-creating-p2sh-output.svg
   :alt: Creating A P2SH Redeem Script And Hash

   Creating A P2SH Redeem Script And Hash

When Bob wants to spend the output, he provides his signature along with the full (serialized) redeem script in the signature script. The `peer-to-peer network <../devguide/p2p_network.html>`__ ensures the full redeem script hashes to the same value as the script hash Alice put in her output; it then processes the redeem script exactly as it would if it were the primary pubkey script, letting Bob spend the output if the redeem script does not return false.

.. figure:: /img/dev/en-unlocking-p2sh-output.svg
   :alt: Unlocking A P2SH Output For Spending

   Unlocking A P2SH Output For Spending

The hash of the redeem script has the same properties as a pubkey hash—so it can be transformed into the standard Bitcoin address format with only one small change to differentiate it from a standard address. This makes collecting a P2SH-style address as simple as collecting a P2PKH-style address. The hash also obfuscates any public keys in the redeem script, so P2SH scripts are as secure as P2PKH pubkey hashes.

Standard Transactions
---------------------

After the discovery of several dangerous bugs in early versions of Bitcoin, a test was added which only accepted transactions from the `network <../devguide/p2p_network.html>`__ if their pubkey scripts and signature scripts matched a small set of believed-to-be-safe templates, and if the rest of the transaction didn’t violate another small set of rules enforcing good `network <../devguide/p2p_network.html>`__ behavior. This is the ``IsStandard()`` test, and transactions which pass it are called standard transactions.

Non-standard transactions—those that fail the test—may be accepted by nodes not using the default Bitcoin Core settings. If they are included in blocks, they will also avoid the IsStandard test and be processed.

Besides making it more difficult for someone to attack Bitcoin for free by broadcasting harmful transactions, the standard transaction test also helps prevent users from creating transactions today that would make adding new transaction features in the future more difficult. For example, as described above, each transaction includes a version number—if users started arbitrarily changing the version number, it would become useless as a tool for introducing backwards-incompatible features.

As of Bitcoin Core 0.9, the standard pubkey script types are:

-  Pay To Public Key Hash (P2PKH)
-  Pay To Script Hash (P2SH)
-  Multisig
-  Pubkey
-  Null Data

Pay To Public Key Hash (P2PKH)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

P2PKH is the most common form of pubkey script used to send a transaction to one or multiple Bitcoin addresses.

::

   Pubkey script: OP_DUP OP_HASH160 <PubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
   Signature script: <sig> <pubkey>

Pay To Script Hash (P2SH)
~~~~~~~~~~~~~~~~~~~~~~~~~

P2SH is used to send a transaction to a script hash. Each of the standard pubkey scripts can be used as a P2SH redeem script, excluding P2SH itself. As of Bitcoin Core 0.9.2, P2SH transactions can contain any valid redeemScript, making the P2SH standard much more flexible and allowing for experimentation with many novel and complex types of transactions. The most common use of P2SH is the standard multisig pubkey script, with the second most common use being the `Open Assets Protocol <https://github.com/OpenAssets/open-assets-protocol/blob/master/specification.mediawiki>`__.

Another common redeemScript used for P2SH is storing textual data on the blockchain. The first bitcoin transaction ever made included text, and P2SH is a convenient method of storing text on the blockchain as its possible to store up to 1.5kb of text data. An example of storing text on the blockchain using P2SH can be found in this `repository <https://github.com/petertodd/checklocktimeverify-demos/blob/master/lib/python-bitcoinlib/examples/publish-text.py>`__.

::

   Pubkey script: OP_HASH160 <Hash160(redeemScript)> OP_EQUAL
   Signature script: <sig> [sig] [sig...] <redeemScript>

This script combination looks perfectly fine to old nodes as long as the script hash matches the redeem script. However, after the soft fork is activated, new nodes will perform a further verification for the redeem script. They will extract the redeem script from the signature script, decode it, and execute it with the remaining stack items(<sig> [sig] [sig..]part). Therefore, to redeem a P2SH transaction, the spender must provide the valid signature or answer in addition to the correct redeem script.

This last step is similar to the verification step in P2PKH or P2Multisig scripts, where the initial part of the signature script(<sig> [sig] [sig..]) acts as the “signature script” in P2PKH/P2Multisig, and the redeem script acts as the “pubkey script”.

Multisig
~~~~~~~~

Although P2SH multisig is now generally used for multisig transactions, this base script can be used to require multiple signatures before a UTXO can be spent.

In multisig pubkey scripts, called m-of-n, *m* is the *minimum* number of signatures which must match a public key; *n* is the *number* of public keys being provided. Both *m* and *n* should be opcodes ``OP_1`` through ``OP_16``, corresponding to the number desired.

Because of an off-by-one error in the original Bitcoin implementation which must be preserved for compatibility, :ref:`“OP_CHECKMULTISIG” <term-op-checkmultisig>` consumes one more value from the stack than indicated by *m*, so the list of `secp256k1 <http://www.secg.org/sec2-v2.pdf>`__ signatures in the signature script must be prefaced with an extra value (``OP_0``) which will be consumed but not used.

The signature script must provide signatures in the same order as the corresponding public keys appear in the pubkey script or redeem script. See the description in :ref:`“OP_CHECKMULTISIG” <term-op-checkmultisig>` for details.

::

   Pubkey script: <m> <A pubkey> [B pubkey] [C pubkey...] <n> OP_CHECKMULTISIG
   Signature script: OP_0 <A sig> [B sig] [C sig...]

Although it’s not a separate transaction type, this is a P2SH multisig with 2-of-3:

::

   Pubkey script: OP_HASH160 <Hash160(redeemScript)> OP_EQUAL
   Redeem script: <OP_2> <A pubkey> <B pubkey> <C pubkey> <OP_3> OP_CHECKMULTISIG
   Signature script: OP_0 <A sig> <C sig> <redeemScript>

Pubkey
~~~~~~



Pubkey outputs are a simplified form of the P2PKH pubkey script, but they aren’t as secure as P2PKH, so they generally aren’t used in new transactions anymore.

::

   Pubkey script: <pubkey> OP_CHECKSIG
   Signature script: <sig>

Null Data
~~~~~~~~~

:term:`Null data <Null data transaction>` transaction type relayed and mined by default in `Bitcoin Core 0.9.0 <https://bitcoin.org/en/release/v0.9.0>`__ and later that adds arbitrary data to a provably unspendable pubkey script that full nodes don’t have to store in their UTXO database. It is preferable to use null data transactions over transactions that bloat the UTXO database because they cannot be automatically pruned; however, it is usually even more preferable to store data outside transactions if possible.

Consensus rules allow null data outputs up to the maximum allowed pubkey script size of 10,000 bytes provided they follow all other consensus rules, such as not having any data pushes larger than 520 bytes.

Bitcoin Core 0.9.x to 0.10.x will, by default, relay and mine null data transactions with up to 40 bytes in a single data push and only one null data output that pays exactly 0 satoshis:

::

   Pubkey Script: OP_RETURN <0 to 40 bytes of data>
   (Null data scripts cannot be spent, so there's no signature script.)

Bitcoin Core 0.11.x increases this default to 80 bytes, with the other rules remaining the same.

Bitcoin Core 0.12.0 defaults to relaying and mining null data outputs with up to 83 bytes with any number of data pushes, provided the total byte limit is not exceeded. There must still only be a single null data output and it must still pay exactly 0 satoshis.

The ``-datacarriersize`` Bitcoin Core configuration option allows you to set the maximum number of bytes in null data outputs that you will relay or mine.

Non-Standard Transactions
~~~~~~~~~~~~~~~~~~~~~~~~~

If you use anything besides a standard pubkey script in an output, peers and miners using the default Bitcoin Core settings will neither accept, broadcast, nor mine your transaction. When you try to broadcast your transaction to a peer running the default settings, you will receive an error.

If you create a redeem script, hash it, and use the hash in a P2SH output, the `network <../devguide/p2p_network.html>`__ sees only the hash, so it will accept the output as valid no matter what the redeem script says. This allows payment to non-standard scripts, and as of Bitcoin Core 0.11, almost all valid redeem scripts can be spent. The exception is scripts that use unassigned `NOP opcodes <https://en.bitcoin.it/wiki/Script#Reserved_words>`__; these opcodes are reserved for future soft forks and can only be relayed or mined by nodes that don’t follow the standard mempool policy.

Note: standard transactions are designed to protect and help the `network <../devguide/p2p_network.html>`__, not prevent you from making mistakes. It’s easy to create standard transactions which make the satoshis sent to them unspendable.

As of `Bitcoin Core 0.9.3 <https://bitcoin.org/en/release/v0.9.3>`__, standard transactions must also meet the following conditions:

-  The transaction must be finalized: either its locktime must be in the past (or less than or equal to the current block height), or all of its sequence numbers must be 0xffffffff.

-  The transaction must be smaller than 100,000 bytes. That’s around 200 times larger than a typical single-input, single-output P2PKH transaction.

-  Each of the transaction’s signature scripts must be smaller than 1,650 bytes. That’s large enough to allow 15-of-15 multisig transactions in P2SH using compressed public keys.

-  Bare (non-P2SH) multisig transactions which require more than 3 public keys are currently non-standard.

-  The transaction’s signature script must only push data to the script evaluation stack. It cannot push new opcodes, with the exception of opcodes which solely push data to the stack.

-  The transaction must not include any outputs which receive fewer than 1/3 as many satoshis as it would take to spend it in a typical input. That’s `currently 546 satoshis <https://github.com/bitcoin/bitcoin/commit/6a4c196dd64da2fd33dc7ae77a8cdd3e4cf0eff1>`__ for a P2PKH or P2SH output on a Bitcoin Core node with the default relay fee. Exception: standard null data outputs must receive zero satoshis.

Signature Hash Types
--------------------

:ref:`“OP_CHECKSIG” <term-op-checksig>` extracts a non-stack argument from each signature it evaluates, allowing the signer to decide which parts of the transaction to sign. Since the signature protects those parts of the transaction from modification, this lets signers selectively choose to let other people modify their transactions.

The various options for what to sign are called :term:`signature hash <Signature hash>` types. There are three base SIGHASH types currently available:

-  :term:`“SIGHASH_ALL” <SIGHASH_ALL>`, the default, signs all the inputs and outputs, protecting everything except the signature scripts against modification.

-  :term:`“SIGHASH_NONE” <SIGHASH_NONE>` signs all of the inputs but none of the outputs, allowing anyone to change where the satoshis are going unless other signatures using other signature hash flags protect the outputs.

-  :term:`“SIGHASH_SINGLE” <SIGHASH_SINGLE>` the only output signed is the one corresponding to this input (the output with the same :ref:`output index <term-output-index>` number as this input), ensuring nobody can change your part of the transaction but allowing other signers to change their part of the transaction. The corresponding output must exist or the value “1” will be signed, breaking the security scheme. This input, as well as other inputs, are included in the signature. The sequence numbers of other inputs are not included in the signature, and can be updated.

The base types can be modified with the :term:`“SIGHASH_ANYONECANPAY” <SIGHASH_ANYONECANPAY>` (anyone can pay) flag, creating three new combined types:

-  ``SIGHASH_ALL|SIGHASH_ANYONECANPAY`` signs all of the outputs but only this one input, and it also allows anyone to add or remove other inputs, so anyone can contribute additional satoshis but they cannot change how many satoshis are sent nor where they go.

-  ``SIGHASH_NONE|SIGHASH_ANYONECANPAY`` signs only this one input and allows anyone to add or remove other inputs or outputs, so anyone who gets a copy of this input can spend it however they’d like.

-  ``SIGHASH_SINGLE|SIGHASH_ANYONECANPAY`` signs this one input and its corresponding output. Allows anyone to add or remove other inputs.

Because each input is signed, a transaction with multiple inputs can have multiple signature hash types signing different parts of the transaction. For example, a single-input transaction signed with ``NONE`` could have its output changed by the miner who adds it to the block chain. On the other hand, if a two-input transaction has one input signed with ``NONE`` and one input signed with ``ALL``, the ``ALL`` signer can choose where to spend the satoshis without consulting the ``NONE`` signer—but nobody else can modify the transaction.

Locktime And Sequence Number
----------------------------

One thing all signature hash types sign is the transaction’s :term:`locktime <Locktime>`. (Called nLockTime in the Bitcoin Core source code.) The locktime indicates the earliest time a transaction can be added to the block chain.

Locktime allows signers to create time-locked transactions which will only become valid in the future, giving the signers a chance to change their minds.

If any of the signers change their mind, they can create a new non-locktime transaction. The new transaction will use, as one of its inputs, one of the same outputs which was used as an input to the locktime transaction. This makes the locktime transaction invalid if the new transaction is added to the block chain before the time lock expires.

Care must be taken near the expiry time of a time lock. The `peer-to-peer network <../devguide/p2p_network.html>`__ allows block time to be up to two hours ahead of real time, so a locktime transaction can be added to the block chain up to two hours before its time lock officially expires. Also, blocks are not created at guaranteed intervals, so any attempt to cancel a valuable transaction should be made a few hours before the time lock expires.

Previous versions of Bitcoin Core provided a feature which prevented transaction signers from using the method described above to cancel a time-locked transaction, but a necessary part of this feature was disabled to prevent denial of service attacks. A legacy of this system are four-byte :term:`sequence numbers <Sequence number>` in every input. Sequence numbers were meant to allow multiple signers to agree to update a transaction; when they finished updating the transaction, they could agree to set every input’s sequence number to the four-byte unsigned maximum (0xffffffff), allowing the transaction to be added to a block even if its time lock had not expired.

Even today, setting all sequence numbers to 0xffffffff (the default in Bitcoin Core) can still disable the time lock, so if you want to use locktime, at least one input must have a sequence number below the maximum. Since sequence numbers are not used by the `network <../devguide/p2p_network.html>`__ for any other purpose, setting any sequence number to zero is sufficient to enable locktime.

Locktime itself is an unsigned 4-byte integer which can be parsed two ways:

-  If less than 500 million, locktime is parsed as a block height. The transaction can be added to any block which has this height or higher.

-  If greater than or equal to 500 million, locktime is parsed using the `Unix epoch time <https://en.wikipedia.org/wiki/Unix_time>`__ format (the number of seconds elapsed since 1970-01-01T00:00 UTC—currently over 1.395 billion). The transaction can be added to any block whose block time is greater than the locktime.

Transaction Fees And Change
---------------------------

Transactions pay fees based on the total byte size of the signed transaction. Fees per byte are calculated based on current demand for space in mined blocks with fees rising as demand increases. The transaction fee is given to the Bitcoin miner, as explained in the `block chain section <../devguide/block_chain.html>`__, and so it is ultimately up to each miner to choose the minimum transaction fee they will accept.

There is also a concept of so-called “:term:`high-priority transactions <High-priority transaction>`” which spend satoshis that have not moved for a long time.

In the past, these “priority” transaction were often exempt from the normal fee requirements. Before Bitcoin Core 0.12, 50 KB of each block would be reserved for these high-priority transactions, however this is now set to 0 KB by default. After the priority area, all transactions are prioritized based on their fee per byte, with higher-paying transactions being added in sequence until all of the available space is filled.

As of Bitcoin Core 0.9, a :term:`minimum fee <Minimum relay fee>` (currently 1,000 satoshis) has been required to broadcast a transaction across the `network <../devguide/p2p_network.html>`__. Any transaction paying only the minimum fee should be prepared to wait a long time before there’s enough spare space in a block to include it. Please see the `verifying payment section <../devguide/payment_processing.html#verifying-payment>`__ for why this could be important.

Since each transaction spends Unspent Transaction Outputs (UTXOs) and because a UTXO can only be spent once, the full value of the included UTXOs must be spent or given to a miner as a transaction fee. Few people will have UTXOs that exactly match the amount they want to pay, so most transactions include a change output.

:term:`Change outputs <Change address>` are regular outputs which spend the surplus satoshis from the UTXOs back to the spender. They can reuse the same P2PKH pubkey hash or P2SH script hash as was used in the UTXO, but for the reasons described in the `next subsection <../devguide/transactions.html#avoiding-key-reuse>`__, it is highly recommended that change outputs be sent to a new P2PKH or P2SH address.

Avoiding Key Reuse
------------------

In a transaction, the spender and receiver each reveal to each other all public keys or addresses used in the transaction. This allows either person to use the public block chain to track past and future transactions involving the other person’s same public keys or addresses.

If the same public key is reused often, as happens when people use Bitcoin addresses (hashed public keys) as static payment addresses, other people can easily track the receiving and spending habits of that person, including how many satoshis they control in known addresses.

It doesn’t have to be that way. If each public key is used exactly twice—once to receive a payment and once to spend that payment—the user can gain a significant amount of financial privacy.

Even better, using new public keys or :ref:`unique addresses <term-unique-address>` when accepting payments or creating change outputs can be combined with other techniques discussed later, such as CoinJoin or :ref:`merge avoidance <term-merge-avoidance>`, to make it extremely difficult to use the block chain by itself to reliably track how users receive and spend their satoshis.

Avoiding key reuse can also provide security against attacks which might allow reconstruction of private keys from public keys (hypothesized) or from signature comparisons (possible today under certain circumstances described below, with more general attacks hypothesized).

1. Unique (non-reused) P2PKH and P2SH addresses protect against the first type of attack by keeping `ECDSA <https://en.wikipedia.org/wiki/Elliptic_Curve_DSA>`__ public keys hidden (hashed) until the first time satoshis sent to those addresses are spent, so attacks are effectively useless unless they can reconstruct private keys in less than the hour or two it takes for a transaction to be well protected by the block chain.

2. Unique (non-reused) private keys protect against the second type of attack by only generating one signature per private key, so attackers never get a subsequent signature to use in comparison-based attacks. Existing comparison-based attacks are only practical today when insufficient entropy is used in signing or when the entropy used is exposed by some means, such as a `side-channel attack <https://en.wikipedia.org/wiki/Side_channel_attack>`__.

So, for both privacy and security, we encourage you to build your applications to avoid public key reuse and, when possible, to discourage users from reusing addresses. If your application needs to provide a fixed URI to which payments should be sent, please see the `“bitcoin:” URI section <../devguide/payment_processing.html#bitcoin-uri>`__ below.

Transaction Malleability
------------------------

None of Bitcoin’s signature hash types protect the signature script, leaving the door open for a limited denial of service attack called :term:`transaction malleability <Transaction malleability>`. The signature script contains the `secp256k1 <http://www.secg.org/sec2-v2.pdf>`__ signature, which can’t sign itself, allowing attackers to make non-functional modifications to a transaction without rendering it invalid. For example, an attacker can add some data to the signature script which will be dropped before the previous pubkey script is processed.

Although the modifications are non-functional—so they do not change what inputs the transaction uses nor what outputs it pays—they do change the computed hash of the transaction. Since each transaction links to previous transactions using hashes as a transaction identifier (txid), a modified transaction will not have the txid its creator expected.

This isn’t a problem for most Bitcoin transactions which are designed to be added to the block chain immediately. But it does become a problem when the output from a transaction is spent before that transaction is added to the block chain.

Bitcoin developers have been working to reduce transaction malleability among standard transaction types, one outcome of those efforts is `BIP 141: Segregated Witness <https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki>`__, which is supported by Bitcoin Core and was activated in August 2017. When SegWit is not being used, new transactions should not depend on previous transactions which have not been added to the block chain yet, especially if large amounts of satoshis are at stake.

Transaction malleability also affects payment tracking. Bitcoin Core’s `RPC <../reference/rpc/index.html>`__ interface lets you track transactions by their txid—but if that txid changes because the transaction was modified, it may appear that the transaction has disappeared from the `network <../devguide/p2p_network.html>`__.

Current best practices for transaction tracking dictate that a transaction should be tracked by the transaction outputs (UTXOs) it spends as inputs, as they cannot be changed without invalidating the transaction.

Best practices further dictate that if a transaction does seem to disappear from the `network <../devguide/p2p_network.html>`__ and needs to be reissued, that it be reissued in a way that invalidates the lost transaction. One method which will always work is to ensure the reissued payment spends all of the same outputs that the lost transaction used as inputs.
