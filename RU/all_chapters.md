## Как создавать базирующиеся на UTXO (выходах) контракты CryptoConditions для блокчейнов в кластере KMD - автор jl777

Исходный текст: https://github.com/jl777/komodo/blob/FSM/src/cc/CC%20made%20easy

Это не единственная методология смарт-контрактов, которую возможно построить на OP_CHECKCRYPTOCONDITION, просто она является первой. Вся заслуга за то, что OP_CHECKCRYPTOCONDITION работает с кодовой базой Komodo принадлежит @libscott. Я просто цепляюсь к коду и пытаюсь сделать его немного проще для создания новых контрактов.

Пожалуй можно придумать какое нибудь причудливое имя в целях маркетинга, но сейчас я просто коротко назову это "CC контракты", зная что это не на 100% технически точно, так как аспект CryptoConditions не является главной характерной чертой. Однако, контракты KMD были построены на кодовой базе CryptoConditions, которая была интегрирована в них, чтобы стать более доступной.
Поскольку CC контракты запускают нативный C/C++ код, они являются Тьюринг-полными, а это значит, что, любой контракт который возможно создать на любой другой платформе можно реализовать через CC контракт.

Начать писать контракты на основе UTXO немного сложнее чем для контрактов основанных на баланах. Однако, они гораздо более безопасны, поскольку используют существующую систему UTXO Bitcoin. Это делает гораздо менее вероятным появление багов, из-за которых можно выпустить квадриллион новых монет, поскольку все операции с CC контрактом также должны подчиняться существующему UTXO протоколу Bitcoin.

