## 크로스-틱 스왑

크로스-틱 스왑은 유니스왑 V3에서 아마도 가장 고도화된 기능 중 하나일 것입니다. 다행히도 크로스-틱 스왑을 구현하는 데 필요한 대부분의 기능은 이미 구현되어 있습니다. 구현에 앞서, 크로스-틱 스왑이 어떻게 작동하는지 먼저 살펴보도록 하겠습니다.

## 크로스-틱 스왑 작동 방식

일반적인 유니스왑 V3 풀은 다수의 중첩된 (그리고 미결제의) 가격 범위를 포함하는 풀입니다. 각 풀은 현재 $\sqrt{P}$ 와 틱을 추적합니다. 사용자가 토큰을 스왑하면 스왑 방향에 따라 현재 가격과 틱이 왼쪽 또는 오른쪽으로 이동합니다. 이러한 움직임은 스왑 과정에서 풀에 토큰이 추가 및 제거됨으로써 발생합니다.

풀은 또한 $L$ (`liquidity`, 우리 코드의 `유동성` 변수)를 추적하는데, 이는 **현재 가격을 포함하는 모든 가격 범위에서 제공하는 총 유동성**입니다. 큰 가격 변동 중에는 현재 가격이 가격 범위를 벗어날 것으로 예상됩니다. 이러한 현상이 발생하면 해당 가격 범위는 비활성화되고 해당 유동성은 $L$ 에서 차감됩니다. 반대로, 현재 가격이 가격 범위 안으로 진입하면 $L$ 이 증가하고 해당 가격 범위가 활성화됩니다.

다음 그림을 분석해 보겠습니다:



![가격 범위의 역학](images/price_range_dynamics.png)

이 그림에는 세 개의 가격 범위가 제시되어 있습니다. 맨 위에 위치한 범위는 현재 가격을 포함하며 현재 활성화된 범위입니다. 이 가격 범위의 유동성은 풀 컨트랙트의 `liquidity` 상태 변수에 설정됩니다.

최상위 가격 범위에서 모든 ETH를 매수하면 가격이 상승하여 오른쪽 가격 범위로 이동합니다. 이 범위는 현재 USDC가 아닌 ETH만을 포함합니다. 수요를 충족할 만큼 충분한 유동성이 존재한다면 해당 가격 범위에서 멈출 수 있습니다. 이 경우, `liquidity` 변수는 해당 가격 범위에서 제공하는 유동성만을 포함하게 됩니다. ETH를 계속 매수하여 오른쪽 가격 범위를 소진하면 해당 가격 범위의 오른쪽에 위치한 다른 가격 범위가 필요합니다. 더 이상 가격 범위가 존재하지 않으면 스왑은 중단되어 부분적으로만 완료될 수 있습니다.

최상위 가격 범위에서 모든 USDC를 매수하면 (ETH를 매도) 가격이 하락하여 왼쪽 가격 범위로 이동합니다. 이 범위는 현재 USDC만을 포함합니다. 이를 소진하면 왼쪽에 위치한 다른 가격 범위가 필요합니다.

현재 가격은 스왑 과정 중에 이동합니다. 한 가격 범위에서 다른 가격 범위로 이동하지만, 항상 가격 범위 내에 머물러야 합니다. 그렇지 않으면 거래가 불가능합니다.

물론, 가격 범위는 겹칠 수 있으므로 실제로는 가격 범위 간 전환이 원활하게 이루어집니다. 또한 간격을 뛰어넘는 것은 불가능합니다. 스왑은 부분적으로 완료될 수 있습니다. 가격 범위가 중첩되는 영역에서는 가격 변동이 완만해진다는 점도 주목할 가치가 있습니다. 이는 이러한 영역에서 공급이 더욱 풍부하고 수요의 영향력이 낮기 때문입니다 (서론에서 높은 수요와 낮은 공급은 가격 상승을 야기한다고 상기하십시오).

현재 구현은 이러한 유동성을 지원하지 않습니다. 단일 활성 가격 범위 내에서만 스왑을 허용합니다. 이제 이를 개선할 것입니다.

## `computeSwapStep` 함수 업데이트

