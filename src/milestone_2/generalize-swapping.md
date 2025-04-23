## 일반화된 스왑

이번 마일스톤에서 가장 어려운 챕터입니다. 코드를 업데이트하기 전에 Uniswap V3의 스왑 알고리즘이 어떻게 작동하는지 이해해야 합니다.

스왑은 주문 이행으로 생각할 수 있습니다. 사용자가 풀에서 특정 양의 토큰을 구매하는 주문을 제출합니다. 풀은 사용 가능한 유동성을 사용하여 입력된 양을 다른 토큰의 출력량으로 "변환"합니다. 현재 가격 범위에 충분한 유동성이 없다면, 다른 가격 범위에서 유동성을 찾으려고 시도합니다 (이전 챕터에서 구현한 함수 사용).

이제 `swap` 함수에서 이 로직을 구현할 것입니다. 하지만 당분간은 현재 가격 범위 내에서만 머무를 것입니다. 틱 간 스왑 (cross-tick swaps)은 다음 마일스톤에서 구현할 예정입니다.

```solidity
function swap(
    address recipient,
    bool zeroForOne,
    uint256 amountSpecified,
    bytes calldata data
) public returns (int256 amount0, int256 amount1) {
    ...
```

`swap` 함수에 두 개의 새로운 매개변수 `zeroForOne`과 `amountSpecified`를 추가합니다. `zeroForOne`은 스왑 방향을 제어하는 플래그입니다. `true`이면 `token0`을 `token1`으로 교환하고, `false`이면 반대 방향으로 교환합니다. 예를 들어, `token0`이 ETH이고 `token1`이 USDC인 경우, `zeroForOne`을 `true`로 설정하면 ETH로 USDC를 구매하는 것을 의미합니다. `amountSpecified`는 사용자가 판매하려는 토큰의 양입니다.

## 주문 이행 (Filling Orders)

Uniswap V3에서는 유동성이 여러 가격 범위에 저장되기 때문에, 풀 컨트랙트는 사용자로부터 "주문을 이행"하는 데 필요한 모든 유동성을 찾아야 합니다. 이는 사용자가 선택한 방향으로 초기화된 틱을 반복하여 수행됩니다.

계속하기 전에 두 개의 새로운 구조체를 정의해야 합니다.
```solidity
struct SwapState {
    uint256 amountSpecifiedRemaining;
    uint256 amountCalculated;
    uint160 sqrtPriceX96;
    int24 tick;
}

struct StepState {
    uint160 sqrtPriceStartX96;
    int24 nextTick;
    uint160 sqrtPriceNextX96;
    uint256 amountIn;
    uint256 amountOut;
}
```

`SwapState`는 현재 스왑의 상태를 유지합니다. `amountSpecifiedRemaining`은 풀이 구매해야 하는 토큰의 남은 양을 추적합니다. 이 값이 0이 되면 스왑이 완료된 것입니다. `amountCalculated`는 컨트랙트에서 계산한 출력량입니다. `sqrtPriceX96`과 `tick`은 스왑 완료 후의 새로운 현재 가격과 틱입니다.

`StepState`는 현재 스왑 단계의 상태를 유지합니다. 이 구조체는 "주문 이행"의 **한 번의 반복** 상태를 추적합니다. `sqrtPriceStartX96`은 반복이 시작되는 가격을 추적합니다. `nextTick`은 스왑에 유동성을 제공할 다음 초기화된 틱이고, `sqrtPriceNextX96`은 다음 틱의 가격입니다. `amountIn`과 `amountOut`은 현재 반복의 유동성으로 제공될 수 있는 양입니다.

> 틱 간 스왑 (즉, 여러 가격 범위를 넘나드는 스왑)을 구현하고 나면, 반복의 개념이 더 명확해질 것입니다.

```solidity
// src/UniswapV3Pool.sol

function swap(...) {
    Slot0 memory slot0_ = slot0;

    SwapState memory state = SwapState({
        amountSpecifiedRemaining: amountSpecified,
        amountCalculated: 0,
        sqrtPriceX96: slot0_.sqrtPriceX96,
        tick: slot0_.tick
    });
    ...
```

주문을 이행하기 전에 `SwapState` 인스턴스를 초기화합니다. `amountSpecifiedRemaining`이 0이 될 때까지 루프를 돌 것입니다. 이는 풀이 사용자로부터 `amountSpecified` 토큰을 구매하기에 충분한 유동성을 가지고 있다는 것을 의미합니다.

```solidity
...
while (state.amountSpecifiedRemaining > 0) {
    StepState memory step;

    step.sqrtPriceStartX96 = state.sqrtPriceX96;

    (step.nextTick, ) = tickBitmap.nextInitializedTickWithinOneWord(
        state.tick,
        1,
        zeroForOne
    );

    step.sqrtPriceNextX96 = TickMath.getSqrtRatioAtTick(step.nextTick);
```

루프에서 스왑에 유동성을 제공해야 하는 가격 범위를 설정합니다. 범위는 `state.sqrtPriceX96`에서 `step.sqrtPriceNextX96`까지이며, 후자는 다음 초기화된 틱의 가격입니다 (`nextInitializedTickWithinOneWord`에서 반환됨 - 이전 챕터에서 이 함수를 알고 있습니다).

