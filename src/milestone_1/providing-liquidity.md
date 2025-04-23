# 유동성 공급

이론은 충분합니다. 이제 코딩을 시작해 봅시다!

새로운 폴더(`uniswapv3-code`라고 하겠습니다)를 만들고, 그 안에서 `forge init --vscode`를 실행하세요. Forge 프로젝트가 초기화될 것입니다. `--vscode` 플래그는 Forge에게 Forge 프로젝트를 위한 Solidity 확장을 구성하도록 지시합니다.

다음으로, 기본 계약과 테스트를 제거합니다:
- `script/Contract.s.sol`
- `src/Contract.sol`
- `test/Contract.t.sol`

이것으로 끝입니다! 이제 첫 번째 계약을 만들어 봅시다!

## 풀 계약

소개에서 배우셨듯이, Uniswap은 여러 풀 계약을 배포하며, 각 계약은 토큰 쌍의 교환 시장입니다. Uniswap은 모든 계약을 두 가지 범주로 그룹화합니다:

- 코어 계약,
- 주변부 계약.

코어 계약은 이름에서 알 수 있듯이 핵심 로직을 구현하는 계약입니다. 이들은 최소한으로, 사용자 친화적이지 않으며, 저수준 계약입니다. 그들의 목적은 한 가지 일을 가능한 한 안정적이고 안전하게 수행하는 것입니다. Uniswap V3에는 그러한 계약이 2가지 있습니다:
1. 풀 계약, 탈중앙화 거래소의 핵심 로직을 구현합니다.
2. 팩토리 계약, 풀 계약의 레지스트리 역할을 하며 풀 배포를 더 쉽게 만드는 계약입니다.

Uniswap의 핵심 기능의 99%를 구현하는 풀 계약부터 시작해 보겠습니다.

`src/UniswapV3Pool.sol`을 생성합니다:

```solidity
pragma solidity ^0.8.14;

contract UniswapV3Pool {}
```

