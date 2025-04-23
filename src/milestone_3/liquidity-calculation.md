# 유동성 계산

Uniswap V3 수학 전반에서 아직 Solidity로 구현되지 않은 부분은 바로 유동성 계산입니다. Python 스크립트에는 다음과 같은 함수들이 있습니다.

```python
def liquidity0(amount, pa, pb):
    if pa > pb:
        pa, pb = pb, pa
    return (amount * (pa * pb) / q96) / (pb - pa)


def liquidity1(amount, pa, pb):
    if pa > pb:
        pa, pb = pb, pa
    return amount * q96 / (pb - pa)
```

`Manager.mint()` 함수에서 유동성을 계산할 수 있도록 Solidity로 이 함수들을 구현해 보겠습니다.

## 토큰 X에 대한 유동성 계산 구현

구현할 함수들은 토큰 양과 가격 범위를 알 때 유동성($L = \sqrt{xy}$)을 계산할 수 있게 해줍니다. 다행히도, 이미 모든 공식을 알고 있습니다. 다음 공식을 다시 한번 살펴보겠습니다.

$$\Delta x = \Delta \frac{1}{\sqrt{P}}L$$

이전 장에서 이 공식을 사용하여 스왑 양(이 경우 $\Delta x$)을 계산했으며, 이제 $L$을 찾기 위해 사용할 것입니다.

$$L = \frac{\Delta x}{\Delta \frac{1}{\sqrt{P}}}$$

또는, 단순화하면 다음과 같습니다.
$$L = \frac{\Delta x \sqrt{P_u} \sqrt{P_l}}{\sqrt{P_u} - \sqrt{P_l}}$$

> 이 공식은 [유동성 양 계산](https://uniswapv3book.com/docs/milestone_1/calculating-liquidity/#liquidity-amount-calculation)에서 유도했습니다.

Solidity에서는 곱셈 및 나눗셈 시 오버플로우를 처리하기 위해 `PRBMath`를 다시 사용합니다.

```solidity
function getLiquidityForAmount0(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint256 amount0
) internal pure returns (uint128 liquidity) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    uint256 intermediate = PRBMath.mulDiv(
        sqrtPriceAX96,
        sqrtPriceBX96,
        FixedPoint96.Q96
    );
    liquidity = uint128(
        PRBMath.mulDiv(amount0, intermediate, sqrtPriceBX96 - sqrtPriceAX96)
    );
}
```

## 토큰 Y에 대한 유동성 계산 구현

마찬가지로, $y$의 양과 가격 범위를 알 때 $L$을 찾기 위해 [유동성 양 계산](https://uniswapv3book.com/docs/milestone_1/calculating-liquidity/#liquidity-amount-calculation)의 다른 공식을 사용합니다.

$$\Delta y = \Delta\sqrt{P} L$$
$$L = \frac{\Delta y}{\sqrt{P_u}-\sqrt{P_l}}$$

```solidity
function getLiquidityForAmount1(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    liquidity = uint128(
        PRBMath.mulDiv(
            amount1,
            FixedPoint96.Q96,
            sqrtPriceBX96 - sqrtPriceAX96
        )
    );
}
```

이해가 되셨기를 바랍니다!

## 적정 유동성 찾기

왜 $L$을 계산하는 두 가지 방법이 있는지, 그리고 항상 $L = \sqrt{xy}$로 계산되는 하나의 $L$만 있었는데, 이 방법들 중 어느 것이 정확한지 궁금할 수 있습니다. 답은 둘 다 옳다는 것입니다.

위의 공식에서 우리는 서로 다른 매개변수, 즉 가격 범위와 토큰 양을 기준으로 $L$을 계산합니다. 서로 다른 가격 범위와 서로 다른 토큰 양은 서로 다른 $L$ 값을 초래합니다. 그리고 두 $L$ 값을 모두 계산하고 그 중 하나를 선택해야 하는 시나리오가 있습니다. `mint` 함수의 이 부분을 다시 떠올려 보세요.

```solidity
if (slot0_.tick < lowerTick) {
    amount0 = Math.calcAmount0Delta(...);
} else if (slot0_.tick < upperTick) {
    amount0 = Math.calcAmount0Delta(...);

    amount1 = Math.calcAmount1Delta(...);

    liquidity = LiquidityMath.addLiquidity(liquidity, int128(amount));
} else {
    amount1 = Math.calcAmount1Delta(...);
}
```

유동성을 계산할 때도 이 로직을 따라야 합니다.
1. 현재 가격보다 높은 범위에 대한 유동성을 계산하는 경우, 공식의 $\Delta x$ 버전을 사용합니다.
2. 현재 가격보다 낮은 범위에 대한 유동성을 계산하는 경우, $\Delta y$ 버전을 사용합니다.
3. 가격 범위가 현재 가격을 포함하는 경우, **둘 다** 계산하고 더 작은 값을 선택합니다.

> 다시 한번, 이러한 아이디어는 [유동성 양 계산](https://uniswapv3book.com/docs/milestone_1/calculating-liquidity/#liquidity-amount-calculation)에서 논의했습니다.

이제 이 로직을 구현해 보겠습니다.

현재 가격이 가격 범위의 하한선보다 낮은 경우:
```solidity
function getLiquidityForAmounts(
    uint160 sqrtPriceX96,
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint256 amount0,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    if (sqrtPriceX96 <= sqrtPriceAX96) {
        liquidity = getLiquidityForAmount0(
            sqrtPriceAX96,
            sqrtPriceBX96,
            amount0
        );
```

현재 가격이 범위 내에 있는 경우, 더 작은 $L$을 선택합니다.
```solidity
} else if (sqrtPriceX96 <= sqrtPriceBX96) {
    uint128 liquidity0 = getLiquidityForAmount0(
        sqrtPriceX96,
        sqrtPriceBX96,
        amount0
    );
    uint128 liquidity1 = getLiquidityForAmount1(
        sqrtPriceAX96,
        sqrtPriceX96,
        amount1
    );

    liquidity = liquidity0 < liquidity1 ? liquidity0 : liquidity1;
```

마지막으로:
```solidity
} else {
    liquidity = getLiquidityForAmount1(
        sqrtPriceAX96,
        sqrtPriceBX96,
        amount1
    );
}
```

완료되었습니다.