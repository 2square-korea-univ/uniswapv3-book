# 일반화된 민팅

이제 `mint` 함수를 업데이트하여 더 이상 값을 하드 코딩할 필요 없이 대신 계산할 수 있습니다.

## 초기화된 틱 인덱싱

`mint` 함수에서 틱에서 사용 가능한 유동성 정보를 저장하기 위해 `TickInfo` 매핑을 업데이트하는 것을 상기해 보세요. 이제 비트맵 인덱스에서 새로 초기화된 틱을 인덱싱해야 합니다. 이 인덱스는 나중에 스왑 중에 다음 초기화된 틱을 찾는 데 사용됩니다.

먼저 `Tick.update` 함수를 업데이트해야 합니다.
```solidity
// src/lib/Tick.sol
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint128 liquidityDelta
) internal returns (bool flipped) {
    ...
    flipped = (liquidityAfter == 0) != (liquidityBefore == 0);
    ...
}
```

이제 `flipped` 플래그를 반환합니다. 이 플래그는 유동성이 빈 틱에 추가되거나 틱에서 전체 유동성이 제거될 때 true로 설정됩니다.

그런 다음 `mint` 함수에서 비트맵 인덱스를 업데이트합니다.
```solidity
// src/UniswapV3Pool.sol
...
bool flippedLower = ticks.update(lowerTick, amount);
bool flippedUpper = ticks.update(upperTick, amount);

if (flippedLower) {
    tickBitmap.flipTick(lowerTick, 1);
}

if (flippedUpper) {
    tickBitmap.flipTick(upperTick, 1);
}
...
```

> 다시 말하지만, 마일스톤 4에서 다른 값을 도입할 때까지 틱 간격을 1로 설정합니다.

## 토큰 수량 계산

`mint` 함수에서 가장 큰 변화는 토큰 수량 계산으로 전환하는 것입니다. 마일스톤 1에서는 다음 값을 하드 코딩했습니다.
```solidity
    amount0 = 0.998976618347425280 ether;
    amount1 = 5000 ether;
```

이제 마일스톤 1의 공식을 사용하여 Solidity에서 계산할 것입니다. 해당 공식을 다시 떠올려 보겠습니다.

$$\Delta x = \frac{L(\sqrt{p(i_u)} - \sqrt{p(i_c)})}{\sqrt{p(i_u)}\sqrt{p(i_c)}}$$
$$\Delta y = L(\sqrt{p(i_c)} - \sqrt{p(i_l)})$$

$\Delta x$는 `token0` 또는 토큰 $x$의 수량입니다. Solidity에서 구현해 보겠습니다.
```solidity
// src/lib/Math.sol
function calcAmount0Delta(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint128 liquidity
) internal pure returns (uint256 amount0) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    require(sqrtPriceAX96 > 0);

    amount0 = divRoundingUp(
        mulDivRoundingUp(
            (uint256(liquidity) << FixedPoint96.RESOLUTION),
            (sqrtPriceBX96 - sqrtPriceAX96),
            sqrtPriceBX96
        ),
        sqrtPriceAX96
    );
}
```

> 이 함수는 Python 스크립트의 `calc_amount0`과 동일합니다.

첫 번째 단계는 빼기 시 언더플로우가 발생하지 않도록 가격을 정렬하는 것입니다. 다음으로 `liquidity`를 2**96을 곱하여 Q96.64 숫자로 변환합니다. 다음으로 공식에 따라 가격 차이를 곱하고 더 큰 가격으로 나눕니다. 그런 다음 더 작은 가격으로 나눕니다. 나눗셈 순서는 중요하지 않지만 가격 곱셈이 오버플로우될 수 있으므로 두 번의 나눗셈을 수행하려고 합니다.

`mulDivRoundingUp`을 사용하여 곱셈과 나눗셈을 한 번에 수행합니다. 이 함수는 `PRBMath`의 `mulDiv`를 기반으로 합니다.
```solidity
function mulDivRoundingUp(
    uint256 a,
    uint256 b,
    uint256 denominator
) internal pure returns (uint256 result) {
    result = PRBMath.mulDiv(a, b, denominator);
    if (mulmod(a, b, denominator) > 0) {
        require(result < type(uint256).max);
        result++;
    }
}
```

`mulmod`는 두 숫자(`a`와 `b`)를 곱하고 결과를 `denominator`로 나누고 나머지를 반환하는 Solidity 함수입니다. 나머지가 양수이면 결과를 올림합니다.

다음은 $\Delta y$입니다.
```solidity
function calcAmount1Delta(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint128 liquidity
) internal pure returns (uint256 amount1) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    amount1 = mulDivRoundingUp(
        liquidity,
        (sqrtPriceBX96 - sqrtPriceAX96),
        FixedPoint96.Q96
    );
}
```

> 이 함수는 Python 스크립트의 `calc_amount1`과 동일합니다.

다시 말하지만, 곱셈 중 오버플로우를 방지하기 위해 `mulDivRoundingUp`을 사용합니다.

이제 끝났습니다! 이제 함수를 사용하여 토큰 수량을 계산할 수 있습니다.
```solidity
// src/UniswapV3Pool.sol
function mint(...) {
    ...
    Slot0 memory slot0_ = slot0;

    amount0 = Math.calcAmount0Delta(
        slot0_.sqrtPriceX96,
        TickMath.getSqrtRatioAtTick(upperTick),
        amount
    );

    amount1 = Math.calcAmount1Delta(
        slot0_.sqrtPriceX96,
        TickMath.getSqrtRatioAtTick(lowerTick),
        amount
    );
    ...
}
```

다른 모든 것은 동일하게 유지됩니다. 풀 테스트에서 수량을 업데이트해야 합니다. 반올림 때문에 약간 다를 것입니다.