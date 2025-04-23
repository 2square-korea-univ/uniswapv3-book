## 배포

자, 풀 컨트랙트가 완성되었습니다. 이제 로컬 이더리움 네트워크에 배포하여 나중에 프론트엔드 앱에서 사용할 수 있도록 해봅시다.

## 로컬 블록체인 네트워크 선택

스마트 컨트랙트 개발에는 개발 및 테스트 중에 컨트랙트를 배포할 로컬 네트워크를 실행해야 합니다. 이러한 네트워크에서 원하는 것은 다음과 같습니다.

1. **실제 블록체인.** 에뮬레이션이 아닌 실제 이더리움 네트워크여야 합니다. 컨트랙트가 메인넷에서와 똑같이 로컬 네트워크에서 작동하는지 확인하고 싶습니다.
2. **속도.** 트랜잭션이 즉시 생성되기를 원하므로 빠르게 반복할 수 있습니다.
3. **이더.** 트랜잭션 수수료를 지불하려면 약간의 이더가 필요하며 로컬 네트워크에서 원하는 양의 이더를 생성할 수 있기를 바랍니다.
4. **치트 코드.** 표준 API를 제공하는 것 외에도 로컬 네트워크에서 더 많은 작업을 수행할 수 있기를 바랍니다. 예를 들어, 임의의 주소에 컨트랙트를 배포하고, 임의의 주소에서 트랜잭션을 실행(다른 주소 사칭), 컨트랙트 상태를 직접 변경 등을 할 수 있기를 바랍니다.

현재 여러 가지 솔루션이 있습니다.

