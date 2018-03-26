---
layout: post
title:  "Naivecoin Korean Translate Version #5"
date:   2018-03-24 21:18:48 +0900
categories: jekyll update
---

navie코인 번역
원문: [lhartikk.github.io](https://lhartikk.github.io)

### #5: Transaction relaying

## 소개
이 챕터에서 우리는 트랜잭션 릴레이를 구현할 거에요. 트랜잭션 릴레이란 아직 블럭체인에 포함되지 않은 트랜잭션들을 포함시키는 작업을 의미하죠. 비트코인에서는 아직 블럭체인에 포함되지 않은 트랜젝션을 unconfirmed transaction이라고 불러요. 누군가 블럭체인에 트랜잭션을 포함시키고자 할 때(코인을 누군가에게 보낸다던가), 그는 트랜젝션을 네트워크에 발송broadcast하고, 다른 노드들이 블럭체인에 그 트랜잭션을 포함시키길 기대하죠.
이 기능은 암호화폐에서 매우 중요해요. 왜냐하면 당신이 어떤 트랜잭션을 블럭체인에 포함시키기 위해 반드시 어떤 블럭을 직접 채굴해야만 하는 건 아니거든요.
결국 노드들은 두가지 다른 타입의 데이터를 공유하게 되죠.
- 블럭체인의 현 상태(블럭체인에 이미 포함되어 있는 블럭과 트랜젝션들)
- unconfirmed transactions(아직 블럭체인에 포함되지 않은 트랜젝션들)

## Transaction pool
우리는 트랜젝션 풀(transaction pool)에 아직 블럭체인에 포함되지 않은 트랜젝션들을 저장할 거에요. 비트코인에는 mempool이라는게 있죠. 트랜젝션 풀은 아직 블럭체인에 포함되지 않은 모든 트랜젝션들을 저장할 하나의 저장소역할을 할 거에요. 이를 위해 배열을 쓰도록 하죠.
{% highlight js %}
let transactionPool: Transaction[] = [];
{% endhighlight %}

새로운 HTTP 인터페이스가 하나 더 필요해요. POST타입의 /sendTransaction. 이를 이용해 wallet에 트랜젝션을 만들고 이 트랜젝션을 트랜젝션풀에 넣는 기능을 더할 거에요. 블럭체인에 새 트랜젝션을 포함시키고자 할 때, 이 인터페이스를 우선적으로 사용할 거에요.
{% highlight js %}
app.post('/sendTransaction', (req, res) => {
        ...
    })
{% endhighlight %}

We create the transaction just like we did in chapter4. We just add the created transaction to the pool instead of instantly trying to mine a block:
챕터4에서 했듯이 트랜잭션을 만들고, 다만 당장 블럭을 블럭체인에 추가하는 대신, 트랜젝션 풀에 넣도록 하죠.
{% highlight js %}
const sendTransaction = (address: string, amount: number): Transaction => {
    const tx: Transaction = createTransaction(address, amount, getPrivateFromWallet(), getUnspentTxOuts(), getTransactionPool());
    addToTransactionPool(tx, getUnspentTxOuts());
    return tx;
};
{% endhighlight %}

## Broadcasting
아직 블럭체인에 포함되지 않은 unconfirmed transactions들에게 결국 그들도 전체 블럭체인 네트워크에 퍼질 것이고 누군가의 노드에도 추가될 것이란 것은 매우 중요하죠. 이를 위해 다음과 같은 룰이 필요해요.
- 한 노드가 unconfirmed transactions를 받으면, 해당 노드는 자신의 전체 트랜젝션 풀을 모든 동료peer 노드들에 발송해야 한다.
- A 노드가 B 노드와 연결되면, A는 B의 트랜잭션풀에 질의query를 해야한다.
We will add two new MessageTypes to serve this purpose: QUERY_TRANSACTION_POOL and RESPONSE_TRANSACTION_POOL. The MessageType enum will now look now like this:
이 룰을 위해 두 가지 새로운 메시지타입MessageType을 추가 할게요. QUERY_TRANSACTION_POOL and RESPONSE_TRANSACTION_POOL. 코드로 이렇게 표현될 거에요.
{% highlight js %}
enum MessageType {
    QUERY_LATEST = 0,
    QUERY_ALL = 1,
    RESPONSE_BLOCKCHAIN = 2,
    QUERY_TRANSACTION_POOL = 3,
    RESPONSE_TRANSACTION_POOL = 4
}
{% endhighlight %}
The transaction pool messages will be created in the following way:
트랜잭션풀의 메시지는 다음 방식으로 만들어져요.
{% highlight js %}
const responseTransactionPoolMsg = (): Message => ({
    'type': MessageType.RESPONSE_TRANSACTION_POOL,
    'data': JSON.stringify(getTransactionPool())
});

const queryTransactionPoolMsg = (): Message => ({
    'type': MessageType.QUERY_TRANSACTION_POOL,
    'data': null
});
{% endhighlight %}

To implement the described transaction broadcasting logic, we add code to handle the MessageType.RESPONSE_TRANSACTION_POOL message type. Whenever, we receive unconfirmed transactions, we try to add those to our transaction pool. If we manage to add a transaction to our pool, it means that the transaction is valid and our node has not seen the transaction before. In this case we broadcast our own transaction pool to all peers.
트랜젝션 발송 로직을 구현하기 위해, 메시지타입에 RESPONSE_TRANSACTION_POOL를 추가할게요. 우리가 unconfirmed transaction을 받을 때는 항상 이들을 자신의 transaction pool에 더해야 해요. 이 작업이 성공해야만 트랜젝션은 우리의 노드에 유효한 것이 되죠. 그리고 우리 자신의 트랜젝션 풀을 모든 다른 peer들에게 발송하는 거에요.
{% highlight js %}
case MessageType.RESPONSE_TRANSACTION_POOL:
    const receivedTransactions: Transaction[] = JSONToObject<Transaction[]>(message.data);
    receivedTransactions.forEach((transaction: Transaction) => {
        try {
            handleReceivedTransaction(transaction);
            //if no error is thrown, transaction was indeed added to the pool
            //let's broadcast transaction pool
            broadCastTransactionPool();
        } catch (e) {
            //unconfirmed transaction not valid (we probably already have it in our pool)
        }
    });
{% endhighlight %}

## 받은 트랜젝션의 유효성 검사 Validating received unconfirmed transactions
peer들이 어떤 트랜잭션을 보내건 간에, 우리는 그 트랜젝션을 풀에 더하기 전에 유효성 검사를 해야만 하겠죠. 이미 다룬 모든 트랜젝션 검사 룰이 적용되요. 트랜젝션의 포맷, 인풋과 아웃풋, 서명 등등.
In addition to the existing rules, we add a new rule: a transaction cannot be added to the pool if any of the transaction inputs are already found in the existing transaction pool. This new rule is embodied in the following code:
그리고 새로운 룰 하나가 더 필요해요. '만약 추가될 트랜젝션의 어느 input이라도 기존 트랜젝션풀에 있는 것과 일치한다면 해당 트랜젝션은 추가될 수 없다' 다음 코드가 이를 위한 거에요.
{% highlight js %}
const isValidTxForPool = (tx: Transaction, aTtransactionPool: Transaction[]): boolean => {
    const txPoolIns: TxIn[] = getTxPoolIns(aTtransactionPool);

    const containsTxIn = (txIns: TxIn[], txIn: TxIn) => {
        return _.find(txPoolIns, (txPoolIn => {
            return txIn.txOutIndex === txPoolIn.txOutIndex && txIn.txOutId === txPoolIn.txOutId;
        }))
    };

    for (const txIn of tx.txIns) {
        if (containsTxIn(txPoolIns, txIn)) {
            console.log('txIn already found in the txPool');
            return false;
        }
    }
    return true;
};
{% endhighlight %}

트랜젝션풀에서 트랜젝션을 제거할 방법은 없어요. 하지만 트랜젝션 풀은 새로운 블럭이 발견되면 언제나 새롭게 업데이트 되어야 해요.

## From transaction pool to blockchain
다음으로 우리는 트랜젝션 풀에 들어온 트랜젝션을 블럭체인 상의 블럭으로 만드는 작업을 할 거에요. 그리 어렵지 않아요. 노드가 블럭을 채굴하는 작업을 시작할 때 트랜젝션 풀로부터 트랜젝션을 받아 새로운 예비블럭에 추가해 담는 거죠.
{% highlight js %}
const generateNextBlock = () => {
    const coinbaseTx: Transaction = getCoinbaseTransaction(getPublicFromWallet(), getLatestBlock().index + 1);
    const blockData: Transaction[] = [coinbaseTx].concat(getTransactionPool());
    return generateRawNextBlock(blockData);
};
{% endhighlight %}
이미 유효성 검사는 수행했으므로 필요없구요.

## Updating the transaction pool
트랜잭션과 함께 새 블럭이 블록체인에 추가되었으므로, 기존이 트랜젝션 풀은 새로고침될 필요가 있죠. 어떤 새로운 블럭이 풀 안의 어떤 트랜잭션을 무효하게 만들 트랜젝션을 포함할 수도 있어요. 예를 들면,
- 풀에 이미 존재하던 트랜젝션이 다시 추가되었다(자신에 의해서건 다른 노드에 의해서건)
- 어떤 트랜젝션에서는 소진되지 않은 트랜젝션 아웃풋이 다른 트랜젝션에서는 소진되어 있는 경우
트랜젝션은 다음 코드에 의해 업데이트될 거에요.
{% highlight js %}
const updateTransactionPool = (unspentTxOuts: UnspentTxOut[]) => {
    const invalidTxs = [];
    for (const tx of transactionPool) {
        for (const txIn of tx.txIns) {
            if (!hasTxIn(txIn, unspentTxOuts)) {
                invalidTxs.push(tx);
                break;
            }
        }
    }
    if (invalidTxs.length > 0) {
        console.log('removing the following transactions from txPool: %s', JSON.stringify(invalidTxs));
        transactionPool = _.without(transactionPool, ...invalidTxs)
    }
};
{% endhighlight %}

As it can be seen, we need to know only the current unspent transaction outputs to make the decision if a transaction should be removed from the pool.
코드가 보여주듯이, 어떤 트랜젝션이 제거되어야하는지 결정하기 위해 현재의 소진되지 않은 트랜젝션 아웃풋을 검토하면 돼요.

## 결론
We can now include transactions to the blockchain without actually having to mine the blocks themselves. There is however no incentive for the nodes to include a received transaction to the block as we did not implement the concept of transaction fees.
우리는 블럭을 채굴하는 과정 없이 블럭체인에 트랜젝션을 포함시켰어요. 하지만 개별 노드들이 트랜젝션을 자신의 블럭체인에 포함시키는 데에 따른 어떤 보상도 없죠. 트랜젝션 수수료transaction fees를 구현하지 않았기 때문이에요.

다음 챕터에서 우리는 wallet을 위한 UI를 만들고 blockchain explorer를 구현할 거에요.

이번 챕터의 코드는 [여기](https://github.com/lhartikk/naivecoin/tree/chapter4)에 있어요.

[To chapter6](https://newpouy.github.io/jekyll/update/2017/07/12/chapter6.html)
