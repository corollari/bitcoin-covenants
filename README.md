## A compilation of research on Bitcoin Covenants {.subtitle}

## What are covenants?
Covenants, also known as spending constraints, is the name given to hipothetical bitcoin scripts that when attached to UTXOs would constrain the way these can be spent, for example restricting the addresses where such coins can be sent.

Covenants which replicate themselves into the UTXOs where they are spent to are called recursive.

## Why are covenants interesting?
Generalized covenants would enable a ton of new applications for scripts, such as:
- Scripts with unbounded length (including arbitrary sized multisigs)
- Turing complete contracts
- Transaction-level MAST (a really coarse-grained version of MAST that would split code paths at the script/transaction level)
- Drivechain-like two-way pegs

Several more use cases are described [on the OP\_CTV page](https://utxos.org/uses/).

**Note**: Not all the covenants described in this document are generalized ones, as several are heavily restricted, therefore the use-cases that each covenant provides are not all the same.

## Why was this created?
Because I spent a whole month researching ways to create covenants with the current capabilities of Bitcoin's Script and, after thinking I had found a beakthrough, I discovered that the same system I had thought about had already been described a few months ago in one of the thousands of posts of the bitcoin mailing list.
The goal of this website is to provide information on bitcoin covenants so that other people don't end up wasting their time like I did.

In other words:

> Those who don't know history are doomed to repeat it

## TL;DR
Bitcoin Cash and the Liquid sidechain have the capabilities needed for building generalized covenants. 

Bitcoin scripts can't use generalized covenants but it's possible to build a very restricted kind of covenants using trusted setups.

Nevertheless, there's several proposals for protocol upgrades that would enable more flexible covenants (yet not general) for Bitcoin, the most active ones being SIGHASH\_ANYPREVOUT/NOINPUT and OP\_CHECKTEMPLATEVERIFY.

## Opcode based covenants
### OP\_CHECKOUTPUTVERIFY
One of the earliest covenant proposals, [this paper](https://maltemoeser.de/paper/covenants.pdf) describes a new opcode to be added to the Bitcoin protocol, this new opcode takes three arguments `(index, value, pattern)` and returns true if and only if the following conditions holdfor the `index`-th output of the transaction that is trying to spend the UTXO:
- The amount of bitcoin spent on that output is equal to `value`
- The script attached to the output is equal to `pattern` except for the parts of `pattern` that contain placeholders for public keys and hashes.

A technically-wrong but exemplary implementation of the opcode in python would be as follows:
```python
def CheckOutputVerify(tx, index, value, pattern):
	output = tx.outputs[index]
	if value != 0:
		if output.value != value:
			return False
	if pattern != "":
		sanitizedPattern = pattern.replace("PubKeyPlaceholder", "00000000").replace("HashPlaceholder", "00000000")
		mask = replaceLocationsOfPlaceholdersWithZeroes("1"*len(sanitizedPattern), ["PubKeyPlaceholder", "HashPlaceholder"])
		if (output.script & mask) != sanitizedPattern:
			return False	
	return True
```
**Note**: This implementation is wrong due to the fact that the binary operations are replaced with string operations, which are themselves wrong in lots of ways (a mask of bytes is not constructed as a string of 1's...) but as a pseudocode example it helps get the concept across

This opcode would enable all kind of covenants, but nowadays it has been superseded by other systems and the probability of it ever getting deployed is null.

### OP\_CHECKOUTPUTSHASHVERIFY
This opcode, which was proposed as an extension to tapscript and aims at being as simple as possible, just serializes the outputs of a transaction, hashes the result twice and compares it with an argument:
```python
def CheckOutputsHashVerify(tx, outputsHash):
	if sha256(sha256(tx.outputs)) == outputsHash:
		return True
	else:
		return False
```

Something uncommon about this opcode is the fact that it's argument is provided in Script after OP\_CHECKOUTPUTSHASHVERIFY, not before. This behaviour is completely different from other opcodes', which take arguments from the stack, in order to prevent the output hashes from being constructed in Script.

The proposal was superseded by OP\_CHECKTEMPLATEVERIFY, which modifies the behaviour to enforce non-malleability in transactions that spend UTXOs encumbered by the covenant and changes the deployment scheme to the classic NOP-replacement instead of a tapscrip extension.

See the [BIP](https://github.com/JeremyRubin/bips/blob/op-checkoutputshashverify/bip-coshv.mediawiki) for more details.

### OP\_CHECKTEMPLATEVERIFY
Previously known as OP_SECURETHEBAG, this is a new version of OP\_CHECKOUTPUTSHASHVERIFY that enforces non-malleability of transactions by including several other parts of the transaction into the hash, whereas OP\_CHECKOUTPUTSHASHVERIFY only included the outputs.  

Specifically, this opcode puts together the following fields:
- Version bits
- nLockTime
- Input's scriptSig
- Number of inputs
- nSequence
- Number of outputs
- Outputs
- Index of the input being verified currently

serializes them together, hashes them and compares the hash with an element in the stack, verifying the transaction if the values are equal.
<!--Some of the values are hashed before serialization, this is not specified here to keep things simple. Read the BIP!-->

For more information, see [BIP119](https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki) and [the website dedicated to it](https://utxos.org/). 

## Signature based covenants
### A primer on signatures
The following is a short explanation on Schnorr signatures, the signatures used in Bitcoin are not of this type but they are conceptually similar:

<!-- This description of schnorr signatures is not mathematically rigorous, as it is missing a ton of details, it is just meant to offer a didactic explanation, if you want to learn it properly check a proper source -->
The main idea behind this signatures is that if you have a point $G$ of a elliptic curve and a number $n$ you can easily multiply them together to obtain another point in the curve $n \cdot G=N$ so that guessing $n$ from just $N$ will be really hard, next to impossible.

With that in mind, we can take a number $p$ that will be our private key and calculate $p \cdot G=P$ to obtain a public key $P$ ($G$ is just a picked point in the curve), then given a transaction $tx$ we will sign it by picking a random number $k$, calculating $s = k - hash(tx) \cdot p$ and $K=k \cdot G$, and constructing a signature as $(s, K)$.

With these values, anyone else can verify that such signature is valid by checking that the equality $s \cdot G = K - hash(tx) \cdot P$ holds, as only someone that knows the value $p$ can construct such a signature due to the fact that if someone with no such knowledge tried to compute $s$ they would need to compute the point $S=K- hash(tx) \cdot P$ and reverse the multiplication to obtain a number that when multiplied by $G$ would yield $S$, which we assumed impossible in the first paragraph of this explanation.

The main idea behind this category of covenants (signature based) is that, while transaction information is not directly available inside Script, it is used to construct the value $hash(tx)$ used internally in OP\_CHECKSIG, so it's possible to perform comparisons against it indirectly through hacks in the signatures passed to OP\_CHECKSIG.

### OP\_CHECKSIGFROMSTACK
At it's heart, this opcode is really simple, it just takes a message, a public key and a signature and checks if the signature is valid. The interesting bit is that you can use this to compare any transaction built inside Script to the transaction that triggered the call and check if they are equal, therefore getting access to all the transaction data that is included inside the hash in OP\_CHECKSIG signatures.

More especifically, this would work by having the verification script (ScriptPubKey) of a UTXO construct (or verify) a serialized transaction, making sure that certain properties are met, such as the outputs being equal to ones defined previously for example, then hash such transaction and run OP\_CHECKSIGFROMSTACK on the resulting hash and the values $(s, K, P)$ taken from the redeem script (ScriptSig) and computed off-chain by the user.

If that signature is valid, then OP\_CHECKSIG would be run with the same values except for the hash, that is the values $(s, K, P)$ used before. If this other opcode also returns true then we can be sure that the transaction spending the UTXO is the same as the one we have constructed (and have enforced our arbitrary conditions on). This is because $s \cdot G = K + hash(tx_{constructed}) \cdot P$ holds if and only if $s \cdot G = K + hash(tx_{real}) \cdot P$ holds.

For more information, see the [paper](https://fc17.ifca.ai/bitcoin/papers/bitcoin17-final28.pdf) and [article](https://blockstream.com/2016/11/02/en-covenants-in-elements-alpha/) about it.

**Note**: This opcode has been implemented inside Elements, the blockchain layer upon which Blockstream's Liquid sidechain is built. Furthermore, an opcode very similar to it called [OP\_CHECKDATASIG](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/op_checkdatasig.md) has been implemented and deployed in Bitcoin Cash.

### SIGHASH\_NOINPUT/ANYPREVOUTS
A kind of somewhat restricted covenants can be created if SIGHASH\_NOINPUT or ANYPREVOUTS is deployed. These covenants are based on creating several transactions beforehand and then constructing public keys for which only a single signature can be computed, making any money sent to that public key only spendable through the transactions defined before.

In other words, someone could create a covenant using the following protocol:
1. Create a transaction $tx$ spending some funds
2. Pick a number $s$ and a point $K$ in a deterministic and verifiable way (could also be constants)
3. Calculate a public key by solving for $P$ in the equation $sG = K - hash(tx)P$, that is computing $P = hash(x)^{-1}(K - sG)$.
4. Send funds to an address where they can only be spent with a signature of $P$.

Because the private key associated with $P$ is unknown, it's not possible to construct any signatures from it apart from the signature used to compute it, signature which signs the specific transaction $tx$. Therefore $tx$ will be the only transaction that will be able to spend any funds locked in $P$.

Now, the huge problem with this scheme is that to construct the transaction $tx$ , and by extension $P$, you need the txids of the transactions that send money to $P$ (step 4), but to create these transactions you need to know $P$, thus you get locked in an unsolvable cyclic dependency.

SIGHASH_NOINPUT solves this problem by allowing $hash(tx)$ to not commit to the txids of the transaction, therefore enabling the computation of $P$ without any other knowledge apart from the outputs of $tx$.

Check out [the bitcoin-dev post where this scheme was initially described](https://www.mail-archive.com/bitcoin-dev@lists.linuxfoundation.org/msg08075.html) for more details.

### Signature construction 
Another similar scheme that trades the dependency on SIGHASH_NOINPUT for a dependency on any opcode that enables comparison against a part of a word on the stack (eg: OP_AND, OP_SUBSTR, OP_CAT...), while allowing the use of generalized covenants, is based on directly constructing the signature for an approved transaction, inside script:
1. Inside script, obtain a transaction $tx$ which verifies all the properties that the covenant enforces
2. Inside script, compute $hash(tx)+1$ and then apply OP\_CHECKSIG using $-G$ as the public key ($P$), $G$ as the nonce ($K$) and the value that we just computed as $s$. An alternative interpretation of this is that the private key $p$ is set to $-1$ and the nonce $k$ to $1$.
3. If OP\_CHECKSIG succeeds the transaction being spent is the same as $tx$

They point of this scheme is that when OP\_CHECKSIG is applied, it results in the equation $sG = G - hash(tx)(-G)$ being checked, which will only hold if and only if $s = hash(tx) + 1$, therefore by constructing $s = hash(tx_{constructed}) + 1$ we end up with $hash(tx_{constructed}) + 1 = hash(tx_{real}) + 1$, which is a direct equality check between the transaction inside script and the transaction being spent.

You may wonder why do we use $1$ instead of $0$, as that would remove all the annoying "+1" from our calculations. The reason behind that is simply that the standard doesn't accept $k=1$.

### Multi-signature variant

---

This text is open source (MIT licensed), and available [on GitHub](https://github.com/corollari/bitcoin-covenants). Contributions are welcome.