```solidity
(state.sqrtPriceX96, step.amountIn, step.amountOut) = SwapMath
    .computeSwapStep(
        state.sqrtPriceX96,
        step.sqrtPriceNextX96,
        liquidity,
        state.amountSpecifiedRemaining
    );
```

다음으로 현재 가격 범위에서 제공할 수 있는 양과 스왑으로 인해 발생하는 새로운 현재 가격을 계산합니다.

```solidity
    state.amountSpecifiedRemaining -= step.amountIn;
    state.amountCalculated += step.amountOut;
    state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
}
```

루프의 마지막 단계는 `SwapState`를 업데이트하는 것입니다. `step.amountIn`은 가격 범위가 사용자로부터 구매할 수 있는 토큰의 수이고, `step.amountOut`은 풀이 사용자에게 판매할 수 있는 다른 토큰의 관련 수입니다. `state.sqrtPriceX96`은 스왑 후에 설정될 현재 가격입니다 (거래는 현재 가격을 변경한다는 것을 상기하십시오).

## SwapMath 컨트랙트

`SwapMath.computeSwapStep`을 더 자세히 살펴보겠습니다.

```solidity
// src/lib/SwapMath.sol
function computeSwapStep(
    uint160 sqrtPriceCurrentX96,
    uint160 sqrtPriceTargetX96,
    uint128 liquidity,
    uint256 amountRemaining
)
    internal
    pure
    returns (
        uint160 sqrtPriceNextX96,
        uint256 amountIn,
        uint256 amountOut
    )
{
    ...
```

이것이 스왑의 핵심 로직입니다. 이 함수는 하나의 가격 범위 내에서 그리고 사용 가능한 유동성을 고려하여 스왑 양을 계산합니다. 새로운 현재 가격과 입력 및 출력 토큰 양을 반환합니다. 입력 양은 사용자가 제공하지만, `computeSwapStep` 호출 한 번으로 사용자 지정 입력 양 중 얼마나 처리되었는지 알기 위해 여전히 계산합니다.

```solidity
bool zeroForOne = sqrtPriceCurrentX96 >= sqrtPriceTargetX96;

sqrtPriceNextX96 = Math.getNextSqrtPriceFromInput(
    sqrtPriceCurrentX96,
    liquidity,
    amountRemaining,
    zeroForOne
);
```

가격을 확인하여 스왑 방향을 결정할 수 있습니다. 방향을 알면 `amountRemaining` 토큰을 스왑한 후의 가격을 계산할 수 있습니다. 이 함수에 대해서는 아래에서 다시 설명하겠습니다.

새로운 가격을 찾은 후에는 이미 가지고 있는 함수 ( `mint` 함수에서 유동성으로부터 토큰 양을 계산하는 데 사용했던 것과 동일한 함수) 를 사용하여 스왑의 입력 및 출력 양을 계산할 수 있습니다.
```solidity
amountIn = Math.calcAmount0Delta(
    sqrtPriceCurrentX96,
    sqrtPriceNextX96,
    liquidity
);
amountOut = Math.calcAmount1Delta(
    sqrtPriceCurrentX96,
    sqrtPriceNextX96,
    liquidity
);
```

그리고 방향이 반대인 경우 양을 스왑합니다.
```solidity
if (!zeroForOne) {
    (amountIn, amountOut) = (amountOut, amountIn);
}
```

`computeSwapStep`에 대한 설명은 여기까지입니다!

## 스왑 양으로 가격 찾기

이제 `Math.getNextSqrtPriceFromInput`을 살펴보겠습니다. 이 함수는 다른 $\sqrt{P}$, 유동성 및 입력 양이 주어졌을 때 $\sqrt{P}$를 계산합니다. 현재 가격과 유동성이 주어졌을 때 지정된 입력 토큰 양을 스왑한 후의 가격이 어떻게 될지 알려줍니다.

좋은 소식은 이미 공식을 알고 있다는 것입니다. Python에서 `price_next`를 계산했던 방법을 상기해 보세요.
```python
# When amount_in is token0
price_next = int((liq * q96 * sqrtp_cur) // (liq * q96 + amount_in * sqrtp_cur))
# When amount_in is token1
price_next = sqrtp_cur + (amount_in * q96) // liq
```

이것을 Solidity로 구현할 것입니다.
```solidity
// src/lib/Math.sol
function getNextSqrtPriceFromInput(
    uint160 sqrtPriceX96,
    uint128 liquidity,
    uint256 amountIn,
    bool zeroForOne
) internal pure returns (uint160 sqrtPriceNextX96) {
    sqrtPriceNextX96 = zeroForOne
        ? getNextSqrtPriceFromAmount0RoundingUp(
            sqrtPriceX96,
            liquidity,
            amountIn
        )
        : getNextSqrtPriceFromAmount1RoundingDown(
            sqrtPriceX96,
            liquidity,
            amountIn
        );
}
```

