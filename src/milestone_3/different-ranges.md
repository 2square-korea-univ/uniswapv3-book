## 다양한 가격 범위

우리가 구현한 방식에 따르면, Pool 컨트랙트는 현재 가격을 포함하는 가격 범위만 생성합니다:
```solidity
// src/UniswapV3Pool.sol
function mint() {
    ...
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

    liquidity += uint128(amount);
    ...
}
```

이 코드 조각에서 현재 사용 가능한 유동성, 즉 현재 가격에서 사용 가능한 유동성만을 추적하는 유동성 추적기를 항상 업데이트하는 것을 볼 수 있습니다.

하지만 실제로는 가격 범위가 현재 가격 **아래 또는 위**에 생성될 수도 있습니다. 다시 말해, Uniswap V3의 설계는 유동성 공급자가 즉시 사용되지 않는 유동성을 제공할 수 있도록 허용합니다. 이러한 유동성은 현재 가격이 이러한 "휴면" 가격 범위에 진입할 때 "주입"됩니다.

존재할 수 있는 가격 범위의 종류는 다음과 같습니다:
1. 활성 가격 범위: 현재 가격을 포함하는 범위.
2. 현재 가격 아래에 위치한 가격 범위: 이 범위의 상단 틱(upper tick)은 현재 틱보다 낮습니다.
3. 현재 가격 위에 위치한 가격 범위: 이 범위의 하단 틱(lower tick)은 현재 틱보다 높습니다.

## 지정가 주문 (Limit Orders)

비활성 유동성 (즉, 현재 가격에서 제공되지 않는 유동성)에 대한 흥미로운 사실은 이것이 *지정가 주문*처럼 작동한다는 점입니다.

거래에서 지정가 주문은 가격이 거래자가 선택한 수준을 넘을 때 실행되는 주문입니다. 예를 들어, 1 ETH 가격이 \$1000로 떨어지면 1 ETH를 구매하는 지정가 주문을 설정할 수 있습니다. 마찬가지로, 지정가 주문을 사용하여 자산을 판매할 수도 있습니다. Uniswap V3를 사용하면 현재 가격 아래 또는 위에 가격 범위를 설정하여 유사한 동작을 얻을 수 있습니다. 이것이 어떻게 작동하는지 살펴봅시다:



![현재 가격 외부의 유동성 범위](images/ranges_outside_current_price.png)

현재 가격보다 낮은 (즉, 선택한 가격 범위가 현재 가격보다 완전히 낮은) 또는 높은 유동성을 제공하면, 전체 유동성은 **단 하나의 자산**으로 구성됩니다. 그 자산은 두 토큰 중 더 저렴한 토큰이 될 것입니다. 우리의 예시에서, 우리는 ETH를 토큰 $x$, USDC를 토큰 $y$로 하는 풀을 구축하고 가격을 다음과 같이 정의합니다:

$$P = \frac{y}{x}$$

현재 가격보다 낮은 유동성을 넣으면, 유동성은 오로지 USDC로만 구성될 것입니다. 왜냐하면 우리가 유동성을 추가한 범위에서는 USDC의 가격이 현재 가격보다 낮기 때문입니다. 마찬가지로, 현재 가격보다 높은 유동성을 넣으면 유동성은 ETH로만 구성될 것입니다. 왜냐하면 해당 범위에서는 ETH가 더 저렴하기 때문입니다.

소개 부분에서 보았던 다음 그림을 다시 떠올려 봅시다:



![가격 범위 소진](../milestone_1/images/range_depleted.png)

만약 이 범위에서 사용 가능한 모든 ETH 양을 구매하면, 해당 범위는 다른 토큰인 USDC만 포함하게 되고 가격은 곡선의 오른쪽으로 이동할 것입니다. 우리가 정의한 가격 ($\frac{y}{x}$)은 **증가**할 것입니다. 이 범위 오른쪽에 가격 범위가 있다면, 다음 스왑을 위해 ETH 유동성을, 그리고 오직 ETH만 가지고 있어야 하며, USDC는 없어야 합니다. 즉, ETH를 제공해야 합니다. 만약 계속해서 구매하고 가격을 올리면, 다음 가격 범위 또한 "소진"시킬 수 있습니다. 이는 모든 ETH를 구매하고 USDC를 판매하는 것을 의미합니다. 다시 말하지만, 가격 범위는 결국 USDC만 가지게 되고 현재 가격은 범위를 벗어나게 됩니다.

마찬가지로, USDC 토큰을 구매하면 가격을 왼쪽으로 이동시키고 풀에서 USDC 토큰을 제거합니다. 다음 가격 범위는 우리의 수요를 충족시키기 위해 USDC 토큰만 포함할 것이며, 위의 시나리오와 유사하게, 모든 USDC를 구매하면 결국 ETH 토큰만 포함하게 될 것입니다.

흥미로운 사실을 주목하세요: 전체 가격 범위를 넘나들 때, 해당 범위의 유동성은 한 토큰에서 다른 토큰으로 스왑됩니다. 그리고 매우 좁은 가격 범위, 즉 가격 이동 중에 빠르게 넘어가는 범위를 설정하면 지정가 주문을 얻게 됩니다! 예를 들어, 더 낮은 가격에 ETH를 구매하고 싶다면, 더 낮은 가격에 USDC만 포함하는 가격 범위를 설정하고 현재 가격이 그 범위를 넘을 때까지 기다려야 합니다. 그 후, 유동성을 제거하고 ETH로 변환된 유동성을 얻어야 합니다!

이 예시가 혼란스럽게 만들지 않았기를 바랍니다! 저는 이것이 가격 범위의 역학을 설명하는 좋은 방법이라고 생각합니다.

## `mint` 함수 업데이트

모든 종류의 가격 범위를 지원하려면, 현재 가격이 사용자가 지정한 가격 범위 아래, 내부 또는 위에 있는지 여부를 알아야 하고 그에 따라 토큰 양을 계산해야 합니다. 가격 범위가 현재 가격 위에 있다면, 유동성이 토큰 $x$로 구성되기를 원합니다:

```solidity
// src/UniswapV3Pool.sol
function mint() {
    ...
    if (slot0_.tick < lowerTick) {
        amount0 = Math.calcAmount0Delta(
            TickMath.getSqrtRatioAtTick(lowerTick),
            TickMath.getSqrtRatioAtTick(upperTick),
            amount
        );
    ...
```

가격 범위가 현재 가격을 포함하는 경우, 가격에 비례하는 양으로 두 토큰 모두를 원합니다 (이것은 우리가 앞서 구현한 시나리오입니다):
```solidity
} else if (slot0_.tick < upperTick) {
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

    liquidity = LiquidityMath.addLiquidity(liquidity, int128(amount));
```

이것이 `liquidity` 변수를 업데이트하고 싶어하는 유일한 시나리오라는 점에 주목하세요. 왜냐하면 이 변수는 즉시 사용 가능한 유동성을 추적하기 때문입니다.

다른 모든 경우, 즉 가격 범위가 현재 가격보다 낮을 때, 해당 범위가 토큰 $y$만 포함하기를 원합니다:
```solidity
} else {
    amount1 = Math.calcAmount1Delta(
        TickMath.getSqrtRatioAtTick(lowerTick),
        TickMath.getSqrtRatioAtTick(upperTick),
        amount
    );
}
```

이것이 전부입니다!