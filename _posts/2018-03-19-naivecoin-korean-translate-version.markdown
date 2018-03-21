---
layout: post
title:  "Naivecoin Korean Translate Version"
date:   2018-03-19 21:18:48 +0900
categories: jekyll update
---

navie코인 번역
원문: [lhartikk.github.io](https://lhartikk.github.io)

## #1: 최소한의 블록체인

# 소개

블록체인의 기본 컨셉은 아주 간단해요. 공통의 기록을 가진 장부를 모두가 갖는 것이죠. 이 챕터에서 우리는 블록체인의 장난감버전을 만들어 볼거에요. 마지막 챕터까지 해낸다면 우리는 다음 과 같은 블록체인의 기본 기능들을 구현할 수 있을 겁니다.
- 블록과 블록체인
- 기존 데이터에 새로운 블록 더하기
- 노드(node) 간 통신과 싱크 맞추기
- 간단한 HTTP를 통한 노드 제어

이 챕터에서 구현할 코드는 여기서 확인하세요.

# 블록의 구조

블록의 구조를 정의하는 걸로 시작해 보죠. 아주 필수적인 것만 포함시킬 거에요.
- Index: 블록체인에서 해당 블록의 순서
- Data: 블록에 담긴 데이터
- Timestamp: 타임스템프
- Hash: sha256알고리즘으로 만들어진 해쉬값
- previousHash: 이전 블록의 해쉬값. 앞의 블록에서 이미 정해지게 되죠.

![blockchain]({{ "/assets/blockchain.png" | absolute_url }})

블록의 구조를 코드로 작성해 보면 다음과 같아요.
{% highlight ruby %}
class Block {

    public index: number;
    public hash: string;
    public previousHash: string;
    public timestamp: number;
    public data: string;

    constructor(index: number, hash: string, previousHash: string, timestamp: number, data: string) {
        this.index = index;
        this.previousHash = previousHash;
        this.timestamp = timestamp;
        this.data = data;
        this.hash = hash;
    }
}
{% endhighlight %}

# Block hash

블록해쉬는 블록에서 가장 중요한 것 중 하나에요. 해쉬값은 블록이 가진 데이터를 기반으로 계산되죠. 즉 블록의 어떤 데이터라도 변하면 기존의 해쉬값은 무의미해지는 거죠. 또 해쉬값은 해당 블록의 고유한 식별자로도 기능해요. 그래서 같은 인덱스를 가진 블록은 있을 수 있지만 해쉬값은 모두 달라요.

우리는 다음 코드로 해쉬값을 계산할 거에요.
{% highlight js %}
const calculateHash = (index: number, previousHash: string, timestamp: number, data: string): string =>
    CryptoJS.SHA256(index + previousHash + timestamp + data).toString();
{% endhighlight %}

아직 채굴(mining)과정과 작업증명(POW, proof-of-work)과정을 거치지 않았다는 걸 기억해 주세요. 우리는 이 코드로 생성된 해쉬값을 블록의 고유성을 유지하고 다음 블록의 ‘이전 블록 해쉬값(previousHas)’으로 사용할 거에요.

hash와 previousHash의 연결성은 아주 중요해요. 이를 통해 블럭의 위변조를 막을 수 있거든요.

아래 그림으로 이해해 보죠. 만약 44블록의 데이터가 변하면 이어지는 모든 블록(45,46,47...)의 데이터가 변해야만 해요. 왜냐하면 블록의 hash는 이전 블록의 hash, 즉 previousHash를 기반으로 만들어지기 때문이죠.

![Blockchain_integrity]({{ "/assets/Blockchain_integrity.png" | absolute_url}})

이에 대해서는 작업증명(POW)를 소개할 때 다시 한번 강조할 거에요. 블록체인의 규모가 커질수록 특정 블록의 해쉬값을 바꾸기는 어렵죠. 왜냐하면 이어지는 모든 블록의 데이터를 수정해야하기 때문이에요.

# 최초의 블록
‘최초의 블록’은 블록체인의 시작이죠. 이 블록만이 previousHash값은 가지지 않아요. 이 블록의 해쉬값을 하드코딩으로 입력하도록 하죠.
{% highlight js %}
const genesisBlock: Block = new Block(
    0, '816534932c2b7154836da6afc367695e6337db8a921823784c14378abed4f7d7', null, 1465154705, 'my genesis block!!'
);
{% endhighlight %}

# 블록 만들기

블록을 만들기 위해서는 이전 블록의 해쉬값을 알아야할 뿐만아니라 나머지 구성요소들도 채워야해요. 다음 코드를 통해 블록의 모든 데이터를 채울 수 있어요.
{% highlight js %}
const generateNextBlock = (blockData: string) => {
    const previousBlock: Block = getLatestBlock();
    const nextIndex: number = previousBlock.index + 1;
    const nextTimestamp: number = new Date().getTime() / 1000;
    const nextHash: string = calculateHash(nextIndex, previousBlock.hash, nextTimestamp, blockData);
    const newBlock: Block = new Block(nextIndex, nextHash, previousBlock.hash, nextTimestamp, blockData);
    return newBlock;
};
{% endhighlight %}

# 블록 저장
지금 우리는 javascript in memory 방식으로 블록체인을 저장할 거에요. 즉, 컴퓨터를 끄는 순간 데이터는 사라지게 될 거에요.
{% highlight js %}
const blockchain: Block[] = [genesisBlock];
{% endhighlight %}

# 블록의 고유성을 검사하기
어떤 블록이 유효한 블록인지 검사하는 것은 매우 중요하겠죠. 특히 다른 노드로부터 받은 블록이라면 더더욱. 다음 항목들로 블록의 유효성을 검사할 거에요.
- 이전 블록보다 인덱스값이 1 클 것.
- previousHash값이 이전블록의 hash값일 것.
- hash값이 유효한 값일 것.
다음의 코드로 이를 검사할 수 있어요.
{% highlight js %}
const isValidNewBlock = (newBlock: Block, previousBlock: Block) => {
    if (previousBlock.index + 1 !== newBlock.index) {
        console.log('invalid index');
        return false;
    } else if (previousBlock.hash !== newBlock.previousHash) {
        console.log('invalid previoushash');
        return false;
    } else if (calculateHashForBlock(newBlock) !== newBlock.hash) {
        console.log(typeof (newBlock.hash) + ' ' + typeof calculateHashForBlock(newBlock));
        console.log('invalid hash: ' + calculateHashForBlock(newBlock) + ' ' + newBlock.hash);
        return false;
    }
    return true;
};
{% endhighlight %}

또 블록의 구조 또한 검사해야해요. 그래야 문제있는 블록이 노드를 망가뜨리는 것을 막을 수 있죠.
{% highlight js %}
const isValidBlockStructure = (block: Block): boolean => {
    return typeof block.index === 'number'
        && typeof block.hash === 'string'
        && typeof block.previousHash === 'string'
        && typeof block.timestamp === 'number'
        && typeof block.data === 'string';
};
{% endhighlight %}

이제 우리는 전체 블록체인을 검사할 수 있게 되었어요. 먼저 ‘최초의 블록’을 검사하고 나서, 이어지는 모든 블록을 검사할 거에요, 다음의 코드를 통해서.
{% highlight js %}
const isValidChain = (blockchainToValidate: Block[]): boolean => {
    const isValidGenesis = (block: Block): boolean => {
        return JSON.stringify(block) === JSON.stringify(genesisBlock);
    };

    if (!isValidGenesis(blockchainToValidate[0])) {
        return false;
    }

    for (let i = 1; i < blockchainToValidate.length; i++) {
        if (!isValidNewBlock(blockchainToValidate[i], blockchainToValidate[i - 1])) {
            return false;
        }
    }
    return true;
};
{% endhighlight %}

가장 긴 체인

특정 시점에 ’옳은’ 체인은 하나만 존재해야만 해요. (모든 블록이 정상적이라는 전제 아래) 일단 가장 긴 체인을 ‘옳은’ 체인으로 간주할게요. 왜냐하면 모든 체인은 (이상적으로는) 동일해야 하고, 체인이 길다는 것은 다른 노드가 ‘아직’ 받지 못한 블록을 가지고 있다고 볼 수 있기 때문이죠. 언젠가는 모든 노드가 그 블록을 받아야 할 거에요.
그림.
![conflict_resolving]({{ "/assets/conflict_resolving.png" | absolute_url}})
conflict_resolving
그래서 이런 로직을 가진 코드를 작성할게요.
코드9.
{% highlight js %}
const replaceChain = (newBlocks: Block[]) => {
    if (isValidChain(newBlocks) && newBlocks.length > getBlockchain().length) {
        console.log('Received blockchain is valid. Replacing current blockchain with received blockchain');
        blockchain = newBlocks;
        broadcastLatest();
    } else {
        console.log('Received blockchain invalid');
    }
};
{% endhighlight %}

# 다른 노드와 통신
노드 사이에 통신이 있어야하는 것은 블록체인의 필수요소에요. 데이터의 ‘싱크’를 맞춰야 하기 때문이죠. 다음 룰이 노드 사이의 네트워크에 적용됨으로써 데이터 싱크를 맞추죠.
- 노드가 새 블록을 만들면 그것을 네트워크로 방출(발송,broadcast)해야한다.
- 노드 간에 연결될 때, 각자 지니고 있는 가장 마지막 블록이 무엇인지를 파악한다.
- 자기보다 긴 체인과 연결되면 상대가 가진 블록 중 내가 가진 블록 이후의 모든 블록을 추가하여 싱크를 맞춘다.
그림3.
![conflict_resolving]({{ "/assets/conflict_resolving.png" | absolute_url}})

노드간 통신을 위해 웹소켓을 사용할 게요. 각각의 노드는 서로간의 연결을 웹소켓 배열에 담아 관리해요. 노드가 자동으로 추가되게 만들지 않을 거에요. (URL에 의한)노드의 추가는 수동으로 이루어져요.

# 노드를 제어하기.
노드를 제어하기 위해 HTTP서버를 만들게요. 소스는 다음과 같아요.
코드9.
{% highlight js %}
const initHttpServer = ( myHttpPort: number ) => {
    const app = express();
    app.use(bodyParser.json());

    app.get('/blocks', (req, res) => {
        res.send(getBlockchain());
    });
    app.post('/mineBlock', (req, res) => {
        const newBlock: Block = generateNextBlock(req.body.data);
        res.send(newBlock);
    });
    app.get('/peers', (req, res) => {
        res.send(getSockets().map(( s: any ) => s._socket.remoteAddress + ':' + s._socket.remotePort));
    });
    app.post('/addPeer', (req, res) => {
        connectToPeers(req.body.peer);
        res.send();
    });

    app.listen(myHttpPort, () => {
        console.log('Listening http on port: ' + myHttpPort);
    });
};
{% endhighlight %}

이 서버를 사용하여 다음과 같은 걸 할거에요.
- 블록의 리스트 가져오기
- 새로운 블록을 만들기
- 노드 목록을 가져오거나 새로운 노드를 추가하기
curl명령어로도 노드를 제어할 수 있어요.
코드10.
{% highlight js %}
#get all blocks from the node
> curl http://localhost:3001/blocks
{% endhighlight %}

전체구조.
노드는 두 개의 웹서버와 통신할 거에요. 하나는 사용자가 노드를 제어하기 위한 HTTP서버. 다른 하나는 다른 노드들과 통신하기 위한 웹소켓HTTP서버.

![naivechain_architecture]({{ "/assets/naivechain_architecture.png" | absolute_url}})

마무리.
블록체인의 기본 개념을 익혔어요. 다음챕터에서 작업증명(POW, proof-of-work) 알고리즘을 알아볼게요. 챕터1의 코드는  [여기](https://github.com/lhartikk/naivecoin/tree/chapter1)에 있어요.

[To chapter2]()
