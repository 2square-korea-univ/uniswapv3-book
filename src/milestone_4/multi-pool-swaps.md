## 멀티풀 스왑

이제 이번 마일스톤의 핵심인 컨트랙트에서 멀티풀 스왑을 구현하는 과정을 진행합니다. 풀 컨트랙트는 핵심 기능만 구현해야 하는 핵심 컨트랙트이므로 이번 마일스톤에서는 건드리지 않을 것입니다. 멀티풀 스왑은 유틸리티 기능이며, 매니저 및 쿼터 컨트랙트에 구현할 것입니다.

## 매니저 컨트랙트 업데이트

### 싱글풀 및 멀티풀 스왑
현재 구현에서 매니저 컨트랙트의 `swap` 함수는 싱글풀 스왑만 지원하며 파라미터로 풀 주소를 받습니다.

```solidity
function swap(
    address poolAddress_,
    bool zeroForOne,
    uint256 amountSpecified,
    uint160 sqrtPriceLimitX96,
    bytes calldata data
) public returns (int256, int256) { ... }
```

이를 두 개의 함수, 즉 싱글풀 스왑과 멀티풀 스왑으로 분리할 것입니다. 이 함수들은 서로 다른 파라미터 집합을 갖게 됩니다.

```solidity
struct SwapSingleParams {
    address tokenIn;
    address tokenOut;
    uint24 tickSpacing;
    uint256 amountIn;
    uint160 sqrtPriceLimitX96;
}

struct SwapParams {
    bytes path;
    address recipient;
    uint256 amountIn;
    uint256 minAmountOut;
}
```

1. `SwapSingleParams`는 풀 파라미터, 입력 금액, 제한 가격을 파라미터로 받습니다. 이는 이전과 거의 동일합니다. `data`는 더 이상 필요하지 않습니다.
2. `SwapParams`는 경로, 출력 금액 수령인, 입력 금액, 최소 출력 금액을 파라미터로 받습니다. 마지막 파라미터는 `sqrtPriceLimitX96`를 대체합니다. 멀티풀 스왑을 수행할 때는 풀 컨트랙트의 슬리피지 보호(제한 가격 사용)를 사용할 수 없기 때문입니다. 다른 슬리피지 보호를 구현해야 합니다. 이는 최종 출력 금액을 확인하고 `minAmountOut`과 비교합니다. 최종 출력 금액이 `minAmountOut`보다 작으면 슬리피지 보호가 실패합니다.

### 핵심 스왑 로직

싱글풀 및 멀티풀 스왑 함수 모두에서 호출될 내부 `_swap` 함수를 구현해 보겠습니다. 이 함수는 파라미터를 준비하고 `Pool.swap`을 호출합니다.

```solidity
function _swap(
    uint256 amountIn,
    address recipient,
    uint160 sqrtPriceLimitX96,
    SwapCallbackData memory data
) internal returns (uint256 amountOut) {
    ...
```

`SwapCallbackData`는 스왑 함수와 `uniswapV3SwapCallback` 간에 전달하는 데이터를 포함하는 새로운 데이터 구조입니다.
```solidity
struct SwapCallbackData {
    bytes path;
    address payer;
}
```

`path`는 스왑 경로이고 `payer`는 스왑에서 입력 토큰을 제공하는 주소입니다. 멀티풀 스왑 중에는 다른 지불자가 있게 됩니다.

`_swap`에서 가장 먼저 하는 일은 `Path` 라이브러리를 사용하여 풀 파라미터를 추출하는 것입니다.

```solidity
// function _swap(...) {
(address tokenIn, address tokenOut, uint24 tickSpacing) = data
    .path
    .decodeFirstPool();
```

그런 다음 스왑 방향을 식별합니다.

```solidity
bool zeroForOne = tokenIn < tokenOut;
```

그런 다음 실제 스왑을 수행합니다.
```solidity
// function _swap(...) {
(int256 amount0, int256 amount1) = getPool(
    tokenIn,
    tokenOut,
    tickSpacing
).swap(
        recipient,
        zeroForOne,
        amountIn,
        sqrtPriceLimitX96 == 0
            ? (
                zeroForOne
                    ? TickMath.MIN_SQRT_RATIO + 1
                    : TickMath.MAX_SQRT_RATIO - 1
            )
            : sqrtPriceLimitX96,
        abi.encode(data)
    );
```

