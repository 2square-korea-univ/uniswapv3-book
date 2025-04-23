# 슬리피지 보호

슬리피지는 탈중앙화 거래소에서 매우 중요한 문제입니다. 슬리피지란 간단히 말해, 트랜잭션을 시작할 때 화면에서 보는 가격과 스왑이 실제로 실행될 때의 가격 간의 차이를 의미합니다. 이러한 차이는 트랜잭션을 보내는 시점과 트랜잭션이 채굴되는 시점 사이에 짧은 (그리고 네트워크 혼잡 및 가스 비용에 따라 때로는 긴) 지연이 발생하기 때문에 나타납니다. 더 기술적인 용어로 설명하자면, 블록체인 상태는 매 블록마다 변경되며 트랜잭션이 특정 블록에서 적용된다는 보장이 없기 때문입니다.

슬리피지 보호가 해결하는 또 다른 중요한 문제는 *샌드위치 공격*입니다. 이는 탈중앙화 거래소 사용자에게 흔히 발생하는 공격 유형입니다. 샌드위치 공격 동안 공격자는 사용자의 스왑 트랜잭션을 두 개의 트랜잭션으로 "감쌉니다". 하나는 사용자 트랜잭션보다 먼저 실행되고 다른 하나는 그 이후에 실행됩니다. 첫 번째 트랜잭션에서 공격자는 풀의 상태를 수정하여 사용자의 스왑이 사용자에게 매우 불리하고 공격자에게는 어느 정도 이득이 되도록 만듭니다. 이는 사용자의 거래가 더 낮은 가격에 체결되도록 풀 유동성을 조정함으로써 달성됩니다. 두 번째 트랜잭션에서 공격자는 풀 유동성과 가격을 재설정합니다. 결과적으로 사용자는 조작된 가격 때문에 예상보다 훨씬 적은 토큰을 받게 되며, 공격자는 약간의 이득을 얻습니다.



![샌드위치 공격](images/sandwich_attack.png)

탈중앙화 거래소에서 슬리피지 보호가 구현되는 방식은 사용자가 실제 가격이 얼마나 떨어지는 것을 허용할지 선택하도록 하는 것입니다. 기본적으로 Uniswap V3는 슬리피지 허용 오차를 0.1%로 설정합니다. 이는 실행 시점의 가격이 사용자가 브라우저에서 본 가격의 99.9%보다 작지 않은 경우에만 스왑이 실행됨을 의미합니다. 이는 매우 좁은 범위이며 사용자는 이 숫자를 조정할 수 있습니다. 이는 변동성이 높을 때 유용합니다.

구현에 슬리피지 보호를 추가해 봅시다!

## 스왑의 슬리피지 보호

스왑을 보호하기 위해 `swap` 함수에 매개변수를 하나 더 추가해야 합니다. 사용자가 스왑이 중지될 중지 가격을 선택할 수 있도록 하려고 합니다. 매개변수를 `sqrtPriceLimitX96`이라고 부르겠습니다.

```solidity
function swap(
    address recipient,
    bool zeroForOne,
    uint256 amountSpecified,
    uint160 sqrtPriceLimitX96,
    bytes calldata data
) public returns (int256 amount0, int256 amount1) {
    ...
    if (
        zeroForOne
            ? sqrtPriceLimitX96 > slot0_.sqrtPriceX96 ||
                sqrtPriceLimitX96 < TickMath.MIN_SQRT_RATIO
            : sqrtPriceLimitX96 < slot0_.sqrtPriceX96 &&
                sqrtPriceLimitX96 > TickMath.MAX_SQRT_RATIO
    ) revert InvalidPriceLimit();
    ...
```

토큰 $x$를 판매할 때(`zeroForOne`이 참인 경우), `sqrtPriceLimitX96`은 현재 가격과 최소 $\sqrt{P}$ 사이에 있어야 합니다. 왜냐하면 토큰 $x$를 판매하면 가격이 하락하기 때문입니다. 마찬가지로 토큰 $y$를 판매할 때, `sqrtPriceLimitX96`은 현재 가격과 최대 $\sqrt{P}$ 사이에 있어야 합니다. 왜냐하면 가격이 상승하기 때문입니다.

while 루프에서 두 가지 조건을 만족시키려고 합니다. 전체 스왑 수량이 채워지지 않았고 현재 가격이 `sqrtPriceLimitX96`과 같지 않은 것입니다.
```solidity
..
while (
    state.amountSpecifiedRemaining > 0 &&
    state.sqrtPriceX96 != sqrtPriceLimitX96
) {
...
```

이는 Uniswap V3 풀이 슬리피지 허용 오차에 도달했을 때 실패하지 않고 스왑을 부분적으로만 실행한다는 것을 의미합니다.

