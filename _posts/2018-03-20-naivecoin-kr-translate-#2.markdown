---
layout: post
title:  "Naivecoin Korean Translate Version #2"
date:   2018-03-20 21:18:48 +0900
categories: jekyll update
---

navie코인 번역
원문: [lhartikk.github.io](https://lhartikk.github.io)

## #2: 작업증명(Proof of Work)

# 소개
이번 챕터에서 우리는 장난감블록체인의 작업증명(PoW) 과정을 구현할 거에요. 챕터1에서 만든 블록체인에서는 아무런 수고 없이 누구나 블록을 추가할 수 있었죠. 하지만 이제 계산을 통한 일종의 퍼즐을 푸는 작업을 (증명)해야만 블록이 추가될 거에요. 이렇게 퍼즐을 푸는 작업을 '채굴(mining)'이라고 부르죠.

작업증명은 또한 블록이 얼마나 자주 채굴될 수 있는지를 조절하는 역할도 해요. 퍼즐 문제의 난이도를 통해서요. 만약 블록이 너무 자주 만들어진다 싶으면 문제의 난이도가 올라가고, 너무 드물면 난이도가 내려가죠.

우린 아직 '트랜잭션'에 대해서는 다루지 않았어요. 채굴에 대한 보상이 없다는 얘기죠. 보통 암호화폐에서는 채굴자는 블록을 채굴한 댓가로 보상을 받죠. 하지만 우리의 블록체인에서는 아직 아니랍니다.

# difficulty, nonce 그리고 proof-of-work 퍼즐
블록에 두 개의 속성을 추가하는 걸로 시작해보죠. difficulty와 nonce. 이들은 작업증명(Proof of Work) 퍼즐을 풀기 위해 필요한 값들이에요.

풀어야할 PoW 퍼즐은 '블록의 해쉬값을 찾기'에요. difficulty는 유효한 블록의 해쉬값은 '얼마나 많은 0으로 시작해야 하는가'를 의미해요. 해쉬값을 2진수로 표현한 다음 시작부분의 0의 갯수를 세어보는 거죠. 아래 그림에서 난이도(difficulty) 4와 난이도 8의 예시를 확인해보아요.
![difficulty_examples]({{ "/assets/difficulty_examples.png" | absolute_url }})

블록의 해시값이 난이도difficulty를 만족하는지 확인하는 코드를 살펴보죠.
{% highlight ruby %}
const hashMatchesDifficulty = (hash: string, difficulty: number): boolean => {
    const hashInBinary: string = hexToBinary(hash);
    const requiredPrefix: string = '0'.repeat(difficulty);
    return hashInBinary.startsWith(requiredPrefix);
};
{% endhighlight %}
난이도difficulty를 만족하는 해시값을 찾기 위해서는 같은 블록에서 여러번 반복적으로 해시값을 계산해야해요. 이 때 nonce라는 값을 사용하죠. SHA256이라는 해시함수를 써서 매순간 다른 해시값을 찾아낼 수 있어요. '채굴(mining'이란 기본적으로 난이도를 만족하는 해시값을 찾기 위해 반복적으로 매번 다른 nonce값을 시도해 보는 과정을 의미해요. difficulty와 noce가 추가된 블록 클래스를 보시죠.
{% highlight js %}
class Block {

    public index: number;
    public hash: string;
    public previousHash: string;
    public timestamp: number;
    public data: string;
    public difficulty: number;
    public nonce: number;

    constructor(index: number, hash: string, previousHash: string,
                timestamp: number, data: string, difficulty: number, nonce: number) {
        this.index = index;
        this.previousHash = previousHash;
        this.timestamp = timestamp;
        this.data = data;
        this.hash = hash;
        this.difficulty = difficulty;
        this.nonce = nonce;
    }
}
{% endhighlight %}
최초의 블록도 이 구조에 맞게 변해야겠죠?!

# 블록찾기
위에서 말한 것처럼 유효한 해시값을 찾기 위해서 우리는 nonce값을 증가시켜야 해요. 만족하는 값을 찾는 과정은 완벽하게 랜덤하게 이루어져요. 우리는 단지 '해시값을 찾을 때까지' 반복문을 돌릴 수 있을 뿐이죠.
{% highlight js %}
const findBlock = (index: number, previousHash: string, timestamp: number, data: string, difficulty: number): Block => {
    let nonce = 0;
    while (true) {
        const hash: string = calculateHash(index, previousHash, timestamp, data, difficulty, nonce);
        if (hashMatchesDifficulty(hash, difficulty)) {
            return new Block(index, hash, previousHash, timestamp, data, difficulty, nonce);
        }
        nonce++;
    }
};
{% endhighlight %}
마침내 블록이 '채굴'되면 네트워크를 통해 발송(broadcast)되야겠죠.

# 난이도(difficulty)를 어떻게 결정할 것인가
우리는 이제 주어진 difficulty값을 가지고 해시값을 찾고, 그 값이 유효한지 검증할 수 있게 되었어요. 하지만 difficulty값은 어떻게 정해야 할까요? 이를 위해서는 difficulty값을 계산하는 로직이 필요하겠네요. 상수를 몇개 정의하는 걸로 시작해 보죠.
- BLOCK_GENERATION_INTERVAL: 블록은 얼마나 자주 채굴되는가(Bitcoin의 경우 10분 간격이죠.)
- DIFFICULTY_ADJUSTMENT_INTERVAL: 난이도difficulty는 얼마나 자주 조정되는가(Bitcoin은 2016블록마다 조정돼요.)

우리는 BLOCK_GENERATION_INTERVAL값은 10초, DIFFICULTY_ADJUSTMENT_INTERVAL값은 10블록 마다로 정할게요. 이 상수값들은 변하지 않으므로 하드코딩할게요.
{% highlight js %}
// in seconds
const BLOCK_GENERATION_INTERVAL: number = 10;

// in blocks
const DIFFICULTY_ADJUSTMENT_INTERVAL: number = 10;
{% endhighlight %}

이제 블록채굴의 난이도를 조정할 도구를 갖게 되었어요. 우리는 10블록마다 현재의 채굴 간격이 예상시간보다 낮거나 높은지 체크할 거에요. 예상시간은 BLOCK_GENERATION_INTERVAL * DIFFICULTY_ADJUSTMENT_INTERVAL 으로 계산하면 돼요. 예상시간은 '해시값 획득률'이 '난이도'와 정확히 맞아 떨어지는 경우에 산출되는 값이죠.

만약 예상시간보다 실제 걸리는 시간이 두 배 이상 크거나 작다면 우리는 difficulty값을 조정함으로써 예상시간에 가까이 가고자 노력할 거에요. 이렇게 난이노difficulty조정은 이루어지죠. 즉 채굴간격을 유지하기 위해 difficulty값을 조정하는 거죠. 코드로 확인해보아요.
{% highlight js %}
const getDifficulty = (aBlockchain: Block[]): number => {
    const latestBlock: Block = aBlockchain[blockchain.length - 1];
    if (latestBlock.index % DIFFICULTY_ADJUSTMENT_INTERVAL === 0 && latestBlock.index !== 0) {
        return getAdjustedDifficulty(latestBlock, aBlockchain);
    } else {
        return latestBlock.difficulty;
    }
};

const getAdjustedDifficulty = (latestBlock: Block, aBlockchain: Block[]) => {
    const prevAdjustmentBlock: Block = aBlockchain[blockchain.length - DIFFICULTY_ADJUSTMENT_INTERVAL];
    const timeExpected: number = BLOCK_GENERATION_INTERVAL * DIFFICULTY_ADJUSTMENT_INTERVAL;
    const timeTaken: number = latestBlock.timestamp - prevAdjustmentBlock.timestamp;
    if (timeTaken < timeExpected / 2) {
        return prevAdjustmentBlock.difficulty + 1;
    } else if (timeTaken > timeExpected * 2) {
        return prevAdjustmentBlock.difficulty - 1;
    } else {
        return prevAdjustmentBlock.difficulty;
    }
};
{% endhighlight %}

# Timestamp validation
챕터1에서 timestamp는 아무 역할도 하지 않았어요. 무슨 값이 되든 상관없었죠. 이제 이 값은 난이도difficulty 조정과 관련하여 블록의 유효성을 검증하는데 사용될 거에요. timestamp값으로 블록이 정상적인 난이도 조정을 거친 블록인지 검증하죠.
{% highlight js %}
const isValidTimestamp = (newBlock: Block, previousBlock: Block): boolean => {
    return ( previousBlock.timestamp - 60 < newBlock.timestamp )
        && newBlock.timestamp - 60 < getCurrentTimestamp();
};
{% endhighlight %}

# 누적 난이도 (Cumulative diffculty)
챕터1에서 '옳은 체인'은 '가장 긴 체인'이라고 했던 거 기억하시나요? 이제 이 명제는 변해야만 해요. 'difficulty가 가장 많이 누적된 체인'이 '옳은 체인'이죠. 다시 말하면 옳은 체인은 새로운 블록을 생성하기 위해 가장 많은 자원(resources)을 요구하는, 가장 오래 걸리는(hashRate*time) 체인이에요.

누적 난이도는 2를 difficulty값만큼 곱한 2^을 모두 더하여 계산돼요. 해시값을 2진수로 바꿨을 때 시작부분의 0의 갯수가 diffculty값이였던 거 기억하시죠? 예를 들면 난이도5와 난이도11은 각각 2의 5승, 2의 11승 만큼의 채굴시간이 걸린다고 보는 거죠. 따라서 아래 그림에서 Chain B가 '옳은 체인'이에요.
![Cumulative_difficulties]({{ "/assets/Cumulative_difficulties.png" | absolute_url}})

누적 난이도를 계산할 때 해시값은 상관없어요. 오직 해당 블록의 difficulty값이 중요해요. 물론 해시값이 난이도를 만족한다는 전제 아래에서요.
이 속성을 나카모토 컨센서스(Nakamoto consensus)라고 불러요. 비트코인의 창시자 사토시가 만든 가장 중요한 발명 중 하나죠.
채굴자들은 어느 체인에 그들의 자원을 투입할 것인가를 결정해야만 해요. <<<그들에게 이로운 것은 기존의 블록체인에 그들 스스로 채굴한 블록을 더하는 것이기 때문에, 결국 같은 체인을 선택하도록 장려되죠.>>>

# 결론
작업증명(Proof of Work)에서 가장 중요한 점은 그 과정이 풀기는 어렵지만, 증명하기는 쉬워야한다는 거에요. 특정한 SHA256 해시값을 찾아내는 것은 딱 맞는 사례이죠. 이번 챕터에서 우리는 난이도difficulty를 어떻게 조정하는지를 구현해봤어요. 블록체인 상의 각각의 노드들인 이 diffculty를 조정하면서 체인에 자신이 만든 블록을 더하게 되죠. 이게 바로 트랜잭션이고 다음 챕터에서 구현해 볼거에요.

이번 챕터의 코드는 [여기](https://github.com/lhartikk/naivecoin/tree/chapter2)에 있어요.

[To chapter3](https://newpouy.github.io/jekyll/update/2018/03/22/naivecoin-kr-translate-3.html)