Данный документ будет в значительной степени основан на примерах, то есть будет использовать много отсылок к существующим CC. После осознания данного документа, вы должны быть достаточно подготовлены чтобы начать создавать новые CC контракты для интеграции в komodod, либо непосредственно dAPPS на основе RPC.
- [Chapter 0 - Bitcoin Protocol Basics](#chapter-0---bitcoin-protocol-basics)
- [Chapter 1 - OP_CHECKCRYPTOCONDITION](#chapter-1---op_checkcryptocondition)
- [Chapter 2 - CC contract basics](#chapter-2---cc-contract-basics)
- [Chapter 3 - CC vouts](#chapter-3---cc-vins-and-vouts)
- [Chapter 4 - CC RPC extensions](#chapter-4---cc-rpc-extensions)
- [Chapter 5 - CC validation](#chapter-5---cc-validation)
- [Chapter 6 - faucet example](#chapter-6---faucet-example)
- Chapter 7 - rewards example
- Chapter 8 - assets example
- Chapter 9 - dice example
- Chapter 10 - lotto example
- Chapter 11 - channels example
- [Chapter 12 - limitless possibilities](#chapter-12---limitless-possibilities)
- [Chapter 13 - different languages](#chapter-13---different-languages)
- [Chapter 14 - runtime bindings](#chapter-14---runtime-bindings)
- [Chapter 15 - RPC based dapps](#chapter-15---rpc-based-dapps)
- [Conclusion](#conclusion)



## Chapter 0 - Bitcoin Protocol Basics
There are many aspects of the bitcoin protocol that isnt needed to understand the CC contracts dependence on it. Such details will not be discussed. The primary aspect is the UTXO, Unspent Transaction Output. Just a fancy name for `txid/vout`, so when you sendtoaddress some coins, it creates a `txid` and the first output is `vout.0`, combine it and `txid/0` is a specific UTXO.

Of course, to understand even this level of detail requires that you understand what a `txid` is, but there are plenty of reference materials on that. It is basically the 64 char long set of letters and numbers that you get when you send funds.

Implicit with the UTXO is that it prevents double spends. Once you spend a UTXO, you cant spend it again. This is quite an important characteristic and while advanced readers will point out chain reorgs can allow a double spend, we will not confuse the issue with such details. The important thing is that given a blockchain at a specific height's blockhash, you can know if a `txid/vout` has been spent or not.

There are also the transactions that are in memory waiting to be mined, the mempool. And it is possible for the utxo to be spent by a `tx` in the mempool. However since it isnt confirmed yet, it is still unspent at the current height, even if we are pretty sure it will be spent in the next block.

A useful example is to think about a queue of people lined up to get into an event. They need to have a valid ticket and also to get into the queue. After some time passes, they get their ticket stamped and allowed into the event.

In the UTXO case, the ticket is the spending transaction and the event is the confirmed blockchain. The queue is the mempool.



## Chapter 1 - OP_CHECKCRYPTOCONDITION
In the prior chapter the UTXO was explained. However, the specific mechanism used to send a payment was not explained. Contrary to what most people might think, on the blockchain there are not entries that say "pay X amount to address". Instead what exists is a bitcoin script that must be satisfied in order for the funds to be able to be spent.

Originally, there was the pay to pubkey script:

```
<pubkey> <checksig>
```

About as simple of a payment script that you can get. Basically the pubkey's signature is checked and if it is valid, you get to spend it. Once problem satoshi realized was that with Quantum Computers such payment scripts are vulnerable! So, he made a way to have a cold address, ie. an address whose pubkey isnt known. At least it isnt known until it is spent, so it is only Quantum resistant prior to the first spend. This line of reasoning is why we have one time use addresses and a new change address for each transaction. Maybe in some ways, this is too forward thinking as it makes things a lot more confusing to use and easier to lose track of all the required private keys.

However, it is here to stay and its script is:

```
<hash the pubkey> <pubkey> <verify hash matches> <checksig>
```

With this, the blockchain has what maps to "pay to address", just that the address is actually a base58 encoded (prefix + pubkeyhash). Hey, if it wasnt complicated, it would be easy!

In order to spend a `p2pkh` (pay to pubkey hash) UTXO, you need to divulge the pubkey in addition to having a valid signature. After the first spend from an address, its security is degraded to `p2pk` (pay to pubkey) as its pubkey is now known. The net result is that each reused address takes 25 extra bytes on the blockchain, and that is why for addresses that are expected to be reused, I just use the `p2pk` script.

Originally, bitcoin allowed any type of script opcodes to be used directly. The problem was some of them caused problems and satoshi decided to disable them and only allow standard forms of payments. Thus the `p2pk` and `p2pkh` became 99%+ of bitcoin transactions. However, going from having a fully scriptable language that can create countless payment scripts (and bugs!), to having just 2... well it was a "short term" limitation. It did last for some years but eventually a compromise `p2sh` script was allowed to be standard. This is a pay to script hash, so it can have a standard format as the normal `p2pkh`, but have infinitely more flexibility.

```
<hash the script> <script> <verify hash matches>
```

Wait, something is wrong! If it was just that, then anybody that found out what the required script (called redeemscript) was, they could just spend it. I forgot to say that the redeemscript is then used to determine if the payment can be spent or not. So you can have a normal `p2pk` or `p2pkh` redeemscript inside a `p2sh` script.

OK, I know that just got really confusing. Let us have a more clear example:

```
redeemscript <- pay to pubkey 
```

`p2sh` becomes the hash of the redeem script + the compares

So to spend it, you need to divulge the redeemscript, which in turn requires you to divulge the pubkey. Put it all together and the `p2sh` mechanism verifies you not only had the correct redeemscript by comparing its hash, but that when the redeemscript is run, it is satisfied. In this case, that the pubkey's signature was valid.

If you are still following, there is some good news! `OP_CHECKCRYPTOCONDITION` scripts are actually simpler than `p2sh` scripts in some sense as there isnt this extra level of script inside a scripthash. [@libscott](https://github.com/libscott) implemented the addition of `OP_CHECKCRYPTOCONDITION` to the set of bitcoin opcodes and what it does is makes sure that a CryptoConditions script is properly signed.

Which gets us to the CryptoConditions specification, which is a monster of a [IETF (Internet standards) draft](https://tools.ietf.org/html/draft-thomas-crypto-conditions-04) and has hundred(s) of pages of specification. I am sure you are happy to know that you dont really need to know about it much at all! Just know that you can create all sorts of cryptoconditions and its binary encoding can be used in a bitcoin UTXO. If the standard CC contracts dont have the power you need, it is always possible to expand on it. So far, most all the CC contracts only need the power of a 1of1 CC script, which is 1 signature combined with custom constraints. The realtime payment channels CC is the only one of the reference CC contracts so far that didnt fit into this model, it needed a 1of2 CC script.

The best part is that all these opcode level things are not needed at all. I just wanted to explain it for those that need to know all the details of everything.



## Chapter 2 - CC Contract Basics
Each CC contract has an eval code, this is just an arbitrary number that is associated with a specific CC contract. The details about a specific CC contract are all determined by the validation logic, that is ultimately what implements a CC contract.

However, unlike the normal bitcoin payments, where it is validated with only information in the transaction, a CC contract has the power to do pretty much anything. It has full access to the blockchain and even the mempool, though using mempool information is inherently more risky and needs to be done carefully or for exclusions, rather than inclusions.

However, this is the CC contract basics chapter, so let us ignore mempool issues and deal with just the basics. Fundamentally there is no structure for `OP_CHECKCRYPTOCONDITION` serialized scripts, but if you are like me, you want to avoid having to read and understand a 1000 page IETF standard. What we really want to do is have a logical way to make a new contract and have it be able to be coded and debugged in an efficient way.

That means to just follow a known working template and only changing the things where the existing templates are not sufficient, ie. the core differentiator of your CC contract.

In the [~/komodo/src/cc/eval.h](https://github.com/jl777/komodo/tree/jl777/src/cc/eval.h) file all the eval codes are defined, currently:

```C
#define FOREACH_EVAL(EVAL)             \
        EVAL(EVAL_IMPORTPAYOUT, 0xe1)  \
        EVAL(EVAL_IMPORTCOIN,   0xe2)  \
        EVAL(EVAL_ASSETS,   0xe3)  \
        EVAL(EVAL_FAUCET, 0xe4) \
        EVAL(EVAL_REWARDS, 0xe5) \
        EVAL(EVAL_DICE, 0xe6) \
        EVAL(EVAL_FSM, 0xe7) \
        EVAL(EVAL_AUCTION, 0xe8) \
        EVAL(EVAL_LOTTO, 0xe9) \
        EVAL(EVAL_MOFN, 0xea) \
        EVAL(EVAL_CHANNELS, 0xeb) \
        EVAL(EVAL_ORACLES, 0xec) \
        EVAL(EVAL_PRICES, 0xed) \
        EVAL(EVAL_PEGS, 0xee) \
        EVAL(EVAL_TRIGGERS, 0xef) \
        EVAL(EVAL_PAYMENTS, 0xf0) \
        EVAL(EVAL_GATEWAYS, 0xf1)
```

Ultimately, we will probably end up with all 256 eval codes used, for now there is plenty of room. I imagined that similar to my coins repo, we can end up with a much larger than 256 number of CC contracts and you select the 256 that you want active for your blockchain. That does mean any specific chain will be limited to "only" having 256 contracts. Since there seems to be so few actually useful contracts so far, this limit seems to be sufficient. I am told that the evalcode can be of any length, but the current CC contracts assumes it is one byte. 

The simplest CC script would be one that requires a signature from a pubkey along with a CC validation. This is the equivalent of the pay to pubkey bitcoin script and is what most of the initial CC contracts use. Only the channels one needed more than this and it will be explained in its chapter.
We end up with CC scripts of the form (`evalcode`) + (`pubkey`) + (other stuff), dont worry about the other stuff, it is automatically handled with some handy internal functions. The important thing to note is that each CC contract of this form needs a single pubkey and eval code and from that we get the CC script. Using the standard bitcoin's "hash and make an address from it" method, this means that the same pubkey will generate a different address for each different CC contract!

This is an important point, so I will say it in a different way. In bitcoin there used to be uncompressed pubkeys which had both the right and left half combined, into a giant 64 byte pubkey. But since you can derive one from the other, compressed pubkeys became the standard, that is why you have bitcoin pubkeys of 33 bytes instead of 65 bytes. There is a 02, 03 or 04 prefix, to mean odd or even or big pubkey. This means there are two different pubkeys for each privkey, the compressed and uncompressed. And in fact you can have two different bitcoin protocol addresses that are spendable by the same privkey. If you use some paper wallet generators, you might have noticed this.

CC contracts are like that, where each pubkey gets a different address for each evalcode. It is the same pubkey, just different address due to the actual script having a different evalcode, it ends up with a different hash and thus a different address. Now funds send to a specific CC address is only accessible by that CC contract and must follow the rules of that contract.
I also added another very useful feature where the convention is for each CC contract to have a special address that is known to all, including its private key. Before you panic about publishing the private key, remember that to spend a CC output, you need to properly sign it AND satisfy all the rules. By everyone having the privkey for the CC contract, everybody can do the "properly sign" part, but they still need to follow the rest of the rules.

From a user's perspective, there is the global CC address for a CC contract and some contracts also use the user pubkey's CC address. Having a pair of new addresses for each contract can get a bit confusing at first, but eventually we will get easy to use GUI that will make it all easy to use.



## Chapter 3 - CC vins and vouts
You might want to review the bitcoin basics and other materials to refresh about how bitcoin outputs become inputs. It is a bit complicated, but ultimately it is about one specific amount of coins that are spent, once spent it is combined with the other coins that are also spent in that transaction and then various outputs are created.

```
vin0 + vin1 + vin2 -> vout0 + vout1
```

That is a 3 input, 2 output transaction. The value from the three inputs are combined and then split into vout0 and vout1, each of the vouts gets a spend script that must be satisfied to be able to be spent. Which means for all three of out vins, all the requirements (as specified in the output that created them) are satisfied.

Yes, I know this is a bit too complicated without a nice chart, so we will hope that a nice chart is added here:

`[nice chart goes here]`

Out of all the aspects of the CC contracts, the flexibility that different vins and vouts created was the biggest surprise. When I started writing the first of these a month ago, I had no idea the power inherent in the smart utxo contracts. I was just happy to have a way to lock funds and release them upon some specific conditions.

After the assets/tokens CC contract, I realized that it was just a tip of the iceberg. I knew it was Turing complete, but after all these years of restricted bitcoin script, to have the full power of any arbitrary algorithm, it was eye opening. Years of writing blockchain code and having really bad consequences with every bug naturally makes you gun shy about doing aggressive things at the consensus level. And that is the way it should be, if not very careful, some really bad things can and do happen. The foundation of building on top of the existing (well tested and reliable) utxo system is what makes the CC contracts less likely for the monster bugs. That being said, lack of validation can easily allow an improperly coded CC contract to have its funds drained.

The CC contract breaks out of the standard limitations of a bitcoin transaction. Already, what I wrote explains the reason, but it was not obvious even to me at first, so likely you might have missed it too. If you are wondering what on earth I am talking about, THAT is what I am talking about!

To recap, we have now a new standard bitcoin output type called a CC output. Further, there can be up to 256 different types of CC outputs active on any given blockchain. We also know that to spend any output, you need to satisfy its spending script, which in our case is the signature and whatever constraints the CC validation imposes. We also have the convention of a globally shared keypair, which gives us a general CC address that can have funds sent to it, along with a user pubkey specific CC address.
Let us go back to the 3+2 transaction example:

```
vin0 + vin1 + vin2 -> vout0 + vout1
```

Given the prior paragraph, try to imagine the possibilities the simple 3+2 transaction can be. Each `vin` could be a normal `vin`, from the global contract address, the user's CC address and the vouts can also have this range. Theoretically, there can be 257 * 257 * 257 * 257 * 257 forms of a 3+2 transaction!

In reality, we really dont want that much degrees of freedom as it will ensure a large degree of bugs! So we need to reduce things to a more manageable level where there are at most 3 types for each, and preferably just 1 type. That will make the job of validating it much simpler and simple is better as long as we dont sacrifice the power. We dont.

Ultimately the CC contract is all about how it constrains its inputs, but before it can constrain them, they need to be created as outputs. More about this in the CC validation chapter.



## Chapter 4 - CC RPC Extensions
Currently, CC contracts need to be integrated at the source level. This limits who is able to create and add new CC contracts, which at first is good, but eventually will be a too strict limitation. The runtime bindings chapter will touch on how to break out of the source based limitation, but there is another key interface level, the RPC.

By convention, each CC contract adds an associated set of RPC calls to the `komodo-cli`. This not only simplifies the creation of the CC contract transactions, it further will allow dapps to be created just via RPC calls. That will require there being enough foundational CC contracts already in place. As we find new usecases that cannot be implemented via rpc, then a new CC contract is made that can handle that (and more) and the power of the RPC level increases. This is a long term process.

The typical RPC calls that are added `<CC>address`, `<CClist>`, `<CCinfo>` return the various special CC addresses, the list of CC contract instances and info about each CC contract instance. Along with an RPC that creates a CC instance and of course the calls to invoke a CC instance.
The role of the RPC calls are to create properly signed `rawtransactions` that are ready for broadcasting. This then allows using only the RPC calls to not only invoke but to create a specific instance of a CC. The faucet contract is special in that it only has a single instance, so some of these RPC calls are skipped.

So, there is no MUSTHAVE RPC calls, just a sane convention to follow so it fits into the general pattern.

One thing that I forgot to describe was how to create a special CC address and even though this is not really an RPC issue, it is kind of separate from the core CC functions, so I will show how to do it here:

```C
const char *FaucetCCaddr = "R9zHrofhRbub7ER77B7NrVch3A63R39GuC";
const char *FaucetNormaladdr = "RKQV4oYs4rvxAWx1J43VnT73rSTVtUeckk";
char FaucetCChexstr[67] = { "03682b255c40d0cde8faee381a1a50bbb89980ff24539cb8518e294d3a63cefe12" };
uint8_t FaucetCCpriv[32] = { 0xd4, 0x4f, 0xf2, 0x31, 0x71, 0x7d, 0x28, 0x02, 0x4b, 0xc7, 0xdd, 0x71, 0xa0, 0x39, 0xc4, 0xbe, 0x1a, 0xfe, 0xeb, 0xc2, 0x46, 0xda, 0x76, 0xf8, 0x07, 0x53, 0x3d, 0x96, 0xb4, 0xca, 0xa0, 0xe9 };
```

Above are the specifics for the faucet CC, but each one has the equivalent in [CCcustom.cpp](https://github.com/jl777/komodo/tree/jl777/src/cc/CCcustom.cpp). At the bottom of the file is a big switch statement where these values are copied into an in memory data structure for each CC type. This allows all the CC codebase to access these special addresses in a standard way.

In order to get the above values, follow these steps:

A. use `getnewaddress` to get a new address and put that in the `<CC>Normaladdr = "";` line

B. use `validateaddress` `<newaddress from A>` to get the pubkey, which is put into the `<CC>hexstr[67] = "";` line

C. stop the daemon [`komodod`] and start with `-pubkey=<pubkey from B>` and do a `<CC>address` RPC call. In the console you will get a printout of the hex for the privkey, assuming the if ( 0 ) in `Myprivkey()` is enabled ([CCutils.cpp](https://github.com/jl777/komodo/tree/jl777/src/cc/CCutils.cpp))

D. update the `CCaddress` and `privkey` and dont forget to change the `-pubkey=` parameter

The first RPC command to add is `<CC>address` and to do that, add a line to [rpcserver.h](https://github.com/jl777/komodo/tree/jl777/src/rpcserver.h) and update the commands array in [rpcserver.cpp](https://github.com/jl777/komodo/tree/jl777/src/rpcserver.cpp)

In the [rpcwallet.cpp](https://github.com/jl777/komodo/tree/jl777/src/wallet/rpcwallet.cpp) file you will find the actual RPC functions, find one of the `<CC>address` ones, copy paste, change the eval code to your eval code and customize the function. Oh, and dont forget to add an entry into [eval.h](https://github.com/jl777/komodo/tree/jl777/src/cc/eval.h)

Now you have made your own CC contract, but it wont link as you still need to implement the actual functions of it. This will be covered in the following chapters.



## Chapter 5 - CC Validation
CC validation is what its all about, not the "hokey pokey"!

Each CC must have its own validation function and when the blockchain is validating a transaction, it will call the CC validation code. It is totally up to the CC validation whether to validate it or not.

Any set of rules that you can think of and implement can be part of the validation. Make sure that there is no ambiguity! Make sure that all transactions that should be rejected are in fact rejected.

Also, make sure any RPC calls that create a CC transaction dont create anything that doesnt validate.

Really, that is all that needs to be said about validation that is generic, as it is just a concept and gets a dedicated function to determine if a transaction is valid or not.

For most of the initial CC contracts, I made a function code for various functions of the CC contract and add that along with the creation txid. That enables the validation of the transactions much easier, as the required data is right there in the opreturn.

You do need to be careful not to cause a deadlock as the CC validation code is called while already locked in the main loop of the bitcoin protocol. As long as the provided CC contracts are used as models, you should keep out of deadlock troubles.




## Chapter 6 - Faucet Example
Finally, we are ready for the first actual example of a CC contract. The faucet. This is a very simple contract and it ran into some interesting bugs in the first incarnation.

The code in [~/komodo/src/cc/faucet.cpp](https://github.com/jl777/komodo/tree/jl777/src/cc/faucet.cpp) is the ultimate documentation for it with all the details, so I will just address the conceptual issues here.

There are only 7 functions in [faucet.cpp](https://github.com/jl777/komodo/tree/jl777/src/cc/faucet.cpp), a bit over 200 lines including comments. The first three are for validation, the last four for the RPC calls to use. 

```C
int64_t IsFaucetvout(struct CCcontract_info *cp,const CTransaction& tx,int32_t v)

bool FaucetExactAmounts(struct CCcontract_info cp,Eval eval,const CTransaction &tx,int32_t minage,uint64_t txfee)

bool FaucetValidate(struct CCcontract_info cp,Eval eval,const CTransaction &tx)

int64_t AddFaucetInputs(struct CCcontract_info *cp,CMutableTransaction &mtx,CPubKey pk,int64_t total,int32_t maxinputs)

std::string FaucetGet(uint64_t txfee)

std::string FaucetFund(uint64_t txfee,int64_t funds)

UniValue FaucetInfo()
```

Functions in `rpcwallet` implement:

`faucetaddress` fully implemented in [rpcwallet.cpp](https://github.com/jl777/komodo/tree/jl777/src/wallet/rpcwallet.cpp)
`faucetfund` calls `FaucetFund`
`faucetget` calls `FaucetGet`
`faucetinfo` calls `FaucetInfo`


Now you might not be a programmer, but I hope you are able to understand the above sequence. user types in a cli call, `komodo-cli` processes it by calling the RPC function, which in turn calls the function inside [faucet.cpp](https://github.com/jl777/komodo/tree/jl777/src/cc/faucet.cpp)

No magic, just simple conversion of a user command line call that runs code inside the komodod. Both the `faucetfund` and `faucetget` create properly signed rawtransaction that is ready to be broadcast to the network using the standard `sendrawtransaction` RPC. It doesnt automatically do this to allow the GUI to have a confirmation step with all the details before doing an irrevocable CC contract transaction.

`faucetfund` allows anybody to add funds to the faucet

`faucetget` allows anybody to get 0.1 coins from the faucet as long as they dont violate the rules.

And we come to what it is all about. The rules of the faucet. Initially it was much less strict and that allowed it to be drained slowly, but automatically and it prevented most from being able to use the faucet. 

To make it much harder to leech, it was made so each `faucetget` returned only 0.1 coins (down from 1.0) so it was worth 90% less. It was also made so that it had to be to a fresh address with less than 3 transactions. Finally each txid was constrained to start and end with 00! This is a cool trick to force usage of precious CPU time (20 to 60 seconds depending on system) to generate a valid txid. Like PoW mining for the txid and I expect other CC contracts to use a similar mechanism if they want to rate limit usage.

Combined, it became such a pain to get 0.1 coins, the faucet leeching problem was solved. It might not seem like too much trouble to change an address to get another 0.1 coins, but the way things are setup you need to launch the `komodod -pubkey=<your pubkey>` to change the pubkey that is active for a node. That means to change the pubkey being used, the `komodod` needs to be restarted and this creates a lot of issues for any automation trying to do this. Combined with the PoW required, only when 0.1 coins becomes worth a significant effort will faucet leeching return. In that case, the PoW requirement can be increased and coin amount decreased, likely with a faucet2 CC contract as I dont expect many such variations to be needed.





## Chapter 7 - Rewards Example

## Chapter 8 - Assets Example

## Chapter 9 - Dice Example

## Chapter 10 - Lotto Example

## Chapter 11 - Channels Example



## Chapter 12 - Limitless Possibilities
As can be seen, CC contracts can do a wide range of things and since they are Turing complete, we know that this is true. However, what is more important is the added security gained from using a UTXO based system. While in some ways it is more complex to have to deal with UTXO, as can be seen by the above examples, it is either solved and made invisible at the RPC level, or actually used as part of the solution.

Being UTXO based, automatically builds in a rate limit to how many tx per block a specific CC contract can do. The state advancing by one transaction at a time is another means that rate limits. Since more UTXO can be made available to increase capacity, it actually offers a way for managing load.

I believe I have made one of the first operational UTXO smart contracts, CC or otherwise and hope that there will be many more developers joining forces to create more foundational CC contracts. Feel free to contact me for feedback on the type of CC contract you want to make. I have not documented all my notes and it could well be I already sort of know how to implement what your want your CC contract to do. Just only so many I can actually make time to code and debug.

Our testing cycle went a lot faster than expected as the bugs found were few and far between. Considering the scope of the assets CC and the realtime response aspects of dice CC, this was quite unexpected. I can only attribute it to the fact that CC validation is just the final validation on top of all the standard bitcoin protocol validations. Not having to worry about double spends is sure a nice luxury, though dont get too complacent about chain rewrites! It is possible to wait for information to be divulged and then reorg the chain to take advantage of this knowledge in a chain which is rewound.

Yes, blockchains are complicated.

## Chapter 13 - Different Languages
The current codebase is integrated into the komodod codebase, which is C/C++. However, it is possible to use different languages and integrate into the C/C++ as zcash has shown by using the rust language for some parts of the `zcashd`.

I think any language that is compiled and can create a linkable library while being able to call and be called by C/C++ functions can be used. If you are able to make such a language binding for a simple CC contract like faucet, this will be good for a 777 KMD bounty. Of course, you need to be the first to submit a properly working pull request.


## Chapter 14 - Runtime Bindings
Once build time linking works, then it is one step away from being able to do runtime linking, ie. dynamically linked libraries. There will be some work required to prevent duplication of eval codes and making sure it is a valid version of the CC contract plugin, but these are issues that have been solved before and I dont see any reason they cant be solved for CC contracts.

This would open up the door for quite an interesting ecosystem of CC plugins that blockchains can subscribe to.

## Chapter 15 - RPC based dapps
Ultimately, I expect there to be so many new RPC calls (one set from each CC contract), that virtually any dapp can be made with RPC calls. We are just at the beginning now, but it is just a matter of time when we get there.

For now, we just need to keep listening to what the market wants as far as dapps go. Then make a new CC contract that enables doing as many of those as possible.

Repeat...

Imagine the scope that will exist after a year or two of continuous new CC contracts being created, along with all the RPC based dapps. I have seen some automatic GUI generators and it could be that for most cases, there can be a special GUI that not only create the dapp's GUI, but also all the RPC calls that are needed to make it work the way it is customized.

This codebase and tools in between the GUI and the RPC level will be a very good area for new initiatives.

---------

## Conclusion
I hope this document has helped you understand what a Komodo utxo based CC contract is and how it is different from the other smart contracts. If you are now able to dive into the cc directory and start making your own CC contract, then I am very happy!