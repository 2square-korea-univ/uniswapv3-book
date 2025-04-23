## Manager Contract

풀 컨트랙트를 배포하기 전에, 한 가지 문제를 해결해야 합니다. 기억하시겠지만, Uniswap V3 컨트랙트는 두 가지 범주로 나뉩니다:
1. 핵심 기능들을 구현하고 사용자 친화적인 인터페이스를 제공하지 않는 **코어 컨트랙트**.
2. 코어 컨트랙트를 위한 사용자 친화적인 인터페이스를 구현하는 **주변부 컨트랙트**.

풀 컨트랙트는 코어 컨트랙트이며, 사용자 친화적이거나 유연하도록 설계되지 않았습니다. 풀 컨트랙트는 호출자가 모든 계산 (가격, 수량)을 수행하고 적절한 호출 매개변수를 제공할 것을 기대합니다. 또한 호출자로부터 토큰을 전송하기 위해 ERC20의 `transferFrom`을 사용하지 않습니다. 대신, 두 가지 콜백을 사용합니다:
1. 유동성을 민팅할 때 호출되는 `uniswapV3MintCallback`;
2. 토큰을 스왑할 때 호출되는 `uniswapV3SwapCallback`.

우리의 테스트에서는 이러한 콜백들을 테스트 컨트랙트 내에서 구현했습니다. 콜백들을 구현할 수 있는 것은 컨트랙트뿐이므로, 풀 컨트랙트는 일반 사용자 (컨트랙트 주소가 아닌 주소)에 의해 호출될 수 없습니다. 이것은 괜찮았습니다. 하지만 더 이상은 아닙니다 🙂.

이 책의 다음 단계는 풀 컨트랙트를 로컬 블록체인에 배포하고 프론트엔드 앱에서 상호작용하는 것입니다. 따라서, 컨트랙트 주소가 아닌 주소들이 풀과 상호작용할 수 있도록 하는 컨트랙트를 구축해야 합니다. 지금 바로 시작해 봅시다!

## 워크플로우

매니저 컨트랙트의 작동 방식은 다음과 같습니다:
1. 유동성을 민팅하기 위해, 매니저 컨트랙트가 토큰을 사용할 수 있도록 승인합니다.
2. 그 다음 매니저 컨트랙트의 `mint` 함수를 호출하고, 민팅 매개변수와 유동성을 제공하려는 풀의 주소를 전달합니다.
3. 매니저 컨트랙트는 풀의 `mint` 함수를 호출하고 `uniswapV3MintCallback`을 구현합니다. 매니저 컨트랙트는 우리의 토큰을 풀 컨트랙트로 보낼 권한을 가지게 됩니다.
4. 토큰을 스왑하기 위해, 마찬가지로 매니저 컨트랙트가 토큰을 사용할 수 있도록 승인합니다.
5. 그 다음 매니저 컨트랙트의 `swap` 함수를 호출하고, 민팅과 유사하게 풀에게 호출을 전달합니다.
매니저 컨트랙트는 우리의 토큰을 풀 컨트랙트로 보내고, 풀 컨트랙트는 토큰을 스왑한 후 결과 수량을 우리에게 보냅니다.

따라서, 매니저 컨트랙트는 사용자와 풀 사이의 중개자 역할을 수행합니다.

## 콜백에 데이터 전달하기

매니저 컨트랙트를 구현하기 전에, 풀 컨트랙트를 업그레이드해야 합니다.

매니저 컨트랙트는 모든 풀과 작동하며, 모든 주소가 매니저 컨트랙트를 호출할 수 있도록 할 것입니다. 이를 달성하기 위해, 콜백을 업그레이드해야 합니다: 서로 다른 풀 주소와 사용자 주소를 콜백에 전달하고 싶습니다. 현재 `uniswapV3MintCallback` 구현 (테스트 컨트랙트 내)을 살펴봅시다:
```solidity
function uniswapV3MintCallback(uint256 amount0, uint256 amount1) public {
    if (transferInMintCallback) {
        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);
    }
}
```

여기서 핵심은 다음과 같습니다:
1. 함수는 테스트 컨트랙트에 속한 토큰을 전송합니다. `transferFrom`을 사용하여 호출자로부터 토큰을 전송하도록 변경하고 싶습니다.
2. 함수는 `token0`과 `token1`을 알고 있지만, 이는 모든 풀마다 다를 것입니다.

