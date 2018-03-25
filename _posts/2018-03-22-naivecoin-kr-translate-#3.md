---
layout: post
title:  "Naivecoin Korean Translate Version ##3"
date:   2018-03-20 21:18:48 +0900
categories: jekyll update
---

navie코인 번역
원문: [lhartikk.github.io](https://lhartikk.github.io)

### #2: 트랜잭션

## 소개
이번 챕터에서 우리는 트랜잭션이라는 개념을 다룰거에요. 트랜잭션이 도입됨으로써 우리가 만드는 체인이 일반적인 블록체인에서 '암호화폐'로 탈바꿈하게 되죠. 화폐를 소유하고 있다는 증거를 보여줄 수 있으면 그들을 보낼 수 있는 거죠.

이를 위해서는 많은 개념이 필요해요. publick-key 암호화, 서명, 트랜잭션인풋과 아웃풋 등등.

## 퍼블릭키 암호화와 서명
퍼블릭키 암호화를 위해서는 키 한 쌍이 필요해요. secret key와 public key. public key는 secret key로부터 만들어지지만 그 반대는 불가능해요. 이름에서 알 수 있듯이 public key는 누구에게나 공유될 수 있어요.

메시지는 private key를 이용하여 signature를 만들 수 있고, 이 signature와 public key를 가진 누구나 그 signature가 특정 private key에 의해 만들어졌다는 걸 증명할 수 있죠.
![Digital_signatures]({{ "/assets/Digital_signatures.png" | absolute_url

퍼블릭 키 암호화를 위해 elliptic이라는 라이브러리에 포함된 다음의 두 개의 암호화 함수를 사용하도록 하죠.
- Hash function (SHA256) for 작업 증명과 블록의 고유성 확보
- Public-key cryptography (ECDSA) for 트랜잭션

## Private key와 public key(in ECDSA)
유효한 private key는 32byte의 문자열로 이루어져요.
{% highlight js %}
19f128debc1b9122da0635954488b208b829879cf13b3d6cac5d1260c0fd967c
{% endhighlight %}
유효한 public key는 '04'가 붙은 64byte 문자열이구요.
{% highlight js %}
04bfcab8722991ae774db48f934ca79cfb7dd991229153b9f732ba5334aafcd8e7266e47076996b55a14bf9913ee3145ce0cfc1372ada8ada74bd287450313534a
{% endhighlight %}
public key는 코인을 수신하는 측에서 사용될 거에요.

## 트랜잭션 개괄
코드를 보기 전에 암호화폐에서 트랜잭션의 인풋과 아웃풋이 무엇인지 그림을 통해 이해해 보죠. 아웃풋은 코인을 어디로 보낼지에 대한 정보에요. 인풋은 보낸(보내진)코인을 실제로 소유했었던 발송자에대한 증거에요.
![transactions]({{ "/assets/transactions.png" | absolute_url}})

## transaction outputs
트랜잭션 아웃풋(txOut)은 주소와 코인의 양으로 구성되요. 주소는 ECDSA 퍼블릭키 값이에요. 이는 유저가 특정 코인에 접근하기 위해서는 해당 퍼블릭키에 대응하는 private key를 가지고 있어야 한다는 거죠.
{% highlight js %}
class TxOut {
    public address: string;
    public amount: number;

    constructor(address: string, amount: number) {
        this.address = address;
        this.amount = amount;
    }
}
{% endhighlight %}

## transaction inputs
Transaction inputs (txIn) provide the information “where” the coins are coming from. Each txIn refer to an earlier output, from which the coins are ‘unlocked’, with the signature. These unlocked coins are now ‘available’ for the txOuts. The signature gives proof that only the user, that has the private-key of the referred public-key ( =address) could have created the transaction.
트랜잭션 인풋(txIn)은 코인이 어디로부터 왔는지에 대한 정보를 제공해요. 각각의 txIn은 이전의 output을 참조하고 서명을 통해 unlocked되요. 서명의 역할은 오직 public key과 한 쌍인 private key를 가진 사용자만이 트랜잭션을 만들수 있음을 보증하죠.
{% highlight js %}
class TxIn {
    public txOutId: string;
    public txOutIndex: number;
    public signature: string;
}
{% endhighlight %}
트랜잭션 인풋txIn은 오직 private key로부터 생성된 signature값만을 가진다는 걸 주목해주세요. 절대 private key 그 자체를 가지지 않아요. 블록체인은 퍼블릭키public-key와 서명signature만을 가져요.

결과적으로 txIn은 코인을 풀고unlock txOut은 코인을 잠그는relock 역할을 해요.
![transactions2]({{ "/assets/transactions2.png" | absolute_url}})

## 트랜잭션 구성
트랜잭션은 아주 간단한 구조를 가져요.
{% highlight js %}
class Transaction {
    public id: string;
    public txIns: TxIn[];
    public txOuts: TxOut[];
}
{% endhighlight %}

## Transaction id
트랜잭션 아이디는 트랜잭션의 컨텐트로부터 계산된 해시값이에요. txIds의 signature는 해시에 포함되지 않지만, 그건 나중에 트랜잭션에 추가될 거에요.
{% highlight js %}
const getTransactionId = (transaction: Transaction): string => {
    const txInContent: string = transaction.txIns
        .map((txIn: TxIn) => txIn.txOutId + txIn.txOutIndex)
        .reduce((a, b) => a + b, '');

    const txOutContent: string = transaction.txOuts
        .map((txOut: TxOut) => txOut.address + txOut.amount)
        .reduce((a, b) => a + b, '');

    return CryptoJS.SHA256(txInContent + txOutContent).toString();
};
{% endhighlight %}

## Transaction signatures
서명된 이후의 트랜잭션 내용이 수정될 수 없다는 사실은 매우 중요해요. 트랜잭션이 public해졌기 때문에 누구나 그 트랜잭션에 접근할 수 있죠. 아직 블록체인에 chaining되지 않은 사람까지도.

트랜잭션 인풋에 서명할 때 오직 txId만이 서명될 거에요. 만약 트랜잭션의 어느 부분이라도 변경되면 txId값도 변경되고, 이는 해당 트랜잭션과 서명을 무효화하죠.
{% highlight js %}
const signTxIn = (transaction: Transaction, txInIndex: number,
                  privateKey: string, aUnspentTxOuts: UnspentTxOut[]): string => {
    const txIn: TxIn = transaction.txIns[txInIndex];
    const dataToSign = transaction.id;
    const referencedUnspentTxOut: UnspentTxOut = findUnspentTxOut(txIn.txOutId, txIn.txOutIndex, aUnspentTxOuts);
    const referencedAddress = referencedUnspentTxOut.address;
    const key = ec.keyFromPrivate(privateKey, 'hex');
    const signature: string = toHexString(key.sign(dataToSign).toDER());
    return signature;
};
{% endhighlight %}
누군가가 트랜잭션에 변경을 시도하면 무슨 일이 일어나는지를 살펴보죠.

1. 한 노드의 해커가 "10코인을 AAA주소에서 BBB주소로 보내"라는 내용을 가진 트랜잭션을 0x555라는 txId값과 함께 받았어요.
2. 해커는 수신 주소를 BBB에서 CCC로 변경하고 이를 네트워크상에 포워딩해요. 이제 트랜잭션의 내용은 "10코인을 AAA주소에서 CCC주소로 보내"로 바뀌었어요.
3. 하지만 수신 주소가 변경되면 기존 txId는 더 이상 유효하지 않아요. 유효한 txId는 0x567.. 같은 게 되겠죠.
4. 서명signature 역시 기존 txId를 기반으로 만들어졌기 때문에 더 이상 유효하지 않게 되요.
5. 따라서 변조된 트랜잭션은 다른 노드들에서 받아들여질 수 없죠.

## Unspent transaction outputs
트랜잭션 인풋은 항상 '보내지지지 않은 트랜잭션 아웃풋(Unspent transaction outputs, uTxO)'을 참조해야해요. 결국, 우리가 블록체인에서 코인을 갖는다는 것은, 보내지지지 않은 트랜잭션 아웃풋(Unspent transaction outputs, uTxO)의 목록을 가진다는 거죠. 이 아웃풋들의 퍼블릭키는 private key와 대응하구요.

트랜잭션 유효성 검증이란 측면에서 uTxO는 중요해요. uTxO는 현재 최신 상태의 블록체인에서 비롯되어야 하기 때문에 우리는 이 업데이트를 구현할 거에요.

uTxO의 데이터 구조는 아래와 같죠.
{% highlight js %}
class UnspentTxOut {
    public readonly txOutId: string;
    public readonly txOutIndex: number;
    public readonly address: string;
    public readonly amount: number;

    constructor(txOutId: string, txOutIndex: number, address: string, amount: number) {
        this.txOutId = txOutId;
        this.txOutIndex = txOutIndex;
        this.address = address;
        this.amount = amount;
    }
}
{% endhighlight %}
uTxO의 목록의 자료구조는 단순한 배열이에요.
{% highlight js %}
let unspentTxOuts: UnspentTxOut[] = [];
{% endhighlight %}

## Updating unspent transaction outputs
새로운 블록이 더해질 때마다 uTxO(Unspent transaction outputs)을 업데이트해야해요.
This is because the new transactions will spend some of the existing transaction outputs and introduce new unspent outputs.
새로운 트랜잭션은 기존의 트랜잭션 아웃풋 목록에 영향을 주고 새로운 아웃풋을 발생시키기 때문이죠.
To handle this, we will first retrieve all new unspent transaction outputs (newUnspentTxOuts) from the new block:
이를 위해 새로 생성된 블록으로부터 new unspent transaction outputs을 순회하는 작업이 이루어질 거에요. 코드를 보죠.
{% highlight js %}
const newUnspentTxOuts: UnspentTxOut[] = newTransactions
    .map((t) => {
        return t.txOuts.map((txOut, index) => new UnspentTxOut(t.id, index, txOut.address, txOut.amount));
    })
    .reduce((a, b) => a.concat(b), []);
{% endhighlight %}

We will also need to know which transaction outputs are consumed by the new transactions of the block (consumedTxOuts). This will be solved by examining the inputs of the new transactions:
블록에서 이미 소비된 트랜잭션 아웃풋들에 대해서도 알아야 해요. 새 트랜잭션의 인풋을 검사하면 알 수 있어요.
{% highlight js %}
const consumedTxOuts: UnspentTxOut[] = newTransactions
       .map((t) => t.txIns)
       .reduce((a, b) => a.concat(b), [])
       .map((txIn) => new UnspentTxOut(txIn.txOutId, txIn.txOutIndex, '', 0));
{% endhighlight %}

Finally, we can generate the new unspent transaction outputs by removing the consumedTxOuts and adding the newUnspentTxOuts to our existing transaction outputs.
이미 소비된 아웃풋을 제거하고 이제 새로은 트랜잭션 아웃풋을 만들수 있게 되었어요.
{% highlight js %}
const resultingUnspentTxOuts = aUnspentTxOuts
        .filter(((uTxO) => !findUnspentTxOut(uTxO.txOutId, uTxO.txOutIndex, consumedTxOuts)))
        .concat(newUnspentTxOuts);
{% endhighlight %}

The described code and functionality is contained in the updateUnspentTxOuts method. It should be noted that this method is called only after the transactions in the block (and the block itself) has been validated.
updateUnspentTxOuts함수가 이 작업들을 총괄 수행할 거에요. 그리고 이 함수는 반드시 트랜잭션의 유효성이 검증된 이후에 수행되어야 해요.

## Transactions validation 트랜잭션 유효성 검증
우리는 마침내 무엇이 트랜잭션을 유효하게 만드는지 와꾸를 잡을수 있게 되었어요.

# Correct transaction structure 정확한 트랜잭셔 구성
트랜잭션은 정의된 방식을 따라야만 해요.
{% highlight js %}
const isValidTransactionStructure = (transaction: Transaction) => {
        if (typeof transaction.id !== 'string') {
            console.log('transactionId missing');
            return false;
        }
        ...
       //check also the other members of class
    }
{% endhighlight %}

# Valid transaction 유효한 트랜잭션 ID
The id in the transaction must be correctly calculated.
ID도 확인해야 하고.
{% highlight js %}
if (getTransactionId(transaction) !== transaction.id) {
        console.log('invalid tx id: ' + transaction.id);
        return false;
    }
{% endhighlight %}

# Valid txIns
The signatures in the txIns must be valid and the referenced outputs must have not been spent.
txIns의 서명도 사용되지 않은 아웃풋을 잘 참조하고 있는지 확인해야 해요.
{% highlight js %}
const validateTxIn = (txIn: TxIn, transaction: Transaction, aUnspentTxOuts: UnspentTxOut[]): boolean => {
    const referencedUTxOut: UnspentTxOut =
        aUnspentTxOuts.find((uTxO) => uTxO.txOutId === txIn.txOutId && uTxO.txOutId === txIn.txOutId);
    if (referencedUTxOut == null) {
        console.log('referenced txOut not found: ' + JSON.stringify(txIn));
        return false;
    }
    const address = referencedUTxOut.address;

    const key = ec.keyFromPublic(address, 'hex');
    return key.verify(transaction.id, txIn.signature);
};
{% endhighlight %}

# Valid txOut values
The sums of the values specified in the outputs must be equal to the sums of the values specified in the inputs. If you refer to an output that contains 50 coins, the sum of the values in the new outputs must also be 50 coins.
아웃풋의 코인 갯수와 인풋의 코인 갯수도 같아야만 해요. 아웃풋이 50개라면 인풋도 50개.
{% highlight js %}
const totalTxInValues: number = transaction.txIns
    .map((txIn) => getTxInAmount(txIn, aUnspentTxOuts))
    .reduce((a, b) => (a + b), 0);

const totalTxOutValues: number = transaction.txOuts
    .map((txOut) => txOut.amount)
    .reduce((a, b) => (a + b), 0);

if (totalTxOutValues !== totalTxInValues) {
    console.log('totalTxOutValues !== totalTxInValues in tx: ' + transaction.id);
    return false;
}
{% endhighlight %}

## Coinbase transaction
Transaction inputs must always refer to unspent transaction outputs, but from where does the initial coins come in to the blockchain? To solve this, a special type of transaction is introduced: coinbase transaction
트랜잭션 인풋은 항상 unspent transaction outputs을 참조해야만 해요. 하지만 최초의 코인은 어디를 참조해야 할까요? 이를 해결하기 위해 coinbase transaction이 필요해요.

The coinbase transaction contains only an output, but no inputs. This means that a coinbase transaction adds new coins to circulation. We specify the amount of the coinbase output to be 50 coins.
코인베이스 트랜잭션(coinbase transaction)은 오직 아웃풋만 포함해요. 인풋은 없죠. 코인베이스 트랜잭션이 새로운 코인을 돌게 만드는 펌프?같은 역할을 할거에요. 코인베이스 아풋풋의 양을 50코인으로 정하죠.
{% highlight js %}
const COINBASE_AMOUNT: number = 50;
{% endhighlight %}

The coinbase transaction is always the first transaction in the block and it is included by the miner of the block. The coinbase reward acts as an incentive for the miners: if you find the block, you are able to collect 50 coins.
코인베이스 트랜잭션은 당연히 블럭의 첫 트랜잭션이고 블럭 채굴자 정보를 포함하죠. 블락을 발견한 댓가로 채굴자는 첫 코인베이스 트랜잭션의 아웃풋으로 코인 50개를 받을 거에요.

We will add the block height to input of the coinbase transaction. This is to ensure that each coinbase transaction has a unique txId. Without this rule, for instance, a coinbase transaction stating “give 50 coins to address 0xabc” would always have the same txId.
코인베이스 트랜잭션의 인풋에는 block의 height값을 넣을게요. 이 값이 코인베이스 트랜잭션의 고유한 ID값 역할을 할 거에요. 이 고유값이 없다면 코인 베이스 트랜잭션은 항상 같은 주소에 50코인을 발행하게 되죠.

The validation of the coinbase transaction differs slightly from the validation of a “normal” transaction
코인베이스 트랜잭션의 유효성검증은 다른 트랜잭션이랑은 조금 다를 거에요.
{% highlight js %}
const validateCoinbaseTx = (transaction: Transaction, blockIndex: number): boolean => {
    if (getTransactionId(transaction) !== transaction.id) {
        console.log('invalid coinbase tx id: ' + transaction.id);
        return false;
    }
    if (transaction.txIns.length !== 1) {
        console.log('one txIn must be specified in the coinbase transaction');
        return;
    }
    if (transaction.txIns[0].txOutIndex !== blockIndex) {
        console.log('the txIn index in coinbase tx must be the block height');
        return false;
    }
    if (transaction.txOuts.length !== 1) {
        console.log('invalid number of txOuts in coinbase transaction');
        return false;
    }
    if (transaction.txOuts[0].amount != COINBASE_AMOUNT) {
        console.log('invalid coinbase amount in coinbase transaction');
        return false;
    }
    return true;
};
{% endhighlight %}

## Conclusions
We included the concept of transactions to the blockchain. The basic idea is quite simple: we refer to unspent outputs in transaction inputs and use signatures to show that the unlocking part is valid. We then use outputs to “relock” them to a receiver address.
이 챕터에서 우리는 블럭채인의 트랜잭션에 대해 알아봤어요. 기본개념은 심플해요. 사용되지 않은 트랜잭션 아웃붓과 블럭채인의 잠금해제된 부분이 유효하다는 걸 보여줄거에요.

However, creating transactions is still very difficult. We must manually create the inputs and outputs of the transactions and sign them using our private keys. This will change when we introduce wallets in the next chapter.
하지만 트랜잭션을 실제로 만들어 내는 것은 여전히 어려워요. 인풋과 아웃풋을 직접 만들어야 하고 프라이빗 키를 사용해서 그들에 서명을 해야하죠. 이 과정은 다음 챕터의 wallet을 배우면 달라질 거에요.

There is also no transaction relaying yet: to include a transaction to the blockchain, you must mine it yourself. This is also the reason we did not yet introduce the concept of transaction fee.
아직 연결된 트랜잭션에 대해서도 살펴보지 않았어요. 블럭체인에 트랜잭션을 일으키기 위해서 채굴을 해야만 해요. 이 때 '트랜잭션 fee'라는 개념도 필요해죠. 다음챕터에서 살펴볼게요.

The full code implemented in this chapter can be found here
이번 챕터의 코드는 [여기](https://github.com/lhartikk/naivecoin/tree/chapter3)에 있어요.

[To chapter4](https://newpouy.github.io/jekyll/update/2017/07/12/chapter3.html)