`sqrtPriceLimitX96`을 고려해야 할 또 다른 위치는 `SwapMath.computeSwapStep`을 호출할 때입니다.

```solidity
(state.sqrtPriceX96, step.amountIn, step.amountOut) = SwapMath
    .computeSwapStep(
        state.sqrtPriceX96,
        (
            zeroForOne
                ? step.sqrtPriceNextX96 < sqrtPriceLimitX96
                : step.sqrtPriceNextX96 > sqrtPriceLimitX96
        )
            ? sqrtPriceLimitX96
            : step.sqrtPriceNextX96,
        state.liquidity,
        state.amountSpecifiedRemaining
    );
```

여기서 `computeSwapStep`이 `sqrtPriceLimitX96` 범위를 벗어나 스왑 금액을 계산하지 않도록 보장하려고 합니다. 이는 현재 가격이 제한 가격을 넘지 않도록 보장합니다.

## 민팅의 슬리피지 보호

유동성을 추가하는 것 또한 슬리피지 보호가 필요합니다. 이는 유동성을 추가할 때 가격을 변경할 수 없다는 사실에서 비롯됩니다 (유동성은 현재 가격에 비례해야 함). 따라서 유동성 공급자 또한 슬리피지로 인해 손해를 볼 수 있습니다. 그러나 `swap` 함수와는 달리, 풀 컨트랙트에서 슬리피지 보호를 구현해야 할 의무는 없습니다. 풀 컨트랙트는 핵심 컨트랙트이며 불필요한 로직을 넣고 싶지 않다는 점을 상기하십시오. 이것이 바로 매니저 컨트랙트를 만든 이유이며, 슬리피지 보호를 구현할 곳은 매니저 컨트랙트입니다.

매니저 컨트랙트는 풀 컨트랙트에 대한 호출을 더 편리하게 만드는 래퍼 컨트랙트입니다. `mint` 함수에서 슬리피지 보호를 구현하려면 풀에서 가져온 토큰 양을 확인하고 이를 사용자가 선택한 최소 양과 비교하기만 하면 됩니다. 또한 사용자가 $\sqrt{P_{lower}}$ 및 $\sqrt{P_{upper}}$뿐만 아니라 유동성을 계산하는 부담을 덜어주고 `Manager.mint()`에서 이를 계산할 수 있습니다.

업데이트된 `mint` 함수는 이제 더 많은 매개변수를 사용하므로 구조체로 그룹화해 보겠습니다.
```solidity
// src/UniswapV3Manager.sol
contract UniswapV3Manager {
    struct MintParams {
        address poolAddress;
        int24 lowerTick;
        int24 upperTick;
        uint256 amount0Desired;
        uint256 amount1Desired;
        uint256 amount0Min;
        uint256 amount1Min;
    }

    function mint(MintParams calldata params)
        public
        returns (uint256 amount0, uint256 amount1)
    {
        ...
```

`amount0Min` 및 `amount1Min`은 슬리피지 허용 오차를 기반으로 계산된 양입니다. 이는 원하는 양보다 작아야 하며, 그 차이는 슬리피지 허용 설정에 의해 제어됩니다. 유동성 공급자는 `amount0Min` 및 `amount1Min`보다 작지 않은 양을 제공할 것으로 예상합니다.

다음으로 $\sqrt{P_{lower}}$, $\sqrt{P_{upper}}$ 및 유동성을 계산합니다.
```solidity
...
IUniswapV3Pool pool = IUniswapV3Pool(params.poolAddress);

(uint160 sqrtPriceX96, ) = pool.slot0();
uint160 sqrtPriceLowerX96 = TickMath.getSqrtRatioAtTick(
    params.lowerTick
);
uint160 sqrtPriceUpperX96 = TickMath.getSqrtRatioAtTick(
    params.upperTick
);

uint128 liquidity = LiquidityMath.getLiquidityForAmounts(
    sqrtPriceX96,
    sqrtPriceLowerX96,
    sqrtPriceUpperX96,
    params.amount0Desired,
    params.amount1Desired
);
...
```

`LiquidityMath.getLiquidityForAmounts`는 새로운 함수이며, 다음 장에서 논의될 것입니다.

다음 단계는 풀에 유동성을 제공하고 풀에서 반환된 양을 확인하는 것입니다. 너무 낮으면 되돌립니다.
```solidity
(amount0, amount1) = pool.mint(
    msg.sender,
    params.lowerTick,
    params.upperTick,
    liquidity,
    abi.encode(
        IUniswapV3Pool.CallbackData({
            token0: pool.token0(),
            token1: pool.token1(),
            payer: msg.sender
        })
    )
);

if (amount0 < params.amount0Min || amount1 < params.amount1Min)
    revert SlippageCheckFailed(amount0, amount1);
```

이것으로 완료되었습니다!