이 함수는 양방향 스왑을 처리합니다. 계산이 다르기 때문에 별도의 함수로 구현할 것입니다.

```solidity
function getNextSqrtPriceFromAmount0RoundingUp(
    uint160 sqrtPriceX96,
    uint128 liquidity,
    uint256 amountIn
) internal pure returns (uint160) {
    uint256 numerator = uint256(liquidity) << FixedPoint96.RESOLUTION;
    uint256 product = amountIn * sqrtPriceX96;

    if (product / amountIn == sqrtPriceX96) {
        uint256 denominator = numerator + product;
        if (denominator >= numerator) {
            return
                uint160(
                    mulDivRoundingUp(numerator, sqrtPriceX96, denominator)
                );
        }
    }

    return
        uint160(
            divRoundingUp(numerator, (numerator / sqrtPriceX96) + amountIn)
        );
}
```
이 함수에서는 두 가지 공식을 구현합니다. 첫 번째 `return`에서 Python에서 구현한 것과 동일한 공식을 구현합니다. 이것이 가장 정확한 공식이지만, `amountIn`과 `sqrtPriceX96`을 곱할 때 오버플로가 발생할 수 있습니다. 공식은 ( "출력량 계산"에서 논의했습니다):
$$\sqrt{P_{target}} = \frac{\sqrt{P}L}{\Delta x \sqrt{P} + L}$$

오버플로가 발생하면 덜 정확한 대안 공식을 사용합니다.
$$\sqrt{P_{target}} = \frac{L}{\Delta x + \frac{L}{\sqrt{P}}}$$

이는 단순히 분자와 분모를 $\sqrt{P}$로 나누어 분자의 곱셈을 제거한 이전 공식입니다.

다른 함수는 더 간단한 수학을 사용합니다.
```solidity
function getNextSqrtPriceFromAmount1RoundingDown(
    uint160 sqrtPriceX96,
    uint128 liquidity,
    uint256 amountIn
) internal pure returns (uint160) {
    return
        sqrtPriceX96 +
        uint160((amountIn << FixedPoint96.RESOLUTION) / liquidity);
}
```

## 스왑 완료

이제 `swap` 함수로 돌아가서 완료해 보겠습니다.

이 시점에서 다음 초기화된 틱을 반복하고, 사용자가 지정한 `amountSpecified`를 이행하고, 입력 및 출력 양을 계산하고, 새로운 가격과 틱을 찾았습니다. 이 마일스톤에서는 하나의 가격 범위 내에서만 스왑을 구현하므로 이것으로 충분합니다. 이제 컨트랙트의 상태를 업데이트하고, 사용자에게 토큰을 보내고, 그 대가로 토큰을 받아야 합니다.

```solidity
if (state.tick != slot0_.tick) {
    (slot0.sqrtPriceX96, slot0.tick) = (state.sqrtPriceX96, state.tick);
}
```

먼저 새로운 가격과 틱을 설정합니다. 이 작업은 컨트랙트 저장소에 쓰기 때문에 가스 소비를 최적화하기 위해 새로운 틱이 다른 경우에만 수행하려고 합니다.

```solidity
(amount0, amount1) = zeroForOne
    ? (
        int256(amountSpecified - state.amountSpecifiedRemaining),
        -int256(state.amountCalculated)
    )
    : (
        -int256(state.amountCalculated),
        int256(amountSpecified - state.amountSpecifiedRemaining)
    );
```

다음으로 스왑 방향과 스왑 루프 중에 계산된 양을 기반으로 스왑 양을 계산합니다.

```solidity
if (zeroForOne) {
    IERC20(token1).transfer(recipient, uint256(-amount1));

    uint256 balance0Before = balance0();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
        amount0,
        amount1,
        data
    );
    if (balance0Before + uint256(amount0) > balance0())
        revert InsufficientInputAmount();
} else {
    IERC20(token0).transfer(recipient, uint256(-amount0));

    uint256 balance1Before = balance1();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
        amount0,
        amount1,
        data
    );
    if (balance1Before + uint256(amount1) > balance1())
        revert InsufficientInputAmount();
}
```

다음으로 스왑 방향에 따라 사용자와 토큰을 교환합니다. 이 부분은 마일스톤 2에 있던 것과 동일하며, 다른 스왑 방향에 대한 처리가 추가되었습니다.

이것으로 끝입니다! 스왑이 완료되었습니다!

## 테스트

테스트는 크게 변경되지 않습니다. `amountSpecified`와 `zeroForOne`을 `swap` 함수에 전달하기만 하면 됩니다. 출력량은 이제 Solidity에서 계산되기 때문에 약간 변경될 것입니다.

이제 반대 방향으로 스왑하는 것을 테스트할 수 있습니다! 숙제로 남겨두겠습니다 (전체 스왑이 단일 가격 범위에서 처리될 수 있도록 작은 입력 양을 선택해야 합니다). 어렵게 느껴진다면 [제 테스트](https://github.com/Jeiwan/uniswapv3-code/blob/milestone_2/test/UniswapV3Pool.t.sol)를 살펴보는 것을 주저하지 마십시오!