아이디어: 사용자 및 풀 주소를 전달할 수 있도록 콜백의 인수를 변경해야 합니다.

이제 스왑 콜백을 살펴봅시다:
```solidity
function uniswapV3SwapCallback(int256 amount0, int256 amount1) public {
    if (amount0 > 0 && transferInSwapCallback) {
        token0.transfer(msg.sender, uint256(amount0));
    }

    if (amount1 > 0 && transferInSwapCallback) {
        token1.transfer(msg.sender, uint256(amount1));
    }
}
```

동일하게, 테스트 컨트랙트로부터 토큰을 전송하고 `token0`과 `token1`을 알고 있습니다.

추가 데이터를 콜백에 전달하려면, 먼저 `mint`와 `swap`에 데이터를 전달해야 합니다 (콜백은 이 함수들로부터 호출되기 때문입니다). 그러나 이 추가 데이터는 함수 내에서 사용되지 않고 함수의 인수를 복잡하게 만들지 않기 위해, [abi.encode()](https://docs.soliditylang.org/en/latest/units-and-global-variables.html?highlight=abi.encode#abi-encoding-and-decoding-functions)를 사용하여 추가 데이터를 인코딩할 것입니다.

추가 데이터를 구조체로 정의해 봅시다:
```solidity
// src/UniswapV3Pool.sol
...
struct CallbackData {
    address token0;
    address token1;
    address payer;
}
...
```

그리고 인코딩된 데이터를 콜백에 전달합니다:
```solidity
function mint(
    address owner,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount,
    bytes calldata data // <--- 새로운 라인
) external returns (uint256 amount0, uint256 amount1) {
    ...
    IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(
        amount0,
        amount1,
        data // <--- 새로운 라인
    );
    ...
}

function swap(address recipient, bytes calldata data) // <--- `data` 추가됨
    public
    returns (int256 amount0, int256 amount1)
{
    ...
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
        amount0,
        amount1,
        data // <--- 새로운 라인
    );
    ...
}
```

이제 테스트 컨트랙트의 콜백에서 추가 데이터를 읽을 수 있습니다.
```solidity
function uniswapV3MintCallback(
    uint256 amount0,
    uint256 amount1,
    bytes calldata data
) public {
    if (transferInMintCallback) {
        UniswapV3Pool.CallbackData memory extra = abi.decode(
            data,
            (UniswapV3Pool.CallbackData)
        );

        IERC20(extra.token0).transferFrom(extra.payer, msg.sender, amount0);
        IERC20(extra.token1).transferFrom(extra.payer, msg.sender, amount1);
    }
}
```

나머지 코드를 스스로 업데이트해 보세요. 너무 어렵다면, [이 커밋](https://github.com/Jeiwan/uniswapv3-code/commit/cda23134fd12a190aaeebe718786545621e16c0e)을 참고하셔도 좋습니다.

## 매니저 컨트랙트 구현하기

콜백을 구현하는 것 외에, 매니저 컨트랙트는 많은 일을 하지는 않을 것입니다: 단순히 풀 컨트랙트로 호출을 리디렉션할 것입니다. 이것은 현재 매우 최소한의 컨트랙트입니다:
```solidity
pragma solidity ^0.8.14;

import "../src/UniswapV3Pool.sol";
import "../src/interfaces/IERC20.sol";

contract UniswapV3Manager {
    function mint(
        address poolAddress_,
        int24 lowerTick,
        int24 upperTick,
        uint128 liquidity,
        bytes calldata data
    ) public {
        UniswapV3Pool(poolAddress_).mint(
            msg.sender,
            lowerTick,
            upperTick,
            liquidity,
            data
        );
    }

    function swap(address poolAddress_, bytes calldata data) public {
        UniswapV3Pool(poolAddress_).swap(msg.sender, data);
    }

    function uniswapV3MintCallback(...) {...}
    function uniswapV3SwapCallback(...) {...}
}
```

콜백은 테스트 컨트랙트의 콜백과 동일하지만, 매니저 컨트랙트는 항상 토큰을 전송하므로 `transferInMintCallback` 및 `transferInSwapCallback` 플래그가 없다는 점이 다릅니다.

자, 이제 풀 컨트랙트를 배포하고 프론트엔드 앱과 통합할 준비가 완전히 되었습니다!