이 부분은 이전과 동일하지만 이번에는 `getPool`을 호출하여 풀을 찾습니다. `getPool`은 토큰을 정렬하고 `PoolAddress.computeAddress`를 호출하는 함수입니다.

```solidity
function getPool(
    address token0,
    address token1,
    uint24 tickSpacing
) internal view returns (IUniswapV3Pool pool) {
    (token0, token1) = token0 < token1
        ? (token0, token1)
        : (token1, token0);
    pool = IUniswapV3Pool(
        PoolAddress.computeAddress(factory, token0, token1, tickSpacing)
    );
}
```

스왑을 수행한 후에는 어떤 금액이 출력 금액인지 파악해야 합니다.
```solidity
// function _swap(...) {
amountOut = uint256(-(zeroForOne ? amount1 : amount0));
```

이것으로 끝입니다. 이제 싱글풀 스왑이 어떻게 작동하는지 살펴보겠습니다.

### 싱글풀 스왑

`swapSingle`은 `_swap`의 래퍼 역할을 합니다.

```solidity
function swapSingle(SwapSingleParams calldata params)
    public
    returns (uint256 amountOut)
{
    amountOut = _swap(
        params.amountIn,
        msg.sender,
        params.sqrtPriceLimitX96,
        SwapCallbackData({
            path: abi.encodePacked(
                params.tokenIn,
                params.tickSpacing,
                params.tokenOut
            ),
            payer: msg.sender
        })
    );
}
```

여기서 단일 풀 경로를 빌드하고 있습니다. 싱글풀 스왑은 풀이 하나인 멀티풀 스왑입니다 🙂.

### 멀티풀 스왑

멀티풀 스왑은 싱글풀 스왑보다 약간 더 어렵습니다. 살펴보겠습니다.

```solidity
function swap(SwapParams memory params) public returns (uint256 amountOut) {
    address payer = msg.sender;
    bool hasMultiplePools;
    ...
```

첫 번째 스왑은 사용자가 입력 토큰을 제공하므로 사용자가 지불합니다.

그런 다음 경로의 풀을 반복하기 시작합니다.

```solidity
...
while (true) {
    hasMultiplePools = params.path.hasMultiplePools();

    params.amountIn = _swap(
        params.amountIn,
        hasMultiplePools ? address(this) : params.recipient,
        0,
        SwapCallbackData({
            path: params.path.getFirstPool(),
            payer: payer
        })
    );
    ...
```

각 반복에서 다음과 같은 파라미터로 `_swap`을 호출합니다.
1. `params.amountIn`은 입력 금액을 추적합니다. 첫 번째 스왑 동안에는 사용자가 제공한 금액입니다. 다음 스왑 동안에는 이전 스왑에서 반환된 금액입니다.
2. `hasMultiplePools ? address(this) : params.recipient` – 경로에 여러 풀이 있는 경우 수령인은 매니저 컨트랙트이며, 스왑 사이에 토큰을 저장합니다. 경로에 풀이 하나만 있는 경우(마지막 풀), 수령인은 파라미터에 지정된 수령인입니다(일반적으로 스왑을 시작하는 동일한 사용자).
3. `sqrtPriceLimitX96`은 0으로 설정되어 풀 컨트랙트에서 슬리피지 보호를 비활성화합니다.
4. 마지막 파라미터는 `uniswapV3SwapCallback`에 전달하는 것입니다. 곧 살펴보겠습니다.

하나의 스왑을 수행한 후에는 경로에서 다음 풀로 진행하거나 반환해야 합니다.
```solidity
    ...

    if (hasMultiplePools) {
        payer = address(this);
        params.path = params.path.skipToken();
    } else {
        amountOut = params.amountIn;
        break;
    }
}
```

여기서 지불자를 변경하고 처리된 풀을 경로에서 제거합니다.

마지막으로 새로운 슬리피지 보호입니다.

```solidity
if (amountOut < params.minAmountOut)
    revert TooLittleReceived(amountOut);
```

### 스왑 콜백

업데이트된 스왑 콜백을 살펴보겠습니다.

```solidity
function uniswapV3SwapCallback(
    int256 amount0,
    int256 amount1,
    bytes calldata data_
) public {
    SwapCallbackData memory data = abi.decode(data_, (SwapCallbackData));
    (address tokenIn, address tokenOut, ) = data.path.decodeFirstPool();

    bool zeroForOne = tokenIn < tokenOut;

    int256 amount = zeroForOne ? amount0 : amount1;

    if (data.payer == address(this)) {
        IERC20(tokenIn).transfer(msg.sender, uint256(amount));
    } else {
        IERC20(tokenIn).transferFrom(
            data.payer,
            msg.sender,
            uint256(amount)
        );
    }
}
```

