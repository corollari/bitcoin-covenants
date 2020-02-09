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
The following is a short explanation on Schnorr signatures, Bitcoin doesn't use this kind of signatures (there's a proposal to include them in the protocol tho) but instead uses ECDSA, which is a bit more complicated than Schnorr but conceptually the same:

### OP\_CHECKSIGFROMSTACK

For more information, see the [paper](https://fc17.ifca.ai/bitcoin/papers/bitcoin17-final28.pdf) and [article](https://blockstream.com/2016/11/02/en-covenants-in-elements-alpha/) about it.

**Note**: This opcode has been implemented inside Elements, the blockchain layer upon which Blockstream's Liquid sidechain is built. Furthermore, an opcode very similar to it called [OP\_CHECKDATASIG](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/op_checkdatasig.md) has been implemented and deployed in Bitcoin Cash.

### Signature costruction 
Approaches based on constructing a public key so that only a single signature for it is known.

Given a curve €C€, a generator point €G in C€, a private key €p€ and a nonce €k€, the public key €P in C€ is usually computed as €P=pG€ and, given a message €m€ a signature for it is created by computing €K=kG, s = k-hash(m, K)*p€ and constructing €(s, K, m)€, which along with €P€ enables the verification of the signature.
The construction described allows the owner of Instead of constructing the signature in this way, this proposal uses the following algorithm:
Given a transaction input  
1. Construct a transaction 
2. Compute a definition of €s in ZZ_p€ and €K in C€ in a deterministic way from the message, for example using €s=hash1(m), K=hash2(m)€ (€hash2€ should return a valid point in €C€)
3. Compute €P€ by solving the equation €sG = K-hash(m, K)*P€

### Multi-party Computation variant

---

This text is open source (MIT licensed), and available [on github](https://github.com/corollari/bitcoin-covenants). Contributions are welcome.