계약이 저장할 데이터에 대해 생각해 봅시다:
1. 모든 풀 계약은 두 토큰의 교환 시장이므로, 두 토큰 주소를 추적해야 합니다. 이 주소들은 정적이며, 풀 배포 중에 한 번 영구적으로 설정됩니다 (따라서 불변(immutable)으로 설정됩니다).
2. 각 풀 계약은 유동성 포지션의 집합입니다. 이들을 매핑에 저장할 것이며, 여기서 키는 고유한 포지션 식별자이고 값은 포지션에 대한 정보를 담고 있는 구조체입니다.
3. 각 풀 계약은 틱 레지스트리도 유지해야 합니다. 이것은 키가 틱 인덱스이고 값이 틱에 대한 정보를 저장하는 구조체인 매핑입니다.
4. 틱 범위는 제한되어 있으므로, 계약에 상수로 제한을 저장해야 합니다.
5. 풀 계약은 유동성 $L$의 양을 저장한다는 것을 상기하십시오. 따라서 이를 위한 변수가 필요합니다.
6. 마지막으로, 현재 가격과 관련 틱을 추적해야 합니다. 가스 소비를 최적화하기 위해 하나의 저장소 슬롯에 저장할 것입니다: 이 변수들은 자주 읽고 쓰여지므로, [Solidity의 상태 변수 패킹 기능](https://docs.soliditylang.org/en/v0.8.17/internals/layout_in_storage.html)의 이점을 활용하는 것이 합리적입니다.

전반적으로, 이것이 우리가 시작하는 것입니다:

```solidity
// src/lib/Tick.sol
library Tick {
    struct Info {
        bool initialized;
        uint128 liquidity;
    }
    ...
}

// src/lib/Position.sol
library Position {
    struct Info {
        uint128 liquidity;
    }
    ...
}

// src/UniswapV3Pool.sol
contract UniswapV3Pool {
    using Tick for mapping(int24 => Tick.Info);
    using Position for mapping(bytes32 => Position.Info);
    using Position for Position.Info;

    int24 internal constant MIN_TICK = -887272;
    int24 internal constant MAX_TICK = -MIN_TICK;

    // 풀 토큰, 불변
    address public immutable token0;
    address public immutable token1;

    // 함께 읽는 변수 패킹
    struct Slot0 {
        // 현재 sqrt(P)
        uint160 sqrtPriceX96;
        // 현재 틱
        int24 tick;
    }
    Slot0 public slot0;

    // 유동성 양, L.
    uint128 public liquidity;

    // 틱 정보
    mapping(int24 => Tick.Info) public ticks;
    // 포지션 정보
    mapping(bytes32 => Position.Info) public positions;

    ...
```

Uniswap V3는 많은 헬퍼 계약을 사용하며 `Tick`과 `Position`은 그 중 두 개입니다. `using A for B`는 Solidity의 기능으로, 라이브러리 계약 `A`의 함수로 타입 `B`를 확장할 수 있습니다. 이것은 복잡한 데이터 구조 관리를 단순화합니다.

> 간결함을 위해, Solidity 구문 및 기능에 대한 자세한 설명은 생략하겠습니다. Solidity는 [훌륭한 문서](https://docs.soliditylang.org/en/latest/)를 가지고 있으니, 명확하지 않은 부분이 있다면 주저하지 말고 참조하십시오!

이제 생성자에서 일부 변수를 초기화해 보겠습니다:

```solidity
    constructor(
        address token0_,
        address token1_,
        uint160 sqrtPriceX96,
        int24 tick
    ) {
        token0 = token0_;
        token1 = token1_;

        slot0 = Slot0({sqrtPriceX96: sqrtPriceX96, tick: tick});
    }
}
```

여기서, 토큰 주소를 불변으로 설정하고 현재 가격과 틱을 설정합니다. 후자에 대해서는 유동성을 제공할 필요가 없습니다.

이것이 우리의 시작점이며, 이 장의 목표는 미리 계산되고 하드 코딩된 값을 사용하여 첫 번째 스왑을 만드는 것입니다.

## 민팅

Uniswap V2에서 유동성을 제공하는 과정을 *민팅*이라고 합니다. 그 이유는 V2 풀 계약이 유동성에 대한 대가로 토큰(LP-토큰)을 민팅하기 때문입니다. V3는 그렇게 하지 않지만, 여전히 함수에 동일한 이름을 사용합니다. 우리도 그것을 사용해 봅시다:

```solidity
function mint(
    address owner,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount
) external returns (uint256 amount0, uint256 amount1) {
    ...
```

우리의 `mint` 함수는 다음을 인수로 받습니다:
1. 유동성의 소유자를 추적하기 위한 소유자 주소.
2. 가격 범위의 경계를 설정하기 위한 상위 및 하위 틱.
3. 제공하고자 하는 유동성의 양.

> 사용자가 실제 토큰 양이 아닌 $L$을 지정한다는 점에 유의하십시오. 물론 이것은 그다지 편리하지 않지만, 풀 계약은 코어 계약이라는 것을 상기하십시오. 코어 로직만 구현해야 하므로 사용자 친화적일 필요는 없습니다. 나중에 헬퍼 계약을 만들어 `Pool.mint`를 호출하기 전에 토큰 양을 $L$로 변환할 것입니다.

민팅이 어떻게 작동할지에 대한 간단한 계획을 간략하게 설명해 보겠습니다:
1. 사용자가 가격 범위와 유동성 양을 지정합니다.
2. 계약이 `ticks` 및 `positions` 매핑을 업데이트합니다.
3. 계약이 사용자가 보내야 하는 토큰 양을 계산합니다 (미리 계산하고 하드 코딩할 것입니다).
4. 계약이 사용자로부터 토큰을 받고 올바른 양이 설정되었는지 확인합니다.

틱을 확인하는 것부터 시작해 보겠습니다:
```solidity
if (
    lowerTick >= upperTick ||
    lowerTick < MIN_TICK ||
    upperTick > MAX_TICK
) revert InvalidTickRange();
```

그리고 일정량의 유동성이 제공되었는지 확인합니다:
```solidity
if (amount == 0) revert ZeroLiquidity();
```

그런 다음, 틱과 포지션을 추가합니다:
```solidity
ticks.update(lowerTick, amount);
ticks.update(upperTick, amount);

Position.Info storage position = positions.get(
    owner,
    lowerTick,
    upperTick
);
position.update(amount);
```

`ticks.update` 함수는 다음과 같습니다:

```solidity
// src/lib/Tick.sol
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint128 liquidityDelta
) internal {
    Tick.Info storage tickInfo = self[tick];
    uint128 liquidityBefore = tickInfo.liquidity;
    uint128 liquidityAfter = liquidityBefore + liquidityDelta;

    if (liquidityBefore == 0) {
        tickInfo.initialized = true;
    }

    tickInfo.liquidity = liquidityAfter;
}
```

틱에 유동성이 0이면 초기화하고, 새로운 유동성을 추가합니다. 보시다시피, 하위 및 상위 틱 모두에서 이 함수를 호출하므로 유동성이 둘 다에 추가됩니다.

`position.update` 함수는 다음과 같습니다:
```solidity
// src/libs/Position.sol
function update(Info storage self, uint128 liquidityDelta) internal {
    uint128 liquidityBefore = self.liquidity;
    uint128 liquidityAfter = liquidityBefore + liquidityDelta;

    self.liquidity = liquidityAfter;
}
```
틱 업데이트 함수와 유사하게, 특정 포지션에 유동성을 추가합니다. 포지션을 얻기 위해 다음을 호출합니다:
```solidity
// src/libs/Position.sol
...
function get(
    mapping(bytes32 => Info) storage self,
    address owner,
    int24 lowerTick,
    int24 upperTick
) internal view returns (Position.Info storage position) {
    position = self[
        keccak256(abi.encodePacked(owner, lowerTick, upperTick))
    ];
}
...
```

각 포지션은 세 가지 키(소유자 주소, 하위 틱 인덱스, 상위 틱 인덱스)로 고유하게 식별됩니다. 데이터를 더 저렴하게 저장하기 위해 세 가지를 해시합니다. 해시되면 각 키는 32바이트를 차지하는 반면, `owner`, `lowerTick`, `upperTick`이 별도의 키일 때는 96바이트를 차지합니다.

> 세 개의 키를 사용하면 세 개의 매핑이 필요합니다. 각 키는 별도로 저장되며, Solidity는 (패킹이 적용되지 않은 경우) 값을 32바이트 슬롯에 저장하므로 32바이트를 차지합니다.

다음으로, 민팅을 계속하면서, 사용자가 예치해야 하는 양을 계산해야 합니다. 다행히도, 이전 파트에서 이미 공식을 알아내고 정확한 양을 계산했습니다. 따라서, 하드 코딩할 것입니다:

```solidity
amount0 = 0.998976618347425280 ether;
amount1 = 5000 ether;
```

> 나중에 실제 계산으로 대체할 것입니다.

또한 추가되는 `amount`를 기반으로 풀의 `liquidity`를 업데이트합니다.

```solidity
liquidity += uint128(amount);
```

이제 사용자로부터 토큰을 받을 준비가 되었습니다. 이것은 콜백을 통해 수행됩니다:
```solidity
function mint(...) ... {
    ...

    uint256 balance0Before;
    uint256 balance1Before;
    if (amount0 > 0) balance0Before = balance0();
    if (amount1 > 0) balance1Before = balance1();
    IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(
        amount0,
        amount1
    );
    if (amount0 > 0 && balance0Before + amount0 > balance0())
        revert InsufficientInputAmount();
    if (amount1 > 0 && balance1Before + amount1 > balance1())
        revert InsufficientInputAmount();

    ...
}

function balance0() internal returns (uint256 balance) {
    balance = IERC20(token0).balanceOf(address(this));
}

function balance1() internal returns (uint256 balance) {
    balance = IERC20(token1).balanceOf(address(this));
}
```

먼저, 현재 토큰 잔액을 기록합니다. 그런 다음 호출자에게서 `uniswapV3MintCallback` 메서드를 호출합니다. 이것이 콜백입니다. 호출자(`mint`를 호출하는 사람)는 계약이어야 합니다. 왜냐하면 비계약 주소는 이더리움에서 함수를 구현할 수 없기 때문입니다. 여기서 콜백을 사용하는 것은 전혀 사용자 친화적이지 않지만, 계약이 현재 상태를 사용하여 토큰 양을 계산할 수 있도록 합니다. 이는 사용자를 신뢰할 수 없기 때문에 매우 중요합니다.

호출자는 `uniswapV3MintCallback`을 구현하고 이 함수에서 풀 계약으로 토큰을 전송해야 합니다. 콜백 함수를 호출한 후, 풀 계약 잔액이 변경되었는지 여부를 계속 확인합니다. 각각 `amount0` 및 `amount1` 이상 증가해야 합니다. 이는 호출자가 토큰을 풀로 전송했음을 의미합니다.

마지막으로, `Mint` 이벤트를 발생시킵니다:
```solidity
emit Mint(msg.sender, owner, lowerTick, upperTick, amount, amount0, amount1);
```

이벤트는 이더리움에서 나중에 검색할 수 있도록 계약 데이터가 인덱싱되는 방식입니다. 계약 상태가 변경될 때마다 블록체인 탐색기가 이를 알 수 있도록 이벤트를 발생시키는 것이 좋은 관행입니다. 이벤트는 유용한 정보도 전달합니다. 우리의 경우, 호출자의 주소, 유동성 포지션 소유자의 주소, 상위 및 하위 틱, 새로운 유동성, 토큰 양입니다. 이 정보는 로그로 저장되며, 다른 사람은 모든 계약 이벤트를 수집하고 모든 블록과 트랜잭션을 탐색하고 분석하지 않고도 계약의 활동을 재현할 수 있습니다.

이제 완료되었습니다! 휴! 이제 민팅을 테스트해 봅시다.

## 테스팅

이 시점에서 모든 것이 올바르게 작동하는지 알 수 없습니다. 계약을 어디든 배포하기 전에 계약이 올바르게 작동하는지 확인하기 위해 많은 테스트를 작성할 것입니다. 다행히도, Forge는 훌륭한 테스팅 프레임워크이며 테스팅을 매우 쉽게 만들어 줄 것입니다.

새로운 테스트 파일을 생성합니다:
```solidity
// test/UniswapV3Pool.t.sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "forge-std/Test.sol";

contract UniswapV3PoolTest is Test {
    function setUp() public {}

    function testExample() public {
        assertTrue(true);
    }
}
```

실행해 봅시다:
```shell
$ forge test
Running 1 test for test/UniswapV3Pool.t.sol:UniswapV3PoolTest
[PASS] testExample() (gas: 279)
Test result: ok. 1 passed; 0 failed; finished in 5.07ms
```

통과했습니다! 물론 그렇습니다! 지금까지 우리의 테스트는 `true`가 `true`인지 확인하는 것뿐입니다!

테스트 계약은 `forge-std/Test.sol`에서 상속받는 계약일 뿐입니다. 이 계약은 테스팅 유틸리티 세트이며, 단계별로 익숙해질 것입니다. 기다리고 싶지 않다면, `lib/forge-std/src/Test.sol`을 열고 훑어보십시오.

테스트 계약은 특정 규칙을 따릅니다:
1. `setUp` 함수는 테스트 케이스를 설정하는 데 사용됩니다. 각 테스트 케이스에서 배포된 계약, 민팅된 토큰, 초기화된 풀과 같은 구성된 환경을 원합니다. 이 모든 것을 `setUp`에서 수행할 것입니다.
2. 모든 테스트 케이스는 `test` 접두사로 시작합니다 (예: `testMint()`). 이렇게 하면 Forge가 테스트 케이스를 헬퍼 함수와 구별할 수 있습니다 (원하는 함수를 가질 수도 있습니다).

이제 실제로 민팅을 테스트해 봅시다.

### 테스트 토큰

민팅을 테스트하려면 토큰이 필요합니다. 테스트에서 어떤 계약이든 배포할 수 있기 때문에 이것은 문제가 되지 않습니다! 게다가, Forge는 오픈 소스 계약을 종속성으로 설치할 수 있습니다. 특히, 민팅 기능을 갖춘 ERC20 계약이 필요합니다. 가스 최적화된 계약 모음인 [Solmate](https://github.com/Rari-Capital/solmate)의 ERC20 계약을 사용하고, Solmate 계약에서 상속받고 민팅을 공개적으로 노출하는 (기본적으로 공개) ERC20 계약을 만들 것입니다.

`solmate`를 설치해 봅시다:
```shell
$ forge install rari-capital/solmate
```

그런 다음, `test` 폴더에 `ERC20Mintable.sol` 계약을 생성해 봅시다 (계약은 테스트에서만 사용할 것입니다):
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "solmate/tokens/ERC20.sol";

contract ERC20Mintable is ERC20 {
    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals
    ) ERC20(_name, _symbol, _decimals) {}

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}
```

우리의 `ERC20Mintable`은 `solmate/tokens/ERC20.sol`의 모든 기능을 상속받고, 추가적으로 임의의 수의 토큰을 민팅할 수 있도록 해주는 공개 `mint` 메서드를 구현합니다.

### 민팅

이제 민팅을 테스트할 준비가 되었습니다.

먼저, 필요한 모든 계약을 배포해 봅시다:
```solidity
// test/UniswapV3Pool.t.sol
...
import "./ERC20Mintable.sol";
import "../src/UniswapV3Pool.sol";

contract UniswapV3PoolTest is Test {
    ERC20Mintable token0;
    ERC20Mintable token1;
    UniswapV3Pool pool;

    function setUp() public {
        token0 = new ERC20Mintable("Ether", "ETH", 18);
        token1 = new ERC20Mintable("USDC", "USDC", 18);
    }

    ...
```
`setUp` 함수에서는 토큰을 배포하지만 풀은 배포하지 않습니다! 모든 테스트 케이스가 동일한 토큰을 사용하지만 각 테스트 케이스마다 고유한 풀을 갖기 때문입니다.

풀 설정을 더 깔끔하고 간단하게 만들기 위해, 테스트 케이스 매개 변수 집합을 사용하는 별도의 함수 `setupTestCase`에서 이를 수행할 것입니다. 첫 번째 테스트 케이스에서는 성공적인 유동성 민팅을 테스트할 것입니다. 이것이 테스트 케이스 매개 변수의 모습입니다:
```solidity
function testMintSuccess() public {
    TestCaseParams memory params = TestCaseParams({
        wethBalance: 1 ether,
        usdcBalance: 5000 ether,
        currentTick: 85176,
        lowerTick: 84222,
        upperTick: 86129,
        liquidity: 1517882343751509868544,
        currentSqrtP: 5602277097478614198912276234240,
        shouldTransferInCallback: true,
        mintLiqudity: true
    });
```
1. 1 ETH와 5000 USDC를 풀에 예치할 계획입니다.
2. 현재 틱이 85176이고, 하위 및 상위 틱이 각각 84222 및 86129가 되기를 원합니다 (이 값들은 이전 장에서 계산했습니다).
3. 미리 계산된 유동성 및 현재 $\sqrt{P}$를 지정합니다.
4. 또한 유동성을 예치하고 (`mintLiquidity` 매개 변수) 풀 계약에서 요청할 때 토큰을 전송하기를 원합니다 (`shouldTransferInCallback`). 각 테스트 케이스에서 이를 수행하고 싶지 않으므로, 플래그를 갖기를 원합니다.

다음으로, 위의 매개 변수로 `setupTestCase`를 호출합니다:
```solidity
function setupTestCase(TestCaseParams memory params)
    internal
    returns (uint256 poolBalance0, uint256 poolBalance1)
{
    token0.mint(address(this), params.wethBalance);
    token1.mint(address(this), params.usdcBalance);

    pool = new UniswapV3Pool(
        address(token0),
        address(token1),
        params.currentSqrtP,
        params.currentTick
    );

    if (params.mintLiqudity) {
        (poolBalance0, poolBalance1) = pool.mint(
            address(this),
            params.lowerTick,
            params.upperTick,
            params.liquidity
        );
    }

    shouldTransferInCallback = params.shouldTransferInCallback;
}
```
이 함수에서는 토큰을 민팅하고 풀을 배포합니다. 또한, `mintLiquidity` 플래그가 설정되면, 풀에 유동성을 민팅합니다. 마지막으로, 민트 콜백에서 읽을 수 있도록 `shouldTransferInCallback` 플래그를 설정합니다:
```solidity
function uniswapV3MintCallback(uint256 amount0, uint256 amount1) public {
    if (shouldTransferInCallback) {
        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);
    }
}
```
유동성을 제공하고 풀에서 `mint` 함수를 호출할 테스트 계약입니다. 사용자는 없습니다. 테스트 계약은 사용자의 역할을 하므로 민트 콜백 함수를 구현할 수 있습니다.

이와 같이 테스트 케이스를 설정하는 것은 필수는 아니며, 가장 편안하게 느끼는 방식으로 수행할 수 있습니다. 테스트 계약은 계약일 뿐입니다.

`testMintSuccess`에서, 풀 계약이 다음을 보장하기를 원합니다:
1. 우리로부터 올바른 양의 토큰을 가져갑니다.
2. 올바른 키와 유동성으로 포지션을 생성합니다.
3. 지정한 상위 및 하위 틱을 초기화합니다.
4. 올바른 $\sqrt{P}$ 및 $L$을 갖습니다.

해 봅시다.

민팅은 `setupTestCase`에서 발생하므로 다시 수행할 필요가 없습니다. 함수는 또한 우리가 제공한 양을 반환하므로, 확인해 봅시다:
```solidity
(uint256 poolBalance0, uint256 poolBalance1) = setupTestCase(params);

uint256 expectedAmount0 = 0.998976618347425280 ether;
uint256 expectedAmount1 = 5000 ether;
assertEq(
    poolBalance0,
    expectedAmount0,
    "잘못된 토큰0 예치 금액"
);
assertEq(
    poolBalance1,
    expectedAmount1,
    "잘못된 토큰1 예치 금액"
);
```
특정 미리 계산된 양을 예상합니다. 그리고 이 양들이 풀로 전송되었는지 확인할 수도 있습니다:
```solidity
assertEq(token0.balanceOf(address(pool)), expectedAmount0);
assertEq(token1.balanceOf(address(pool)), expectedAmount1);
```

다음으로, 풀이 우리를 위해 생성한 포지션을 확인해야 합니다. `positions` 매핑의 키는 해시라는 것을 기억하십시오. 수동으로 계산한 다음 계약에서 포지션을 가져와야 합니다:
```solidity
bytes32 positionKey = keccak256(
    abi.encodePacked(address(this), params.lowerTick, params.upperTick)
);
uint128 posLiquidity = pool.positions(positionKey);
assertEq(posLiquidity, params.liquidity);
```

> `Position.Info`는 [구조체](https://docs.soliditylang.org/en/latest/types.html#structs)이므로, 가져올 때 구조 분해됩니다. 각 필드는 별도의 변수에 할당됩니다.

다음은 틱입니다. 다시 말하지만, 간단합니다:
```solidity
(bool tickInitialized, uint128 tickLiquidity) = pool.ticks(
    params.lowerTick
);
assertTrue(tickInitialized);
assertEq(tickLiquidity, params.liquidity);

(tickInitialized, tickLiquidity) = pool.ticks(params.upperTick);
assertTrue(tickInitialized);
assertEq(tickLiquidity, params.liquidity);
```

마지막으로, $\sqrt{P}$ 및 $L$:
```solidity
(uint160 sqrtPriceX96, int24 tick) = pool.slot0();
assertEq(
    sqrtPriceX96,
    5602277097478614198912276234240,
    "잘못된 현재 sqrtP"
);
assertEq(tick, 85176, "잘못된 현재 틱");
assertEq(
    pool.liquidity(),
    1517882343751509868544,
    "잘못된 현재 유동성"
);
```

보시다시피, Solidity에서 테스트를 작성하는 것은 어렵지 않습니다!

### 실패

물론, 성공적인 시나리오만 테스트하는 것으로는 충분하지 않습니다. 실패하는 경우도 테스트해야 합니다. 유동성을 제공할 때 무엇이 잘못될 수 있을까요? 몇 가지 힌트가 있습니다:
1. 상위 및 하위 틱이 너무 크거나 너무 작습니다.
2. 유동성이 0으로 제공됩니다.
3. 유동성 공급자가 충분한 토큰을 가지고 있지 않습니다.

이러한 시나리오를 구현하는 것은 여러분에게 맡기겠습니다! [저장소의 코드](https://github.com/Jeiwan/uniswapv3-code/blob/milestone_1/test/UniswapV3Pool.t.sol)를 자유롭게 살펴보십시오.