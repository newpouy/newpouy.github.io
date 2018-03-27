---
layout: post
title:  "Naivecoin Korean Translate Version #6"
date:   2018-03-25 21:18:48 +0900
categories: jekyll update
---

navie코인 번역
원문: [lhartikk.github.io](https://lhartikk.github.io)

### #6: Wallet UI and blockchain explorer

## 소개
우리는 이번 챕터에서 지갑에 UI를 붙이는 작업과 블럭체인 탐색기를 만드는 작업을 할거에요. 이미 우리의 블럭체인 노드는 HTTP 엔드포인트를 가지고 있으므로 웹페이지로 시각화하는 작업을 할 수 있어요ㅣ
To achieve all this, we must add some additional endpoints and logic your node, for instance:
먼저 몇 개의 엔드포인트를 추가하도록 하죠
- Query information about blocks and transactions 블럭과 트랜잭션 조회
- Query information about a specific address 특정한 address 정보 조회

## New endpoints
Let’s add an endpoint from which the user can query a specific block, if the hash is known.
사용가 이미 해시값이 알려진 특정 블럭을 조회하는 엔드포인트 를 만들어요.
{% highlight js %}
app.get('/block/:hash', (req, res) => {
        const block = _.find(getBlockchain(), {'hash' : req.params.hash});
        res.send(block);
    });
{% endhighlight %}

트랜잭션에도 같은 기능을 하는 게 필요하죠.
{% highlight js %}
app.get('/transaction/:id', (req, res) => {
    const tx = _(getBlockchain())
        .map((blocks) => blocks.data)
        .flatten()
        .find({'id': req.params.id});
    res.send(tx);
});
{% endhighlight %}

특정 주소의 노드가 가진 정보도 보여줄 거에요. 일단 '소진되지 않은 아웃풋' 목록을 리턴하는 걸 만들죠. 이를 이용해서 해당 주소의 노드 가진 잔고를 계산할 수 있어요.
{% highlight js %}
app.get('/address/:address', (req, res) => {
        const unspentTxOuts: UnspentTxOut[] =
            _.filter(getUnspentTxOuts(), (uTxO) => uTxO.address === req.params.address)
        res.send({'unspentTxOuts': unspentTxOuts});
    });
{% endhighlight %}

'소진된 아웃풋'에 관한 정보도 필요해요. 해당 노드의 변경이력을 확인 할 수 있죠.

## Frontend techs
웹화면을 위해 Vue.js를 사용할게요. 하지만 이 튜토리얼은 프론트엔드 개발에 초점을 맞추고 있지 않기 때문에 자세한 설명은 생략한다. 코드는 확인할 수 있을 거에요.

## 블럭체인 탐색기 Blockchain explorer
블럭체인 탐색기는 블럭체인의 현 상태를 명시적으로 확인할 수있는 웹사이트에요. 특정 주소의 잔액 조회, 트랜잭션의 유효성 확인 등을 할 수 있죠.
우리는 노드에 http request를 보내고 응답을 받는 용도로 사용할 거에요. 블럭체인을 변경시키는 것은 블럭체인 탐색기Blockchain explorer에 포함시키지 않을 거에요. 블럭체인이 하는 일은 노드의 정보를 의미있게 보여주는 일, 그것이 전부죠.
![explorer_ui]({{ "/assets/explorer_ui.png" | absolute_url}})

## Wallet UI
wallet의 UI도 웹사이트가 될 거에요. 사용자는 웹사이트에서 코인을 보내고 잔고를 확인하고 트랜잭션 풀도 볼 수 있어요.
![wallet_ui]({{ "/assets/wallet_ui.png" | absolute_url}})
