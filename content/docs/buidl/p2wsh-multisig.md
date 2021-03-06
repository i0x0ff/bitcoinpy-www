---
title: "P2WSH address (multisig)"
weight: 3
---

{{< tip "warning" >}}
The example for 1-of-1 should only serve as an example. We don't recommend using it in the real
world because it is not its intention. Instead of 1-of-1 use P2PKH!
{{< /tip >}}

## Generate address (1-of-1)

```py
import hashlib

from buidl.ecc import PrivateKey, Signature
from buidl.helper import decode_base58, big_endian_to_int
from buidl.bech32 import decode_bech32, encode_bech32_checksum
from buidl.script import P2PKHScriptPubKey, RedeemScript, WitnessScript, P2WPKHScriptPubKey
from buidl.tx import Tx, TxIn, TxOut
from buidl.witness import Witness

h = hashlib.sha256(b'correct horse battery staple').digest()
private_key = PrivateKey(secret=big_endian_to_int(h), network="signet")

# Create a witnessScript. witnessScript in SegWit is equivalent to redeemScript in P2SH transaction,
# however, while the redeemScript of a P2SH transaction is included in the ScriptSig, the 
# WitnessScript is included in the Witness field, making P2WSH inputs cheaper to spend than P2SH 
# inputs.
witness_script = WitnessScript([private_key.point.sec(), 0xac])

address = witness_script.address("signet")
print('Address:', str(address))
# outputs: tb1qgatzazqjupdalx4v28pxjlys2s3yja9gr3xuca3ugcqpery6c3sqtuzpzy
```

## Spend from address (1-of-1)