`swap` 함수에서 사용자가 요청한 양을 채우기 위해 초기화된 틱 (즉, 유동성이 존재하는 틱)을 반복적으로 처리합니다. 각 반복 단계에서 다음을 수행합니다:

1. `tickBitmap.nextInitializedTickWithinOneWord` 를 사용하여 다음 초기화된 틱을 찾습니다.
2. 현재 가격과 다음 초기화된 틱 사이의 범위에서 스왑을 수행합니다 (`SwapMath.computeSwapStep` 사용).
3. 항상 현재 유동성이 스왑을 충족하기에 충분하다고 가정합니다 (즉, 스왑 후 가격은 현재 가격과 다음 초기화된 틱 사이에 위치합니다).

그러나 세 번째 단계가 참이 아니라면 어떻게 될까요? 테스트에서 이러한 시나리오를 다루었습니다:
```solidity
// test/UniswapV3Pool.t.sol
function testSwapBuyEthNotEnoughLiquidity() public {
    ...

    uint256 swapAmount = 5300 ether;

    ...

    vm.expectRevert(stdError.arithmeticError);
    pool.swap(address(this), false, swapAmount, extra);
}
```

"Arithmetic over/underflow" 오류는 풀이 보유한 것보다 더 많은 이더를 전송하려고 시도할 때 발생합니다. 이 오류는 현재 구현에서 항상 모든 스왑을 충족할 만큼 충분한 유동성이 있다고 가정하기 때문에 발생합니다:

```solidity
// src/lib/SwapMath.sol
function computeSwapStep(...) {
    ...

    sqrtPriceNextX96 = Math.getNextSqrtPriceFromInput(
        sqrtPriceCurrentX96,
        liquidity,
        amountRemaining,
        zeroForOne
    );

    amountIn = ...
    amountOut = ...
}
```

이를 개선하기 위해 몇 가지 상황을 고려해야 합니다:
1. 현재 틱과 다음 틱 사이의 범위에 `amountRemaining` 을 충족할 만큼 충분한 유동성이 있는 경우;
2. 해당 범위가 전체 `amountRemaining` 을 충족하지 못하는 경우.

첫 번째 경우에는 스왑이 해당 범위 내에서 완전히 완료됩니다. 이는 현재 구현된 시나리오입니다. 두 번째 상황에서는 해당 범위에서 제공하는 전체 유동성을 소진하고 **다음 범위로 이동**합니다 (존재하는 경우). 이러한 점들을 염두에 두고 `computeSwapStep` 함수를 재작업해 보겠습니다:
```solidity
// src/lib/SwapMath.sol
function computeSwapStep(...) {
    ...
    amountIn = zeroForOne
        ? Math.calcAmount0Delta(
            sqrtPriceCurrentX96,
            sqrtPriceTargetX96,
            liquidity
        )
        : Math.calcAmount1Delta(
            sqrtPriceCurrentX96,
            sqrtPriceTargetX96,
            liquidity
        );

    if (amountRemaining >= amountIn) sqrtPriceNextX96 = sqrtPriceTargetX96;
    else
        sqrtPriceNextX96 = Math.getNextSqrtPriceFromInput(
            sqrtPriceCurrentX96,
            liquidity,
            amountRemaining,
            zeroForOne
        );

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
}
```
먼저, 현재 범위가 충족할 수 있는 입력량인 `amountIn` 을 계산합니다. 만약 이것이 `amountRemaining` 보다 작다면, 현재 가격 범위가 전체 스왑을 충족할 수 없다고 판단하여 다음 $\sqrt{P}$ 는 가격 범위의 상한/하한 $\sqrt{P}$ 가 됩니다 (즉, 가격 범위의 전체 유동성을 사용합니다). `amountIn` 이 `amountRemaining` 보다 크다면 `sqrtPriceNextX96` 를 계산합니다. 이것은 현재 가격 범위 내의 가격이 됩니다.

마지막으로, 다음 가격을 파악한 후 이 짧은 가격 범위 내에서 `amountIn` 을 다시 계산하고 `amountOut` 을 계산합니다 (전체 유동성을 소비하지는 않습니다).

이해가 되셨기를 바랍니다!

## `swap` 함수 업데이트

