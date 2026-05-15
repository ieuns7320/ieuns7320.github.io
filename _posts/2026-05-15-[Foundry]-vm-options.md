---
title: Foundry > Hello Ethernaut
author: ieuns
date: 2026-05-15 10:51:00 +0900
categories: [Web3, Solidity, Foundry]
tags: [tools]
---

# Foundry Test 라이브러리 `vm.` Cheatcode 정리

## 1. 전체 요약

Foundry에서 `vm.`은 **Cheatcode**를 호출할 때 사용하는 객체다.

Cheatcode란 실제 Solidity/EVM에는 존재하지 않지만, Foundry 테스트 환경에서만 사용할 수 있는 특수 기능이다.

주로 다음과 같은 작업을 할 때 사용한다.

- 특정 주소로 `msg.sender` 조작
- 테스트 계정에 ETH 지급
- revert 발생 여부 검증
- block timestamp / block number 변경
- storage slot 직접 읽기/쓰기
- 메인넷/테스트넷 fork 생성
- 환경변수 읽기
- 배포 스크립트에서 트랜잭션 broadcast

즉, `vm.`은 스마트 컨트랙트 테스트를 위해 **EVM 상태를 강제로 조작하거나 검증하는 도구**라고 보면 된다.

---

## 2. 기본 사용 방식

Foundry 테스트 파일에서는 보통 `forge-std/Test.sol`을 import하고 `Test`를 상속한다.

```solidity
import "forge-std/Test.sol";

contract MyTest is Test {
    function testSomething() public {
        vm.prank(address(1));
        // 다음 호출의 msg.sender가 address(1)이 됨
    }
}
```

여기서 `vm.prank()`처럼 `vm.` 뒤에 붙는 함수들이 Foundry Cheatcode다.

---

# 3. 자주 사용하는 `vm.` 옵션

## 3.1 `vm.prank()`

### 역할

다음 **한 번의 외부 호출**에서 `msg.sender`를 특정 주소로 바꾼다.

### 문법

```solidity
vm.prank(address);
```

### 예시

```solidity
address alice = makeAddr("alice");

vm.prank(alice);
vault.deposit{value: 1 ether}();
```

### 설명

위 코드는 `vault.deposit()`을 호출할 때 `msg.sender`가 테스트 컨트랙트가 아니라 `alice`가 되도록 만든다.

즉, 실제로는 테스트 컨트랙트가 호출하지만, 컨트랙트 입장에서는 `alice`가 호출한 것처럼 보인다.

### 자주 쓰는 상황

- 특정 사용자가 함수를 호출하는 상황 테스트
- owner가 아닌 사용자의 접근 테스트
- 공격자 주소로 함수 호출 테스트

---

## 3.2 `vm.startPrank()` / `vm.stopPrank()`

### 역할

여러 번의 호출 동안 `msg.sender`를 특정 주소로 유지한다.

### 문법

```solidity
vm.startPrank(address);
...
vm.stopPrank();
```

### 예시

```solidity
address alice = makeAddr("alice");

vm.startPrank(alice);

vault.deposit{value: 1 ether}();
vault.withdraw(0.5 ether);

vm.stopPrank();
```

### 설명

`vm.prank()`는 다음 호출 1번에만 적용된다.

반면 `vm.startPrank()`는 `vm.stopPrank()`가 호출되기 전까지 계속 적용된다.

### 차이점

```solidity
vm.prank(alice);
target.callOne(); // alice로 호출됨
target.callTwo(); // 다시 테스트 컨트랙트가 호출한 것으로 처리됨
```

```solidity
vm.startPrank(alice);
target.callOne(); // alice로 호출됨
target.callTwo(); // alice로 호출됨
vm.stopPrank();
```

### 자주 쓰는 상황

- 한 사용자가 여러 행동을 연속으로 수행하는 테스트
- deposit 후 withdraw 테스트
- approve 후 transferFrom 테스트

---

## 3.3 `vm.deal()`

### 역할

특정 주소에 ETH 잔액을 넣는다.

### 문법

```solidity
vm.deal(address, amount);
```

### 예시

```solidity
address alice = makeAddr("alice");

vm.deal(alice, 10 ether);
```

### 설명

테스트에서 만든 주소는 기본적으로 ETH가 없다.

따라서 payable 함수에 ETH를 보내려면 먼저 `vm.deal()`로 잔액을 넣어줘야 한다.

### 실전 예시

```solidity
function testDeposit() public {
    address alice = makeAddr("alice");

    vm.deal(alice, 1 ether);

    vm.prank(alice);
    vault.deposit{value: 1 ether}();

    assertEq(vault.balances(alice), 1 ether);
}
```

### 흐름

```text
1. alice 주소 생성
2. alice에게 1 ETH 지급
3. alice가 호출한 것처럼 msg.sender 조작
4. alice가 1 ETH deposit
5. vault 내부 balance 확인
```

---

## 3.4 `vm.expectRevert()`

### 역할

다음 호출이 revert 되는지 검증한다.

### 문법

```solidity
vm.expectRevert();
```

또는 revert 메시지를 지정할 수도 있다.

