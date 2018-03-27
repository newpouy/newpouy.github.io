---
layout: post
title:  "Naivecoin Korean Translate Version #4"
date:   2018-03-24 21:18:48 +0900
categories: jekyll update
---

navie코인 번역
원문: [lhartikk.github.io](https://lhartikk.github.io)

### #4: Wallet

## 소개
이번 챕터의 목적은 사용자를 위한 wallet 인터페이스를 만드는 거에요.
엔드유저는
- private key를 가진 지갑(wallet)을 만들 수 있어야 하고
- 지갑에 잔액을 볼 수 있어야 하고
- 코인을 다른 노드로 보낼 수 있어야 해요.

txIns나 txOut같은 것들이 어떻게 작동하는지 엔드유저는 알 필요가 없어요. 비트코인처럼 어떤 노드로 코인을 보내고, 또 코인을 받을 나만의 주소를 가질 수 있으면 되는 거죠.

## private key를 만들고 저장하기

이 튜토리얼에서는 wallet을 만들기 위해 아주 간단한 방법을 사용할 거에요. private key를 암호화하지는 않은 채로 node/wallet/private_key 이 경로에 만들도록 하죠.
{% highlight js %}
const privateKeyLocation = 'node/wallet/private_key';

const generatePrivatekey = (): string => {
    const keyPair = EC.genKeyPair();
    const privateKey = keyPair.getPrivate();
    return privateKey.toString(16);
};

const initWallet = () => {
    //let's not override existing private keys
    if (existsSync(privateKeyLocation)) {
        return;
    }
    const newPrivateKey = generatePrivatekey();

    writeFileSync(privateKeyLocation, newPrivateKey);
    console.log('new wallet with private key created');
};
{% endhighlight %}

이미 살펴봤듯이 private key로부터 public key(=address, 주소의 역할)를 만들어 낼 거에요.
{% highlight js %}
const getPublicFromWallet = (): string => {
    const privateKey = getPrivateFromWallet();
    const key = EC.keyFromPrivate(privateKey, 'hex');
    return key.getPublic().encode('hex');
};
{% endhighlight %}
private key를 암호화 하지 않은 것은 보안상 매우 위험하다는 걸 유념해주세요. 이 과정은 간단히 만드는 데 초점을 두고 있기 때문에 그렇게 하는 것일 뿐이에요. wallet은 오직 하나의 private key만을 가져야 하고 이로부터 pulic하게 접근가능한 address를 가진 지갑을 만들 거에요.

## Wallet balance
이전 챕터에서 배운 걸 기억해 보죠. 당신이 블럭체인 상에서 코인을 가지고 있다는 것은 '쓰이지 않은 트랜잭션 아웃풋(unspent transaction outputs)' 한 꾸러미를 가지고 있다는 것을 의미해요. 이 아웃풋 목록은 당연히 당신만의 private key와 매칭된 public address값을 가지고 있죠.

그럼 '잔고'계산은 참 쉽겠죠잉? 걍 '쓰이지 않은 트랜잭션 아웃풋(unspent transaction outputs)'을 다 더해버리면 되는 거죠.
{% highlight js %}
const getBalance = (address: string, unspentTxOuts: UnspentTxOut[]): number => {
    return _(unspentTxOuts)
        .filter((uTxO: UnspentTxOut) => uTxO.address === address)
        .map((uTxO: UnspentTxOut) => uTxO.amount)
        .sum();
};
{% endhighlight %}
코드가 보여주듯, 잔고조회에 private key는 필요없어요. 이는 누구나 주어진 address의 잔고를 알 수 있다는 거죠.

## 트랜잭션 생성 Generating transactions
코인을 보낼 때 사용자는 트랜잭션 인풋이니 아웃풋이니 알 필요가 없다고 했죠. 그럼 이 사용자 A가 단 한번의 트랜잭션으로 그가 가진 50코인 중 단지 10코인만 B에게 보내고 싶다면 어떻게 해야 할까요?
방법은 10코인은 B에게 보내고 나머지 40코인은 그 자신에게 돌려보내(back)는 거죠. 전체 트랜잭션 아웃풋은 항상 소진(spent)되어야 하고, 만약 부분으로 나누고 싶다면 새로운 아웃풋을 생성함으로 가능해요. 그림을 보시죠.
![tx_generation]({{ "/assets/tx_generation.png" | absolute_url}})

좀 더 복잡한 트랜잭션을 시도해 보죠.
1. User C에겐 코인이 없었어요
2. User C가 3트랜잭션으로 10, 20, 30코인을 차례로 받았어요
3. User C는 55코인을 user D에게 보내길 원해요.
트랜잭션은 어떻게 이루어져야 할까요?

이 경우 3번에 걸친 아웃풋이 모두 사용되어야 해요. 55코인은 D에게 보내고 5는 자신에게 back.
![tx_generation2]({{ "/assets/tx_generation2.png" | absolute_url}})

코드로 확인해 보죠. 먼저 트랜잭션 인풋을 만들어야 해요. 이를 위해 '소진되지 않은 트랜잭션 아웃(unspent transaction outputs)' 목록을 순회하며 우리가 원하는 금액이 될 때까지 반복문을 돌리죠.
{% highlight js %}
const findTxOutsForAmount = (amount: number, myUnspentTxOuts: UnspentTxOut[]) => {
    let currentAmount = 0;
    const includedUnspentTxOuts = [];
    for (const myUnspentTxOut of myUnspentTxOuts) {
        includedUnspentTxOuts.push(myUnspentTxOut);
        currentAmount = currentAmount + myUnspentTxOut.amount;
        if (currentAmount >= amount) {
            const leftOverAmount = currentAmount - amount;
            return {includedUnspentTxOuts, leftOverAmount}
        }
    }
    throw Error('not enough coins to send transaction');
};
{% endhighlight %}
코드에서 보이듯이, leftOverAmount는 나중에 자신에게 다시 back할 금액이에요.
소진되지 않은 트랜잭션 아웃풋을 가진 만큼 트랜책션 txIns를 만들어낼 수 있어요.
{% highlight js %}
const toUnsignedTxIn = (unspentTxOut: UnspentTxOut) => {
    const txIn: TxIn = new TxIn();
    txIn.txOutId = unspentTxOut.txOutId;
    txIn.txOutIndex = unspentTxOut.txOutIndex;
    return txIn;
};
const {includedUnspentTxOuts, leftOverAmount} = findTxOutsForAmount(amount, myUnspentTxouts);
const unsignedTxIns: TxIn[] = includedUnspentTxOuts.map(toUnsignedTxIn);
{% endhighlight %}

다음으로 두 개의 txOuts를 만들어야 해요. 하나는 보낼 것. 다른 하나는 다시 back할 것. 만약 txIns가 보낼 금액과 같다면 leftOverAmount값은 0이므로 back을 위한 트랜잭션은 만들지 않아도 되죠.
{% highlight js %}
const createTxOuts = (receiverAddress:string, myAddress:string, amount, leftOverAmount: number) => {
    const txOut1: TxOut = new TxOut(receiverAddress, amount);
    if (leftOverAmount === 0) {
        return [txOut1]
    } else {
        const leftOverTx = new TxOut(myAddress, leftOverAmount);
        return [txOut1, leftOverTx];
    }
};
{% endhighlight %}

마지막으로 트랜잭션id값을 생성하고 txIns에 서명하면 끝.
{% highlight js %}
const tx: Transaction = new Transaction();
    tx.txIns = unsignedTxIns;
    tx.txOuts = createTxOuts(receiverAddress, myAddress, amount, leftOverAmount);
    tx.id = getTransactionId(tx);

    tx.txIns = tx.txIns.map((txIn: TxIn, index: number) => {
        txIn.signature = signTxIn(tx, index, privateKey, unspentTxOuts);
        return txIn;
    });
{% endhighlight %}


## 지갑 사용하기 Using the wallet
지갑을 사용하기 위해 기능을 넣어 보죠.
{% highlight js %}
app.post('/mineTransaction', (req, res) => {
        const address = req.body.address;
        const amount = req.body.amount;
        const resp = generatenextBlockWithTransaction(address, amount);
        res.send(resp);
    });
{% endhighlight %}

위에서 보듯 사용자는 단지 주소와 코인금액만 제공하면 되요. 블럭체인의 노드가 나머지는 알아서 뿅.

## 결론
우리가 한건 아주 간단한 수준의 트랜잭션 알고리즘을 가진 비암호화된 지갑이에요. 이 알고리즘은 2개의 outputs만을 만들어낼 뿐이지만, 실제 현실의 블럭체인은 다르죠.
당신은 이 지갑에서 50코인의 input을 가지고 5,15,20의 output을 만들 수 있어요. 하지만 이것도 /mineRawBlock 인터페이스를 사용해서 조절 가능하죠.

블럭체인에서 직접 채굴을 함으로써 원하는 트랜잭션을 발생시킬 수도 있어요. 발생시킨 트랜잭션을 블럭체인에 연결시키는 것을 다음 챕터에서 다루게 될 거에요.
이번 챕터의 코드는 [여기](https://github.com/lhartikk/naivecoin/tree/chapter4)에 있어요.

[To chapter5](https://newpouy.github.io/jekyll/update/2017/07/12/chapter5.html)