이제 `swap` 함수에서 이전 파트에서 소개한 경우를 처리해야 합니다. 스왑 가격이 가격 범위의 경계에 도달했을 때입니다. 이러한 상황이 발생하면, 떠나는 가격 범위를 비활성화하고 다음 가격 범위를 활성화하려고 시도합니다. 또한 루프의 다음 반복을 시작하고 유동성이 존재하는 다른 틱을 찾으려고 합니다.

루프를 업데이트하기 전에 `tickBitmap.nextInitializedTickWithinOneWord()` 호출에서 반환된 두 번째 값을 `step.initialized` 에 저장해 보겠습니다:
```solidity
(step.nextTick, step.initialized) = tickBitmap.nextInitializedTickWithinOneWord(
    state.tick,
    1,
    zeroForOne
);
```

(이전 마일스톤에서는 `step.nextTick` 만 저장했습니다.)

다음 틱이 초기화되었는지 여부를 확인함으로써 틱 비트맵의 현재 워드에 초기화된 틱이 없는 상황에서 가스 비용을 절약하는 데 도움이 됩니다.

이제 루프의 끝에 추가해야 할 사항은 다음과 같습니다:
```solidity
if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
    if (step.initialized) {
        int128 liquidityDelta = ticks.cross(step.nextTick);

        if (zeroForOne) liquidityDelta = -liquidityDelta;

        state.liquidity = LiquidityMath.addLiquidity(
            state.liquidity,
            liquidityDelta
        );

        if (state.liquidity == 0) revert NotEnoughLiquidity();
    }

    state.tick = zeroForOne ? step.nextTick - 1 : step.nextTick;
} else {
    state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
}
```

두 번째 분기는 이전과 동일합니다. 현재 가격이 범위 내에 유지되는 경우를 처리합니다. 따라서 첫 번째 분기에 집중해 보겠습니다.

여기서는 현재 유동성을 업데이트하지만, 다음 틱이 초기화된 경우에만 업데이트합니다 (그렇지 않으면 가스 비용을 절약하기 위해 유동성에 0을 추가하는 것을 건너뜁니다).

`state.sqrtPriceX96` 는 새로운 현재 가격, 즉 현재 스왑 이후에 설정될 가격입니다. `step.sqrtPriceNextX96` 는 다음 초기화된 틱의 가격입니다. 이 두 값이 동일하다면 가격 범위 경계에 도달한 것입니다. 위에서 설명한 것처럼, 이러한 상황이 발생하면 $L$ 을 업데이트하고 (유동성 추가 또는 제거) 경계 틱을 현재 틱으로 사용하여 스왑을 계속하려고 시도합니다.

관례상 틱을 교차하는 것은 왼쪽에서 오른쪽으로 교차하는 것을 의미합니다. 따라서 낮은 틱을 교차하면 항상 유동성이 추가되고, 높은 틱을 교차하면 항상 유동성이 제거됩니다. 그러나 `zeroForOne` 가 참이면 부호를 반전시킵니다. 가격이 하락하면 (토큰 $x$ 가 판매됨) 높은 틱은 유동성을 추가하고 낮은 틱은 유동성을 제거합니다.

`state.tick` 을 업데이트할 때 가격이 하락하면 (`zeroForOne` 가 참) 가격 범위에서 벗어나기 위해 1을 빼야 합니다. 가격이 상승하면 (`zeroForOne` 가 거짓) 현재 틱은 항상 `TickBitmap.nextInitializedTickWithinOneWord` 에서 제외됩니다.

또 다른 작지만 매우 중요한 변경 사항은 틱을 교차할 때 $L$ 을 업데이트해야 한다는 것입니다. 루프 이후에 다음을 수행합니다:
```solidity
if (liquidity_ != state.liquidity) liquidity = state.liquidity;
```

루프 내에서 가격 범위에 진입/이탈할 때 `state.liquidity` 를 여러 번 업데이트합니다. 스왑 이후에는 새로운 현재 가격에서 사용 가능한 유동성을 반영하도록 전역 $L$ 을 업데이트해야 합니다. 또한 스왑을 완료할 때만 전역 변수를 업데이트하는 이유는 컨트랙트의 스토리지에 쓰는 것이 비용이 많이 드는 작업이기 때문에 가스 소비 최적화 때문입니다.