콜백은 경로와 지불자 주소가 인코딩된 `SwapCallbackData`를 예상합니다. 경로에서 풀 토큰을 추출하고, 스왑 방향(`zeroForOne`)과 컨트랙트가 전송해야 하는 금액을 파악합니다. 그런 다음 지불자 주소에 따라 다르게 작동합니다.
1. 지불자가 현재 컨트랙트인 경우(연속 스왑을 수행할 때), 현재 컨트랙트의 잔액에서 다음 풀(이 콜백을 호출한 풀)로 토큰을 전송합니다.
2. 지불자가 다른 주소인 경우(스왑을 시작한 사용자), 사용자 잔액에서 토큰을 전송합니다.

## 쿼터 컨트랙트 업데이트

쿼터는 멀티풀 스왑에서 출력 금액을 찾는 데 사용하고 싶기 때문에 업데이트해야 하는 또 다른 컨트랙트입니다. 매니저와 마찬가지로 `quote` 함수의 두 가지 변형, 즉 싱글풀과 멀티풀 변형이 있습니다. 먼저 전자를 살펴보겠습니다.

### 싱글풀 견적
현재 `quote` 구현에서 몇 가지 변경 사항만 적용하면 됩니다.
1. 이름을 `quoteSingle`로 변경합니다.
2. 파라미터를 구조체로 추출합니다(주로 외형적인 변경).
3. 풀 주소 대신 파라미터에 두 개의 토큰 주소와 틱 간격을 사용합니다.

```solidity
// src/UniswapV3Quoter.sol
struct QuoteSingleParams {
    address tokenIn;
    address tokenOut;
    uint24 tickSpacing;
    uint256 amountIn;
    uint160 sqrtPriceLimitX96;
}

function quoteSingle(QuoteSingleParams memory params)
    public
    returns (
        uint256 amountOut,
        uint160 sqrtPriceX96After,
        int24 tickAfter
    )
{
    ...
```

함수 본문에서 유일하게 변경된 사항은 `getPool`을 사용하여 풀 주소를 찾는 것입니다.
```solidity
    ...
    IUniswapV3Pool pool = getPool(
        params.tokenIn,
        params.tokenOut,
        params.tickSpacing
    );

    bool zeroForOne = params.tokenIn < params.tokenOut;
    ...
```

### 멀티풀 견적

멀티풀 견적 구현은 멀티풀 스왑 구현과 유사하지만 더 적은 파라미터를 사용합니다.

```solidity
function quote(bytes memory path, uint256 amountIn)
    public
    returns (
        uint256 amountOut,
        uint160[] memory sqrtPriceX96AfterList,
        int24[] memory tickAfterList
    )
{
    sqrtPriceX96AfterList = new uint160[](path.numPools());
    tickAfterList = new int24[](path.numPools());
    ...
```

파라미터로는 입력 금액과 스왑 경로만 필요합니다. 함수는 `quoteSingle`과 유사한 값을 반환하지만 "price after" 및 "tick after"는 각 스왑 후에 수집되므로 배열을 반환해야 합니다.

```solidity
uint256 i = 0;
while (true) {
    (address tokenIn, address tokenOut, uint24 tickSpacing) = path
        .decodeFirstPool();

    (
        uint256 amountOut_,
        uint160 sqrtPriceX96After,
        int24 tickAfter
    ) = quoteSingle(
            QuoteSingleParams({
                tokenIn: tokenIn,
                tokenOut: tokenOut,
                tickSpacing: tickSpacing,
                amountIn: amountIn,
                sqrtPriceLimitX96: 0
            })
        );

    sqrtPriceX96AfterList[i] = sqrtPriceX96After;
    tickAfterList[i] = tickAfter;
    amountIn = amountOut_;
    i++;

    if (path.hasMultiplePools()) {
        path = path.skipToken();
    } else {
        amountOut = amountIn;
        break;
    }
}
```

루프의 로직은 업데이트된 `swap` 함수의 로직과 동일합니다.
1. 현재 풀의 파라미터를 가져옵니다.
2. 현재 풀에서 `quoteSingle`을 호출합니다.
3. 반환된 값을 저장합니다.
4. 경로에 더 많은 풀이 있으면 반복하고, 그렇지 않으면 반환합니다.