Assuming the previously generated address has received funds, we can spend them. In order to spend
them, we'll need information about the transaction id (txid) and a vector of an output (vout). You
can get both from an explorer or by querying your running Bitcoin node by running
[listunspent](https://chainquery.com/bitcoin-cli/listunspent) along with some filters:

`bitcoin-cli listunspent 1 9999999 "[\"address\"]"`

Note that you must have an address in the watchlist in order to get any output. To add an address
to a watchlist run [importaddress](https://chainquery.com/bitcoin-cli/importaddress):

`bitcoin-cli importaddress <address> "<label>" false false`

```py
# we are continuing the code from above

txid = bytes.fromhex("f24d3d8c85ded6d0fbe898a09a2c9f8a8388e4edcf139e52c8714814d85f8273")
vout = 0

# Specify the amount send to your P2WSH address.
COIN = 100000000
amount = int(0.001 * COIN)

# Calculate an amount for the upcoming new UTXO. Set a high fee (5%) to bypass bitcoind minfee
# setting on regtest.
amount_less_fee = int(amount * 0.99)

# Create the txin structure, which includes the outpoint. The scriptSig defaults to being empty as
# is necessary for spending a P2WSH output.
txin = TxIn(txid, vout)

# Specify a destination address and create the txout.
h160 = decode_bech32("tb1qqqlcpznqkfa65wqd48mzzghpwzefgpvtvl0a7k")[2]
txout = TxOut(amount=amount_less_fee, script_pubkey=P2WPKHScriptPubKey(h160))

tx = Tx(1, [txin], [txout], 0, network="signet", segwit=True)

sig1 = tx.get_sig_segwit(0, private_key, witness_script=witness_script)

tx.check_sig_segwit(
    0,
    private_key.point,
    Signature.parse(sig1[:-1]),
    witness_script=witness_script,
)

txin.witness = Witness([sig1, witness_script.raw_serialize()])

print(tx.serialize().hex())
# outputs: 0100000000010173825fd8144871c8529e13cfede488838a9f2c9aa098e8fbd0d6de858c3d4df20000000000ffffffff01b882010000000000160014003f808a60b27baa380da9f62122e170b294058b0247304402201b812e3a58b18bf83ee65db660af469708583073beaecbd4d7147757068e5ece022034a0b51dc40cdbcba1362f544a467e3dc605395f5e0029d5d2e342aea1ddfac20123210378d430274f8c5ec1321338151e9f27f4c676a008bdf8638d07c0b6be9ab35c71ac00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `19e8dc2d719e14bc652bda4809007a72c17bdee6e174d4c02f570c48cad691cd`.

## Generate address (2-of-2)

In this example we show how to create a 2-of-2 multisig address. This means that two signatures are
required in order to unlock funds.

```py
import hashlib

from buidl.ecc import PrivateKey, Signature
from buidl.helper import decode_base58, big_endian_to_int
from buidl.bech32 import decode_bech32, encode_bech32_checksum
from buidl.script import P2PKHScriptPubKey, RedeemScript, WitnessScript, P2WPKHScriptPubKey
from buidl.tx import Tx, TxIn, TxOut
from buidl.witness import Witness

# first key
h = hashlib.sha256(b'correct horse battery staple first').digest()
private_key1 = PrivateKey(secret=big_endian_to_int(h), network="signet")

# second key
h = hashlib.sha256(b'correct horse battery staple second').digest()
private_key2 = PrivateKey(secret=big_endian_to_int(h), network="signet")

# Create a witnessScript. witnessScript in SegWit is equivalent to redeemScript in P2SH transaction,
# however, while the redeemScript of a P2SH transaction is included in the ScriptSig, the 
# WitnessScript is included in the Witness field, making P2WSH inputs cheaper to spend than P2SH 
# inputs.
witness_script = WitnessScript(
    [0x52, private_key1.point.sec(), private_key2.point.sec(), 0x52, 0xAE]
)

address = witness_script.address("signet")
print('Address:', str(address))
# outputs: tb1qljlyqaexx4mmhpl66e6nqdtagjaht87pghuq6p0f98a765c9uj9susmlvt
```

## Spend from address (2-of-2)

Assuming the previously generated address has received funds, we can spend them. In order to spend
them, we'll need information about the transaction id (txid) and a vector of an output (vout). You
can get both from an explorer or by querying your running Bitcoin node by running
[listunspent](https://chainquery.com/bitcoin-cli/listunspent) along with some filters:

`bitcoin-cli listunspent 1 9999999 "[\"address\"]"`

Note that you must have an address in the watchlist in order to get any output. To add an address
to a watchlist run [importaddress](https://chainquery.com/bitcoin-cli/importaddress):

`bitcoin-cli importaddress <address> "<label>" false false`

```py
# we are continuing the code from above

txid = bytes.fromhex("e810379ffa5ca20a30f2210d93517aef28d3f20e20b920190161d4f6491a0903")
vout = 0

# Specify the amount send to your P2WSH address.
COIN = 100000000
amount = int(0.001 * COIN)

# Calculate an amount for the upcoming new UTXO. Set a high fee (5%) to bypass bitcoind minfee
# setting on regtest.
amount_less_fee = int(amount * 0.99)

# Create the txin structure, which includes the outpoint. The scriptSig defaults to being empty as
# is necessary for spending a P2WSH output.
txin = TxIn(txid, vout)

# Specify a destination address and create the txout.
h160 = decode_bech32("tb1qwp3c26rlgzlq4axergvt04300shexn4f56q5f7")[2]
txout = TxOut(amount=amount_less_fee, script_pubkey=P2WPKHScriptPubKey(h160))

tx = Tx(1, [txin], [txout], 0, network="signet", segwit=True)

sig1 = tx.get_sig_segwit(0, private_key1, witness_script=witness_script)
sig2 = tx.get_sig_segwit(0, private_key2, witness_script=witness_script)

tx.check_sig_segwit(
    0,
    private_key1.point,
    Signature.parse(sig1[:-1]),
    witness_script=witness_script,
)

tx.check_sig_segwit(
    0,
    private_key2.point,
    Signature.parse(sig2[:-1]),
    witness_script=witness_script,
)

txin.finalize_p2wsh_multisig([sig1, sig2], witness_script)

print(tx.serialize().hex())
# outputs: 0100000000010103091a49f6d461011920b9200ef2d328ef7a51930d21f2300aa25cfa9f3710e80000000000ffffffff01b882010000000000160014706385687f40be0af4d91a18b7d62f7c2f934ea90400483045022100b24100d90fdd15d3e694789106f780c4c13f4929ec5cc82445418bedac9dfc93022038f5cc4ade88b41f1398d5f1e31a9d4f9d861f67aa24dff30a6d20e509b591c801483045022100fa9f78c7769010a57aa96939b32fad0d216a1dcf3a1f44a09f0e7b29f56b773402207a795dad938599c920d864001b454ae2f44358f5ac098e913049ea7fd9925cb001475221038d19497c3922b807c91b829d6873ae5bfa2ae500f3237100265a302fdce87b052103d3a9dff5a0bb0267f19a9ee1c374901c39045fbe041c1c168d4da4ce0112595552ae00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `6064651b405e0e1d6b5cdc0056c3deda11af3593ca64d43b1cd4abff45b7376b`.


## Generate address (1-of-3)

In this example we show how to create a 1-of-3 multisig address. This means that one out of three
signatures can unlock and spend bitcoins.

```py
import hashlib

from buidl.ecc import PrivateKey, Signature
from buidl.helper import decode_base58, big_endian_to_int
from buidl.bech32 import decode_bech32, encode_bech32_checksum
from buidl.script import P2PKHScriptPubKey, RedeemScript, WitnessScript, P2WPKHScriptPubKey
from buidl.tx import Tx, TxIn, TxOut
from buidl.witness import Witness

# first key
h = hashlib.sha256(b'correct horse battery staple first').digest()
private_key1 = PrivateKey(secret=big_endian_to_int(h), network="signet")

# second key
h = hashlib.sha256(b'correct horse battery staple second').digest()
private_key2 = PrivateKey(secret=big_endian_to_int(h), network="signet")

# third key
h = hashlib.sha256(b'correct horse battery staple third').digest()
private_key3 = PrivateKey(secret=big_endian_to_int(h), network="signet")

# Create a witnessScript. witnessScript in SegWit is equivalent to redeemScript in P2SH transaction,
# however, while the redeemScript of a P2SH transaction is included in the ScriptSig, the 
# WitnessScript is included in the Witness field, making P2WSH inputs cheaper to spend than P2SH 
# inputs.
witness_script = WitnessScript(
    [0x51, private_key1.point.sec(), private_key2.point.sec(), private_key3.point.sec(), 0x53, 0xAE]
)

address = witness_script.address("signet")
print('Address:', str(address))
# outputs: tb1qywycrgq9mvh7lm607hfz2nr99juda6v7ks5gvdtg6qz4l6f80nsqu9wr4h
```

## Spend from address (1-of-3)

Assuming the previously generated address has received funds, we can spend them. In order to spend
them, we'll need information about the transaction id (txid) and a vector of an output (vout). You
can get both from an explorer or by querying your running Bitcoin node by running
[listunspent](https://chainquery.com/bitcoin-cli/listunspent) along with some filters:

`bitcoin-cli listunspent 1 9999999 "[\"address\"]"`

Note that you must have an address in the watchlist in order to get any output. To add an address
to a watchlist run [importaddress](https://chainquery.com/bitcoin-cli/importaddress):

`bitcoin-cli importaddress <address> "<label>" false false`

```py
# we are continuing the code from above

txid = bytes.fromhex("430f7f5ef675a910786f335cabff19d6661910ea365097c6fa1d72e97803ea81")
vout = 0

# Specify the amount send to your P2WSH address.
COIN = 100000000
amount = int(0.001 * COIN)

# Calculate an amount for the upcoming new UTXO. Set a high fee (5%) to bypass bitcoind minfee
# setting on regtest.
amount_less_fee = int(amount * 0.99)

# Create the txin structure, which includes the outpoint. The scriptSig defaults to being empty as
# is necessary for spending a P2WSH output.
txin = TxIn(txid, vout)

# Specify a destination address and create the txout.
h160 = decode_bech32("tb1qwp3c26rlgzlq4axergvt04300shexn4f56q5f7")[2]
txout = TxOut(amount=amount_less_fee, script_pubkey=P2WPKHScriptPubKey(h160))

tx = Tx(1, [txin], [txout], 0, network="signet", segwit=True)

sig2 = tx.get_sig_segwit(0, private_key2, witness_script=witness_script)

tx.check_sig_segwit(
    0,
    private_key2.point,
    Signature.parse(sig2[:-1]),
    witness_script=witness_script,
)

txin.finalize_p2wsh_multisig([sig2], witness_script)

print(tx.serialize().hex())
# outputs: 0100000000010181ea0378e9721dfac6975036ea101966d619ffab5c336f7810a975f65e7f0f430000000000ffffffff01b882010000000000160014706385687f40be0af4d91a18b7d62f7c2f934ea90300473044022036d2596a339e9205d8916d457aeba40332abce8a8435fdb428b1d1b93a8b1af202202fddec53a106e0f8d2d788f83206a3e2e9b9abacf74f0cc16c4458be9c6b89f501695121038d19497c3922b807c91b829d6873ae5bfa2ae500f3237100265a302fdce87b052103d3a9dff5a0bb0267f19a9ee1c374901c39045fbe041c1c168d4da4ce01125955210228769768e3083e084dd5c03b6077d51b7e7a22bd66fc99ff481dcf9b6b80d03053ae00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `3fbd9fe43c30c9af8852f6eaf3e0f1158079195e9b9eea51576445f6c8098fdb`.


## Generate address (2-of-3)

In this example we show how to create a 2-of-3 multisig address. This means that two out of three
signatures can unlock and spend bitcoins.

```py
import hashlib

from buidl.ecc import PrivateKey, Signature
from buidl.helper import decode_base58, big_endian_to_int
from buidl.bech32 import decode_bech32, encode_bech32_checksum
from buidl.script import P2PKHScriptPubKey, RedeemScript, WitnessScript, P2WPKHScriptPubKey
from buidl.tx import Tx, TxIn, TxOut
from buidl.witness import Witness

# first key
h = hashlib.sha256(b'correct horse battery staple first').digest()
private_key1 = PrivateKey(secret=big_endian_to_int(h), network="signet")

# second key
h = hashlib.sha256(b'correct horse battery staple second').digest()
private_key2 = PrivateKey(secret=big_endian_to_int(h), network="signet")

# third key
h = hashlib.sha256(b'correct horse battery staple third').digest()
private_key3 = PrivateKey(secret=big_endian_to_int(h), network="signet")

# Create a witnessScript. witnessScript in SegWit is equivalent to redeemScript in P2SH transaction,
# however, while the redeemScript of a P2SH transaction is included in the ScriptSig, the 
# WitnessScript is included in the Witness field, making P2WSH inputs cheaper to spend than P2SH 
# inputs.
witness_script = WitnessScript(
    [0x52, private_key1.point.sec(), private_key2.point.sec(), private_key3.point.sec(), 0x53, 0xAE]
)

address = witness_script.address("signet")
print('Address:', str(address))
# outputs: tb1q3xlhms84s9vakaa3ds6pzh849mkddvfqmkp43unxyac4stkyyssseuv3rk
```

## Spend from address (2-of-3)

Assuming the previously generated address has received funds, we can spend them. In order to spend
them, we'll need information about the transaction id (txid) and a vector of an output (vout). You
can get both from an explorer or by querying your running Bitcoin node by running
[listunspent](https://chainquery.com/bitcoin-cli/listunspent) along with some filters:

`bitcoin-cli listunspent 1 9999999 "[\"address\"]"`

Note that you must have an address in the watchlist in order to get any output. To add an address
to a watchlist run [importaddress](https://chainquery.com/bitcoin-cli/importaddress):

`bitcoin-cli importaddress <address> "<label>" false false`

```py
# we are continuing the code from above

txid = bytes.fromhex("271ec7372f58eda37ab686a1ad08cd1cac0188b51bcc29d40f8bc5aa06b4db85")
vout = 0

# Specify the amount send to your P2WSH address.
COIN = 100000000
amount = int(0.001 * COIN)

# Calculate an amount for the upcoming new UTXO. Set a high fee (5%) to bypass bitcoind minfee
# setting on regtest.
amount_less_fee = int(amount * 0.99)

# Create the txin structure, which includes the outpoint. The scriptSig defaults to being empty as
# is necessary for spending a P2WSH output.
txin = TxIn(txid, vout)

# Specify a destination address and create the txout.
h160 = decode_bech32("tb1qwp3c26rlgzlq4axergvt04300shexn4f56q5f7")[2]
txout = TxOut(amount=amount_less_fee, script_pubkey=P2WPKHScriptPubKey(h160))

tx = Tx(1, [txin], [txout], 0, network="signet", segwit=True)

sig2 = tx.get_sig_segwit(0, private_key2, witness_script=witness_script)
sig3 = tx.get_sig_segwit(0, private_key3, witness_script=witness_script)

tx.check_sig_segwit(
    0,
    private_key2.point,
    Signature.parse(sig2[:-1]),
    witness_script=witness_script,
)
tx.check_sig_segwit(
    0,
    private_key3.point,
    Signature.parse(sig3[:-1]),
    witness_script=witness_script,
)


txin.finalize_p2wsh_multisig([sig2, sig3], witness_script)

print(tx.serialize().hex())
# outputs: 0100000000010185dbb406aac58b0fd429cc1bb58801ac1ccd08ada186b67aa3ed582f37c71e270000000000ffffffff01b882010000000000160014706385687f40be0af4d91a18b7d62f7c2f934ea904004730440220378efe62e2d47d3a0301a71a4f50263f62d8b6ffd42638a4f1f12ff54b1a653b02204998939a432ddd60a60b5d5051d9abbd282520ad1d9a56852cb83bbb32013abf014830450221008ea8d8ed0bdb1c69390af52786bfe3a5702cf48bf804484b5782eb50f39c23d1022020eaee9a54d8e7bfa2fee5a4ef4a86bad6049c8a8de42ea5ae35c00b98623b3e01695221038d19497c3922b807c91b829d6873ae5bfa2ae500f3237100265a302fdce87b052103d3a9dff5a0bb0267f19a9ee1c374901c39045fbe041c1c168d4da4ce01125955210228769768e3083e084dd5c03b6077d51b7e7a22bd66fc99ff481dcf9b6b80d03053ae00000000
```

Now that we have our signed and encoded transaction, we can broadcast it using
[sendrawtransaction](https://chainquery.com/bitcoin-cli/sendrawtransaction):

`bitcoin-cli sendrawtransaction <transaction>`

If the transaction is broadcasted successfully a transaction id will be returned. In this case it
was `056b104250da42463ef8b203ae5967a4193057a449c575f1d5d74f9cbb238dd2`.