## 유동성 추적 및 틱 교차

이제 업데이트된 `Tick` 라이브러리를 살펴보겠습니다.

첫 번째 변경 사항은 `Tick.Info` 구조체에 있습니다. 이제 틱 유동성을 추적하기 위해 두 개의 변수가 존재합니다:
```solidity
struct Info {
    bool initialized;
    // 틱의 총 유동성
    uint128 liquidityGross;
    // 틱이 교차될 때 추가 또는 차감되는 유동성 양
    int128 liquidityNet;
}
```

`liquidityGross` 는 틱의 절대 유동성 양을 추적합니다. 틱이 반전되었는지 여부를 확인하는 데 필요합니다. 반면에 `liquidityNet` 은 부호 있는 정수입니다. 틱이 교차될 때 추가 (낮은 틱의 경우) 또는 제거 (높은 틱의 경우)되는 유동성 양을 추적합니다.

`liquidityNet` 은 `update` 함수에서 설정됩니다:
```solidity
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    int128 liquidityDelta,
    bool upper
) internal returns (bool flipped) {
    ...

    tickInfo.liquidityNet = upper
        ? int128(int256(tickInfo.liquidityNet) - liquidityDelta)
        : int128(int256(tickInfo.liquidityNet) + liquidityDelta);
}
```

위에서 살펴본 `cross` 함수는 단순히 `liquidityNet` 을 반환합니다 (향후 마일스톤에서 새로운 기능을 도입한 이후에는 더욱 복잡해질 것입니다):
```solidity
function cross(mapping(int24 => Tick.Info) storage self, int24 tick)
    internal
    view
    returns (int128 liquidityDelta)
{
    Tick.Info storage info = self[tick];
    liquidityDelta = info.liquidityNet;
}
```

## 테스팅

이제 다양한 유동성 설정을 검토하고 풀 구현이 이를 올바르게 처리할 수 있는지 확인하기 위해 테스트를 진행해 보겠습니다.

### 단일 가격 범위



![가격 범위 내 스왑](images/swap_within_price_range.png)

이것은 이전의 시나리오입니다. 코드를 업데이트한 후에도 이전 기능이 여전히 올바르게 작동하는지 확인해야 합니다.