```solidity
vm.expectRevert("Not owner");
```

custom error를 검증할 수도 있다.

```solidity
vm.expectRevert(MyError.selector);
```

### 예시

```solidity
address attacker = makeAddr("attacker");

vm.prank(attacker);
vm.expectRevert("Not owner");
target.withdraw();
```

### 설명

위 코드는 `attacker`가 `withdraw()`를 호출했을 때 `"Not owner"` 메시지와 함께 revert 되어야 테스트가 통과한다.

만약 revert가 발생하지 않으면 테스트는 실패한다.

### 주의점

`vm.expectRevert()`는 **바로 다음 external call**에 적용된다.

```solidity
vm.expectRevert();
target.someFunction();
```

이런 식으로 바로 다음 줄에 검증하려는 호출을 두는 것이 좋다.

---

## 3.5 `vm.warp()`

### 역할

`block.timestamp` 값을 변경한다.

### 문법

```solidity
vm.warp(newTimestamp);
```

### 예시

```solidity
vm.warp(block.timestamp + 7 days);
staking.withdraw();
```

### 설명

staking, vesting, timelock, cooldown처럼 시간이 중요한 컨트랙트를 테스트할 때 사용한다.

### 자주 쓰는 상황

- 7일 후 출금 가능 여부 테스트
- lock 기간 이후 claim 테스트
- deadline 만료 테스트

---

## 3.6 `vm.roll()`

### 역할

`block.number` 값을 변경한다.

### 문법

```solidity
vm.roll(newBlockNumber);
```

### 예시

```solidity
vm.roll(block.number + 100);
```

### 설명

블록 번호 기반 로직을 테스트할 때 사용한다.

### 자주 쓰는 상황

- 특정 블록 이후 실행 가능
- 블록 번호 기반 보상 계산
- governance voting delay 테스트

---

## 3.7 `vm.load()`

### 역할

특정 컨트랙트의 storage slot 값을 직접 읽는다.

### 문법

```solidity
bytes32 value = vm.load(address target, bytes32 slot);
```

### 예시

```solidity
bytes32 slot = bytes32(uint256(0));
bytes32 value = vm.load(address(target), slot);
```

### 설명

Solidity의 storage는 slot 단위로 저장된다.

`vm.load()`를 사용하면 getter 함수가 없어도 특정 slot 값을 직접 읽을 수 있다.

### 자주 쓰는 상황

- proxy storage 확인
- EIP-1967 implementation slot 확인
- storage collision 분석
- Ethernaut PuzzleWallet류 문제 분석

---

## 3.8 `vm.store()`

### 역할

특정 컨트랙트의 storage slot 값을 강제로 변경한다.

### 문법

```solidity
vm.store(address target, bytes32 slot, bytes32 value);
```

### 예시

```solidity
bytes32 slot = bytes32(uint256(0));
bytes32 newValue = bytes32(uint256(123));

vm.store(address(target), slot, newValue);
```

### 설명

`vm.store()`는 컨트랙트 내부 로직을 거치지 않고 storage 값을 직접 바꾼다.

그래서 일반적인 기능 테스트보다는 보안 테스트, storage 분석, 특수 상황 재현에 많이 사용된다.

### 주의점

실제 블록체인에서는 이런 식으로 storage를 마음대로 바꿀 수 없다.

테스트 환경에서만 가능한 조작이다.

---

## 3.9 `vm.createFork()`

### 역할

메인넷이나 테스트넷 상태를 복사해서 로컬 테스트 환경에 fork를 만든다.

### 문법

```solidity
uint256 forkId = vm.createFork(RPC_URL);
```

특정 블록 기준으로 fork할 수도 있다.

```solidity
uint256 forkId = vm.createFork(RPC_URL, blockNumber);
```

### 예시

```solidity
string memory rpc = vm.envString("RPC_URL");

uint256 fork = vm.createFork(rpc);
vm.selectFork(fork);
```

### 설명

실제 배포된 컨트랙트를 대상으로 테스트하고 싶을 때 사용한다.

예를 들어 이미 배포된 ERC20, DeFi protocol, proxy contract 상태를 그대로 가져와서 테스트할 수 있다.

---

## 3.10 `vm.selectFork()`

### 역할

여러 fork 중 현재 사용할 fork를 선택한다.

### 문법

```solidity
vm.selectFork(forkId);
```

### 예시

```solidity
uint256 mainnetFork = vm.createFork(MAINNET_RPC_URL);
uint256 sepoliaFork = vm.createFork(SEPOLIA_RPC_URL);

vm.selectFork(mainnetFork);
```

### 설명

여러 체인 또는 여러 블록 상태를 테스트할 때 사용한다.

---

## 3.11 `vm.envString()`, `vm.envUint()`, `vm.envAddress()`

### 역할

환경변수를 읽는다.

### 문법

```solidity
vm.envString("RPC_URL");
vm.envUint("PK");
vm.envAddress("OWNER");
```

### 예시

```solidity
string memory rpc = vm.envString("RPC_URL");
uint256 privateKey = vm.envUint("PK");
address owner = vm.envAddress("OWNER");
```

### 설명