1. [Ganache](https://trufflesuite.com/ganache/) (Truffle Suite 제공).
2. [Hardhat](https://hardhat.org/), 유용한 기능 외에도 로컬 노드를 포함하는 개발 환경입니다.
3. [Anvil](https://github.com/foundry-rs/foundry/tree/master/anvil) (Foundry 제공).

이 모든 것들이 실행 가능한 솔루션이며 각각 우리의 요구 사항을 충족할 것입니다. 프로젝트들은 (가장 오래된 솔루션인) Ganache에서 (요즘 가장 널리 사용되는 것으로 보이는) Hardhat으로, 그리고 이제 새로운 강자인 Foundry로 서서히 마이그레이션하고 있습니다. Foundry는 또한 (다른 솔루션들은 JavaScript를 사용하는 반면) 테스트 작성에 Solidity를 사용하는 유일한 솔루션입니다. 게다가 Foundry는 Solidity로 배포 스크립트를 작성하는 것도 허용합니다. 따라서, 모든 곳에서 Solidity를 사용하기로 결정했으므로, Anvil을 사용하여 로컬 개발 블록체인을 실행하고, Solidity로 배포 스크립트를 작성할 것입니다.

## 로컬 블록체인 실행

Anvil은 구성이 필요 없으며, 단일 명령으로 실행할 수 있으며 다음과 같이 작동합니다.

```shell
$ anvil --code-size-limit 50000
                             _   _
                            (_) | |
      __ _   _ __   __   __  _  | |
     / _` | | '_ \  \ \ / / | | | |
    | (_| | | | | |  \ V /  | | | |
     \__,_| |_| |_|   \_/   |_| |_|

    0.1.0 (d89f6af 2022-06-24T00:15:17.897682Z)
    https://github.com/foundry-rs/foundry
...
Listening on 127.0.0.1:8545
```

> 우리는 이더리움 컨트랙트 크기 제한(`24576` 바이트)에 맞지 않는 큰 컨트랙트를 작성할 것이므로, Anvil에게 더 큰 스마트 컨트랙트를 허용하도록 알려줘야 합니다.

Anvil은 단일 이더리움 노드를 실행하므로, 이것은 네트워크가 아니지만 괜찮습니다. 기본적으로 각 계정에 10,000 ETH가 있는 10개의 계정을 생성합니다. 시작할 때 주소와 관련 개인 키를 출력합니다. UI에서 컨트랙트를 배포하고 상호 작용할 때 이러한 주소 중 하나를 사용할 것입니다.

Anvil은 `127.0.0.1:8545`에서 JSON-RPC API 인터페이스를 노출합니다. 이 인터페이스는 이더리움 노드와 상호 작용하는 주요 방법입니다. 전체 API 참조는 [여기](https://ethereum.org/en/developers/docs/apis/json-rpc/)에서 찾을 수 있습니다. 그리고 이것이 `curl`을 통해 호출하는 방법입니다.
```shell
$ curl -X POST -H 'Content-Type: application/json' \
  --data '{"id":1,"jsonrpc":"2.0","method":"eth_chainId"}' \
  http://127.0.0.1:8545
{"jsonrpc":"2.0","id":1,"result":"0x7a69"}
$ curl -X POST -H 'Content-Type: application/json' \
  --data '{"id":1,"jsonrpc":"2.0","method":"eth_getBalance","params":["0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266","latest"]}' \
  http://127.0.0.1:8545
{"jsonrpc":"2.0","id":1,"result":"0x21e19e0c9bab2400000"}
```

Foundry의 일부인 `cast`를 사용할 수도 있습니다.
```shell
$ cast chain-id
31337
$ cast balance 0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
10000000000000000000000
```

이제 풀과 매니저 컨트랙트를 로컬 네트워크에 배포해 봅시다.

## 첫 번째 배포

핵심적으로 컨트랙트 배포는 다음을 의미합니다.

1. 소스 코드를 EVM 바이트코드로 컴파일합니다.
2. 바이트코드가 포함된 트랜잭션을 보냅니다.
3. 새 주소를 생성하고, 바이트코드의 생성자 부분을 실행하고, 배포된 바이트코드를 주소에 저장합니다. 이 단계는 컨트랙트 생성 트랜잭션이 마이닝될 때 이더리움 노드에 의해 자동으로 수행됩니다.

배포는 일반적으로 여러 단계로 구성됩니다. 파라미터 준비, 보조 컨트랙트 배포, 주요 컨트랙트 배포, 컨트랙트 초기화 등입니다. 스크립팅은 이러한 단계를 자동화하는 데 도움이 되며, Solidity로 스크립트를 작성할 것입니다!

다음 내용으로 `scripts/DeployDevelopment.sol` 컨트랙트를 생성합니다.
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "forge-std/Script.sol";

contract DeployDevelopment is Script {
    function run() public {
      ...
    }
}
```

테스트 컨트랙트와 매우 유사하게 보이며, 유일한 차이점은 `Test`가 아닌 `Script` 컨트랙트에서 상속한다는 것입니다. 그리고 관례상 배포 스크립트의 본문이 될 `run` 함수를 정의해야 합니다. `run` 함수에서 먼저 배포 파라미터를 정의합니다.
```solidity
uint256 wethBalance = 1 ether;
uint256 usdcBalance = 5042 ether;
int24 currentTick = 85176;
uint160 currentSqrtP = 5602277097478614198912276234240;
```
이것들은 이전에 사용했던 값과 동일합니다. 5042 USDC를 민트하려고 합니다. 이는 풀에 유동성으로 제공할 5000 USDC와 스왑으로 판매할 42 USDC입니다.

다음으로 배포 트랜잭션으로 실행될 단계 집합을 정의합니다 (각 단계는 별도의 트랜잭션이 됩니다). 이를 위해 `startBroadcast/endBroadcast` 치트 코드를 사용합니다.
```solidity
vm.startBroadcast();
...
vm.stopBroadcast();
```

> 이러한 치트 코드는 [Foundry에서 제공](https://github.com/foundry-rs/foundry/tree/master/forge#cheat-codes)됩니다. `forge-std/Script.sol`에서 상속하여 스크립트 컨트랙트에서 얻었습니다.

`broadcast()` 치트 코드 이후 또는 `startBroadcast()/stopBroadcast()` 사이에 오는 모든 것은 트랜잭션으로 변환되고 이러한 트랜잭션은 스크립트를 실행하는 노드로 전송됩니다.

브로드캐스트 치트 코드 사이에 실제 배포 단계를 넣을 것입니다. 먼저 토큰을 배포해야 합니다.
```solidity
ERC20Mintable token0 = new ERC20Mintable("Wrapped Ether", "WETH", 18);
ERC20Mintable token1 = new ERC20Mintable("USD Coin", "USDC", 18);
```
토큰 없이는 풀을 배포할 수 없으므로 먼저 토큰을 배포해야 합니다.

> 로컬 개발 네트워크에 배포하므로 토큰을 직접 배포해야 합니다. 메인넷 및 공용 테스트 네트워크(Ropsten, Goerli, Sepolia)에서는 토큰이 이미 생성되어 있습니다. 따라서 이러한 네트워크에 배포하려면 네트워크별 배포 스크립트를 작성해야 합니다.

다음 단계는 풀 컨트랙트를 배포하는 것입니다.
```solidity
UniswapV3Pool pool = new UniswapV3Pool(
    address(token0),
    address(token1),
    currentSqrtP,
    currentTick
);
```

다음은 매니저 컨트랙트 배포입니다.
```solidity
UniswapV3Manager manager = new UniswapV3Manager();
```

마지막으로, ETH와 USDC를 주소로 약간 민트할 수 있습니다.
```solidity
token0.mint(msg.sender, wethBalance);
token1.mint(msg.sender, usdcBalance);
```

> Foundry 스크립트에서 `msg.sender`는 `broadcast` 블록 내에서 트랜잭션을 보내는 주소입니다. 스크립트를 실행할 때 설정할 수 있습니다.

마지막으로 스크립트 끝에 배포된 컨트랙트의 주소를 출력하는 `console.log` 호출을 추가합니다.
```solidity
console.log("WETH address", address(token0));
console.log("USDC address", address(token1));
console.log("Pool address", address(pool));
console.log("Manager address", address(manager));
```

좋습니다, 스크립트를 실행해 봅시다 (Anvil이 다른 터미널 창에서 실행 중인지 확인하십시오).
```shell
$ forge script scripts/DeployDevelopment.s.sol --broadcast --fork-url http://localhost:8545 --private-key $PRIVATE_KEY  --code-size-limit 50000
```

> 컴파일러가 실패하지 않도록 스마트 컨트랙트 코드 크기를 다시 늘리고 있습니다.

`--broadcast`는 트랜잭션 브로드캐스팅을 활성화합니다. 모든 스크립트가 트랜잭션을 보내는 것은 아니기 때문에 기본적으로 활성화되어 있지 않습니다. `--fork-url`은 트랜잭션을 보낼 노드의 주소를 설정합니다. `--private-key`는 발신자 지갑을 설정합니다. 트랜잭션에 서명하려면 개인 키가 필요합니다. Anvil이 시작될 때 출력하는 개인 키 중 아무거나 선택할 수 있습니다. 저는 첫 번째 것을 선택했습니다.
> 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

배포는 몇 초가 걸립니다. 마지막에 보낸 트랜잭션 목록이 표시됩니다. 또한 `broadcast` 폴더에 트랜잭션 영수증을 저장합니다. Anvil에서는 `eth_sendRawTransaction`, `eth_getTransactionByHash` 및 `eth_getTransactionReceipt`가 있는 많은 줄도 볼 수 있습니다. Forge는 Anvil에 트랜잭션을 보낸 후 JSON-RPC API를 사용하여 상태를 확인하고 트랜잭션 실행 결과(영수증)를 얻습니다.

축하합니다! 스마트 컨트랙트를 배포했습니다!

## 컨트랙트와 상호 작용, ABI

이제 배포된 컨트랙트와 상호 작용하는 방법을 알아봅시다.

모든 컨트랙트는 공개 함수 집합을 노출합니다. 풀 컨트랙트의 경우 `mint(...)` 및 `swap(...)`입니다. 또한 Solidity는 공개 변수에 대한 getter를 생성하므로 `token0()`, `token1()`, `positions()` 등을 호출할 수도 있습니다. 그러나 컨트랙트는 컴파일된 바이트코드이므로 **함수 이름은 컴파일 중에 손실되고 블록체인에 저장되지 않습니다**. 대신 모든 함수는 함수 시그니처 해시의 처음 4바이트인 셀렉터로 식별됩니다. 유사 코드:
```
hash("transfer(address,address,uint256)")[0:4]
```

> EVM은 [Keccak 해싱 알고리즘](https://en.wikipedia.org/wiki/SHA-3)을 사용하며, 이는 SHA-3으로 표준화되었습니다. 특히 Solidity의 해싱 함수는 `keccak256`입니다.

이를 알고 배포된 컨트랙트에 두 번 호출해 봅시다. 하나는 `curl`을 통한 저수준 호출이고, 다른 하나는 `cast`를 사용하여 수행됩니다.

### 토큰 잔액

배포자 주소의 WETH 잔액을 확인해 봅시다. 함수 시그니처는 `balanceOf(address)`입니다 ([ERC-20](https://eips.ethereum.org/EIPS/eip-20)에 정의됨). 이 함수의 ID(셀렉터)를 찾으려면 해시하고 처음 4바이트를 취합니다.
```shell
$ cast keccak "balanceOf(address)"| cut -b 1-10
0x70a08231
```

주소를 전달하려면 함수 셀렉터에 주소를 추가하기만 하면 됩니다 (그리고 주소는 함수 호출 데이터에서 32바이트를 차지하므로 최대 32자리까지 왼쪽 패딩을 추가합니다).
> 0x70a08231000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266

`0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266`은 잔액을 확인할 주소입니다. 이것은 Anvil의 첫 번째 계정인 우리의 주소입니다.

다음으로 `eth_call` JSON-RPC 메서드를 실행하여 호출합니다. 이것은 트랜잭션 전송이 필요하지 않습니다. 이 엔드포인트는 컨트랙트에서 데이터를 읽는 데 사용됩니다.

```shell
$ params='{"from":"0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266","to":"0xe7f1725e7734ce288f8367e1bb143e90bb3f0512","data":"0x70a08231000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266"}'
$ curl -X POST -H 'Content-Type: application/json' \
  --data '{"id":1,"jsonrpc":"2.0","method":"eth_call","params":['"$params"',"latest"]}' \
  http://127.0.0.1:8545
{"jsonrpc":"2.0","id":1,"result":"0x00000000000000000000000000000000000000000000011153ce5e56cf880000"}
```

> "to" 주소는 USDC 토큰입니다. 배포 스크립트에 의해 출력되며 사용자마다 다를 수 있습니다.

이더리움 노드는 결과를 원시 바이트로 반환하며, 이를 파싱하려면 반환된 값의 유형을 알아야 합니다. `balanceOf` 함수의 경우 반환된 값의 유형은 `uint256`입니다. `cast`를 사용하여 십진수로 변환한 다음 이더로 변환할 수 있습니다.
```shell
$ cast --to-dec 0x00000000000000000000000000000000000000000000011153ce5e56cf880000| cast --from-wei
5042.000000000000000000
```

잔액이 정확합니다! 주소로 5042 USDC를 민트했습니다.

### 현재 틱 및 가격

위의 예는 저수준 컨트랙트 호출의 데모입니다. 일반적으로 `curl`을 통해 호출하지 않고 더 쉽게 만들 수 있는 도구나 라이브러리를 사용합니다. 그리고 Cast가 여기서 다시 우리를 도울 수 있습니다!

`cast`를 사용하여 풀의 현재 가격과 틱을 가져와 봅시다.
```shell
$ cast call POOL_ADDRESS "slot0()"| xargs cast --abi-decode "a()(uint160,int24)"
5602277097478614198912276234240
85176
```

좋습니다! 첫 번째 값은 현재 $\sqrt{P}$이고 두 번째 값은 현재 틱입니다.

> `--abi-decode`는 전체 함수 시그니처가 필요하므로 함수 출력만 디코딩하려는 경우에도 "a()"를 지정해야 합니다.

### ABI

컨트랙트와의 상호 작용을 단순화하기 위해 Solidity 컴파일러는 ABI (Application Binary Interface)를 출력할 수 있습니다.

ABI는 컨트랙트의 모든 public 메서드와 이벤트에 대한 설명을 포함하는 JSON 파일입니다. 이 파일의 목표는 함수 파라미터를 더 쉽게 인코딩하고 반환 값을 디코딩하는 것입니다. Forge에서 ABI를 얻으려면 다음 명령을 사용하십시오.

```shell
$ forge inspect UniswapV3Pool abi
```

파일 내용을 더 잘 이해하기 위해 자유롭게 훑어보십시오.