> 간결성을 위해 테스트의 가장 중요한 부분만 제시하겠습니다. 전체 테스트는 [코드 저장소](https://github.com/Jeiwan/uniswapv3-code/blob/milestone_3/test/UniswapV3Pool.Swaps.t.sol)에서 확인하실 수 있습니다.

- ETH 매수 시:
    ```solidity
    function testBuyETHOnePriceRange() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](1);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            -0.008396874645169943 ether,
            42 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5604415652688968742392013927525, // 5003.8180249710795
                tick: 85183,
                currentLiquidity: liquidity[0].amount
            })
        );
    }
    ```
- USDC 매수 시:
    ```solidity
    function testBuyUSDCOnePriceRange() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](1);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            0.01337 ether,
            -66.807123823853842027 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5598737223630966236662554421688, // 4993.683362269102
                tick: 85163,
                currentLiquidity: liquidity[0].amount
            })
        );
    }
    ```

이 두 시나리오 모두에서 소량의 ETH 또는 USDC를 매수합니다. 가격이 우리가 생성한 유일한 가격 범위를 벗어나지 않을 만큼 충분히 작은 양입니다. 스왑 완료 후의 주요 값은 다음과 같습니다:
1. `sqrtPriceX96` 는 초기 가격보다 약간 높거나 낮으며 가격 범위 내에 유지됩니다.
2. `currentLiquidity` 는 변경되지 않은 상태로 유지됩니다.

### 여러 개의 동일하고 중첩된 가격 범위



![중첩된 범위 내 스왑](images/swap_within_overlapping_price_ranges.png)

- ETH 매수 시:
    ```solidity
    function testBuyETHTwoEqualPriceRanges() public {
        LiquidityRange memory range = liquidityRange(
            4545,
            5500,
            1 ether,
            5000 ether,
            5000
        );
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = range;
        liquidity[1] = range;

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            -0.008398516982770993 ether,
            42 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5603319704133145322707074461607, // 5001.861214026131
                tick: 85179,
                currentLiquidity: liquidity[0].amount + liquidity[1].amount
            })
        );
    }
    ```

- USDC 매수 시:
    ```solidity
    function testBuyUSDCTwoEqualPriceRanges() public {
        LiquidityRange memory range = liquidityRange(
            4545,
            5500,
            1 ether,
            5000 ether,
            5000
        );
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = range;
        liquidity[1] = range;

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            0.01337 ether,
            -66.827918929906650442 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5600479946976371527693873969480, // 4996.792621611429
                tick: 85169,
                currentLiquidity: liquidity[0].amount + liquidity[1].amount
            })
        );
    }
    ```

이 시나리오는 이전 시나리오와 유사하지만, 이번에는 두 개의 동일한 가격 범위를 생성합니다. 이들은 완전히 중첩된 가격 범위이므로, 실제로 더 많은 유동성을 가진 하나의 가격 범위처럼 작동합니다. 따라서 가격 변화가 이전 시나리오보다 완만합니다. 또한 더 깊은 유동성 덕분에 토큰을 약간 더 많이 얻을 수 있습니다.

### 연속적인 가격 범위



![연속적인 가격 범위에 걸친 스왑](images/swap_consecutive_price_ranges.png)

- ETH 매수 시:
    ```solidity
    function testBuyETHConsecutivePriceRanges() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);
        liquidity[1] = liquidityRange(5500, 6250, 1 ether, 5000 ether, 5000);

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            -1.820694594787485635 ether,
            10000 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 6190476002219365604851182401841, // 6105.045728033458
                tick: 87173,
                currentLiquidity: liquidity[1].amount
            })
        );
    }
    ```
- USDC 매수 시:
    ```solidity
    function testBuyUSDCConsecutivePriceRanges() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);
        liquidity[1] = liquidityRange(4000, 4545, 1 ether, 5000 ether, 5000);

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            2 ether,
            -9103.264925902176327184 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5069962753257045266417033265661, // 4094.9666586581643
                tick: 83179,
                currentLiquidity: liquidity[1].amount
            })
        );
    }
    ```

이러한 시나리오에서는 가격이 가격 범위를 벗어나는 큰 스왑을 수행합니다. 결과적으로 두 번째 가격 범위가 활성화되고 스왑을 충족할 만큼 충분한 유동성을 제공합니다. 두 시나리오 모두에서 가격이 현재 가격 범위 밖에 있고 가격 범위가 비활성화되었음을 확인할 수 있습니다 (현재 유동성은 두 번째 가격 범위의 유동성과 동일합니다).

### 부분적으로 중첩된 가격 범위



![부분적으로 중첩된 가격 범위에 걸친 스왑](images/swap_partially_overlapping_price_ranges.png)

- ETH 매수 시:
    ```solidity
    function testBuyETHPartiallyOverlappingPriceRanges() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);
        liquidity[1] = liquidityRange(5001, 6250, 1 ether, 5000 ether, 5000);

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            -1.864220641170389178 ether,
            10000 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 6165345094827913637987008642386, // 6055.578153852725
                tick: 87091,
                currentLiquidity: liquidity[1].amount
            })
        );
    }
    ```

- USDC 매수 시:
    ```solidity
    function testBuyUSDCPartiallyOverlappingPriceRanges() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);
        liquidity[1] = liquidityRange(4000, 4999, 1 ether, 5000 ether, 5000);

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            2 ether,
            -9321.077831210790476918 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5090915820491052794734777344590, // 4128.883835866256
                tick: 83261,
                currentLiquidity: liquidity[1].amount
            })
        );
    }
    ```

이것은 이전 시나리오의 변형이지만, 이번에는 가격 범위가 부분적으로 중첩됩니다. 가격 범위가 중첩되는 영역에는 더 깊은 유동성이 존재하므로 가격 변동이 완만해집니다. 이는 중첩 범위에 더 많은 유동성을 제공하는 것과 유사합니다.

또한 두 스왑 모두에서 "연속적인 가격 범위" 시나리오보다 더 많은 토큰을 얻었습니다. 이는 다시 중첩 범위의 더 깊은 유동성 때문입니다.