`.env` 파일이나 터미널 환경변수에 저장된 값을 Solidity 테스트나 스크립트 안에서 읽을 때 사용한다.

### 자주 쓰는 상황

- RPC URL 읽기
- private key 읽기
- 배포자 주소 읽기
- 테스트 대상 컨트랙트 주소 읽기

---

## 3.12 `vm.startBroadcast()` / `vm.stopBroadcast()`

### 역할

배포 스크립트에서 실제 트랜잭션을 broadcast한다.

주로 `Script.sol`에서 사용한다.

### 문법

```solidity
vm.startBroadcast(privateKey);
...
vm.stopBroadcast();
```

### 예시

```solidity
import "forge-std/Script.sol";

contract Deploy is Script {
    function run() external {
        uint256 pk = vm.envUint("PK");

        vm.startBroadcast(pk);

        new MyContract();

        vm.stopBroadcast();
    }
}
```

### 설명

`forge script`를 사용해서 실제 네트워크에 컨트랙트를 배포할 때 사용한다.

테스트보다는 배포 자동화에서 더 자주 사용한다.

---

## 3.13 `vm.expectEmit()`

### 역할

이벤트가 정상적으로 발생하는지 검증한다.

### 문법

```solidity
vm.expectEmit();
```

좀 더 세밀하게 topic/data 검증 여부를 지정할 수 있다.

```solidity
vm.expectEmit(
    checkTopic1,
    checkTopic2,
    checkTopic3,
    checkData
);
```

### 예시

```solidity
vm.expectEmit(true, true, false, true);
emit Transfer(alice, bob, 100);

token.transfer(bob, 100);
```

### 설명

위 코드는 `token.transfer()`를 호출했을 때 예상한 `Transfer` 이벤트가 발생하는지 검증한다.

---

## 3.14 `vm.expectCall()`

### 역할

특정 컨트랙트에 특정 call이 발생하는지 검증한다.

### 문법

```solidity
vm.expectCall(target, data);
```

### 예시

```solidity
vm.expectCall(
    address(token),
    abi.encodeWithSelector(token.transfer.selector, bob, 100)
);

vault.withdraw(bob, 100);
```

### 설명

`vault.withdraw()` 내부에서 `token.transfer(bob, 100)`이 호출되는지 검증할 수 있다.

### 자주 쓰는 상황

- ERC20 transfer 호출 여부 확인
- 외부 컨트랙트 call 검증
- protocol 내부 상호작용 검증

---

# 4. `console.log`와 `vm.`의 차이

`console.log`는 `vm.` cheatcode가 아니다.

하지만 Foundry 테스트에서 자주 같이 사용한다.

```solidity
import "forge-std/console.sol";
```

예시:

```solidity
console.log("owner:", owner);
console.log("balance:", address(vault).balance);
```

`console.log`는 테스트 중 값을 확인하기 위한 디버깅 도구다.

반면 `vm.`은 EVM 상태를 조작하거나 검증하는 cheatcode다.

---

# 5. Vault 테스트 기준으로 꼭 알아야 할 것

Vault를 직접 구현하고 테스트하는 단계라면 우선 아래 4개만 제대로 익혀도 된다.

```solidity
vm.deal(user, 1 ether);
vm.prank(user);
vm.expectRevert();
assertEq(actual, expected);
```

정리하면 다음과 같다.

| 기능 | 사용 예시 | 의미 |
|---|---|---|
| ETH 지급 | `vm.deal(alice, 1 ether)` | alice에게 테스트용 ETH 지급 |
| 호출자 변경 | `vm.prank(alice)` | 다음 호출의 msg.sender를 alice로 변경 |
| 실패 검증 | `vm.expectRevert()` | 다음 호출이 revert 되어야 함 |
| 값 검증 | `assertEq(a, b)` | a와 b가 같은지 확인 |

---

# 6. Vault deposit 테스트 예시

```solidity
function testDeposit() public {
    address alice = makeAddr("alice");

    vm.deal(alice, 1 ether);

    vm.prank(alice);
    vault.deposit{value: 1 ether}();

    assertEq(vault.balances(alice), 1 ether);
}
```

## 코드 설명

```text
1. makeAddr("alice")로 테스트용 alice 주소를 만든다.
2. vm.deal(alice, 1 ether)로 alice에게 1 ETH를 준다.
3. vm.prank(alice)로 다음 호출의 msg.sender를 alice로 바꾼다.
4. alice가 vault.deposit{value: 1 ether}()를 호출한 것처럼 만든다.
5. vault.balances(alice)가 1 ether인지 assertEq로 확인한다.
```

---

# 7. Vault withdraw 실패 테스트 예시

```solidity
function testWithdrawRevertIfInsufficientBalance() public {
    address alice = makeAddr("alice");

    vm.prank(alice);
    vm.expectRevert();
    vault.withdraw(1 ether);
}
```

## 코드 설명

```text
1. alice 주소를 만든다.
2. alice가 호출한 것처럼 msg.sender를 바꾼다.
3. 다음 호출이 revert 되어야 한다고 지정한다.
4. alice는 deposit한 ETH가 없으므로 withdraw(1 ether)는 실패해야 한다.
```

