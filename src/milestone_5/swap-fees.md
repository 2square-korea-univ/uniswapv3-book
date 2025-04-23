다음 마크다운으로 구현 된 학술 논문을 한국어로 번역했습니다. 학술적인 용어와 맥락을 유지하면서 자연스러운 한국어로 번역했습니다. 이미지 태그 등은 그대로 유지했습니다.

# 스왑 수수료

서론에서 언급했듯이, 스왑 수수료는 유니스왑의 핵심 메커니즘입니다. 유동성 공급자들은 자신들이 제공하는 유동성에 대한 보상을 받아야 합니다. 그렇지 않으면 다른 곳에 유동성을 활용할 것이기 때문입니다. 유동성 공급자에게 인센티브를 제공하기 위해, 거래는 각 스왑마다 소량의 수수료를 지불합니다. 이 수수료는 총 풀 유동성에서 각자의 지분에 비례하여 모든 유동성 공급자에게 분배됩니다.

수수료 징수 및 분배 메커니즘을 더 잘 이해하기 위해, 작동 방식에 대해 자세히 살펴보겠습니다.

## 스왑 수수료 징수 방법



![유동성 범위 및 수수료](images/liquidity_ranges_fees.png)

스왑 수수료는 가격 범위가 활성화될 때만 (거래에 사용될 때) 징수됩니다. 따라서 가격 범위 경계가 교차되는 시점을 추적해야 합니다. 가격 범위가 활성화되는 시점은 다음과 같으며, 이때 수수료 징수를 시작합니다.
1. 가격이 상승하고 틱이 왼쪽에서 오른쪽으로 교차될 때;
2. 가격이 하락하고 틱이 오른쪽에서 왼쪽으로 교차될 때.

가격 범위가 비활성화되는 시점은 다음과 같습니다.
1. 가격이 상승하고 틱이 오른쪽에서 왼쪽으로 교차될 때;
2. 가격이 하락하고 틱이 왼쪽에서 오른쪽으로 교차될 때.



![유동성 범위 활성화/비활성화](images/liquidity_range_engaged.png)

가격 범위가 활성화/비활성화되는 시점을 파악하는 것 외에도, 각 가격 범위가 누적한 수수료를 추적해야 합니다.

수수료 회계 처리를 단순화하기 위해, 유니스왑 V3는 **유동성 1단위당 생성된 전역 수수료**를 추적합니다. 가격 범위 수수료는 전역 수수료를 기반으로 계산됩니다. 가격 범위 외부에서 누적된 수수료는 전역 수수료에서 차감됩니다. 가격 범위 외부에서 누적된 수수료는 틱이 교차될 때 추적됩니다 (그리고 틱은 스왑이 가격을 움직일 때 교차되며, 수수료는 스왑 중에 징수됩니다). 이러한 접근 방식을 통해, 모든 스왑에서 각 포지션별로 누적된 수수료를 업데이트할 필요가 없어 가스비를 크게 절약하고 풀과의 상호 작용 비용을 절감할 수 있습니다.

계속 진행하기 전에 명확히 정리하기 위해 요약해 보겠습니다.
1. 수수료는 토큰을 스왑하는 사용자가 지불합니다. 소량의 금액이 입력 토큰에서 차감되어 풀의 잔액에 누적됩니다.
2. 각 풀에는 유동성 단위당 총 누적 수수료를 추적하는 `feeGrowthGlobal0X128` 및 `feeGrowthGlobal1X128` 상태 변수가 있습니다 (즉, 수수료 금액을 풀의 유동성으로 나눈 값).
3. 가스 사용량을 최적화하기 위해 실제 포지션은 이 시점에서 업데이트되지 않습니다.
4. 틱은 자신을 기준으로 외부에서 누적된 수수료 기록을 보관합니다. 새로운 포지션을 추가하고 틱을 활성화할 때 (이전에 비어 있던 틱에 유동성을 추가할 때), 틱은 외부에서 누적된 수수료를 기록합니다 (관례상, 모든 수수료는 **틱 아래에서** 누적되었다고 가정합니다).
5. 틱이 활성화될 때마다, 틱 외부에서 누적된 수수료는 틱 외부에서 누적된 전역 수수료와 마지막으로 교차된 이후 틱 외부에서 누적된 수수료 간의 차이로 업데이트됩니다.
6. 틱 외부에 누적된 수수료를 알고 있는 틱을 통해 포지션 내부에서 누적된 수수료를 계산할 수 있습니다 (포지션은 두 틱 사이의 범위입니다).
7. 포지션 내부에서 누적된 수수료를 알면 유동성 공급자가 받을 수 있는 수수료 지분을 계산할 수 있습니다. 포지션이 스왑에 관여하지 않은 경우, 포지션 내부에 누적된 수수료는 0이 되며, 이 범위에 유동성을 제공한 유동성 공급자는 수익을 얻지 못합니다.

이제 포지션에서 누적된 수수료를 계산하는 방법 (6단계)을 살펴보겠습니다.

## 포지션 누적 수수료 계산

포지션에서 누적된 총 수수료를 계산하려면, 현재 가격이 포지션 내부에 있는지 외부에 있는지 두 가지 경우를 고려해야 합니다. 두 경우 모두, 포지션의 하단 및 상단 틱 외부에서 징수된 수수료를 전역적으로 징수된 수수료에서 차감합니다. 그러나 이러한 수수료는 현재 가격에 따라 다르게 계산합니다.

현재 가격이 포지션 내부에 있는 경우, 이때까지 틱 외부에서 징수된 수수료를 차감합니다.



![가격 범위 내부 및 외부에서 발생한 수수료](images/fees_inside_and_outside_price_range.png)

현재 가격이 포지션 외부에 있는 경우, 상단 또는 하단 틱에서 징수된 수수료를 업데이트한 후 전역적으로 징수된 수수료에서 차감해야 합니다. 틱이 교차되지 않으므로 계산 시에만 업데이트하고 틱에 덮어쓰지는 않습니다.

틱 외부에서 징수된 수수료를 업데이트하는 방법은 다음과 같습니다.

$$f_{o}(i) = f_{g} - f_{o}(i)$$

틱 외부에서 징수된 수수료 ($f_{o}(i)$)는 전역적으로 징수된 수수료 ($f_{g}$)와 마지막으로 교차했을 때 틱 외부에서 징수된 수수료 간의 차이입니다. 틱이 교차될 때 카운터를 재설정하는 것과 유사합니다.

포지션 내부에서 징수된 수수료를 계산하는 방법은 다음과 같습니다.

$$f_{r} = f_{g} - f_{b}(i_{l}) - f_{a}(i_{u})$$

하단 틱 아래에서 징수된 수수료 ($f_{b}(i_{l})$)와 상단 틱 위에서 징수된 수수료 ($f_{a}(i_{u})$)를 모든 가격 범위에서 전역적으로 징수된 수수료 ($f_{g}$)에서 차감합니다. 이것이 위 그림에서 본 내용입니다.

이제 현재 가격이 하단 틱 위에 있는 경우 (즉, 포지션이 활성화된 경우), 하단 틱 아래에서 누적된 수수료를 업데이트할 필요 없이 하단 틱에서 가져오면 됩니다. 현재 가격이 상단 틱 아래에 있는 경우 상단 틱 외부에서 징수된 수수료에 대해서도 마찬가지입니다. 다른 두 경우에는 업데이트된 수수료를 고려해야 합니다.
1. 하단 틱 아래에서 징수된 수수료를 가져오고 현재 가격도 틱 아래에 있는 경우 (하단 틱이 최근에 교차되지 않은 경우).
2. 상단 틱 위에서 징수된 수수료를 가져오고 현재 가격도 틱 위에 있는 경우 (상단 틱이 최근에 교차되지 않은 경우).

너무 혼란스럽지 않기를 바랍니다. 다행히 이제 코딩을 시작하기 위한 모든 것을 알게 되었습니다!

## 스왑 수수료 발생

간단하게 유지하기 위해, 수수료를 코드베이스에 단계별로 추가할 것입니다. 그리고 스왑 수수료 발생부터 시작하겠습니다.

### 필수 상태 변수 추가
가장 먼저 해야 할 일은 Pool에 수수료 금액 매개변수를 추가하는 것입니다. 모든 풀은 배포 중에 고정되고 불변하는 수수료로 구성됩니다. 이전 장에서 풀 배포를 통합하고 단순화한 Factory 컨트랙트를 추가했습니다. 필수 풀 매개변수 중 하나는 틱 간격이었습니다. 이제 틱 간격을 수수료 금액으로 대체하고 수수료 금액을 틱 간격과 연결할 것입니다. 수수료 금액이 클수록 틱 간격이 커집니다. 이는 낮은 변동성 풀 (스테이블 코인 풀)의 수수료를 낮추기 위함입니다.

Factory를 업데이트해 보겠습니다.
```solidity
// src/UniswapV3Factory.sol
contract UniswapV3Factory is IUniswapV3PoolDeployer {
    ...
    mapping(uint24 => uint24) public fees; // `tickSpacings`가 `fees`로 대체됨

    constructor() {
        fees[500] = 10;
        fees[3000] = 60;
    }

    function createPool(
        address tokenX,
        address tokenY,
        uint24 fee
    ) public returns (address pool) {
        ...
        parameters = PoolParameters({
            factory: address(this),
            token0: tokenX,
            token1: tokenY,
            tickSpacing: fees[fee],
            fee: fee
        });
        ...
    }
}
```

수수료 금액은 베이시스 포인트의 100분의 1입니다. 즉, 1 수수료 단위는 0.0001%, 500은 0.05%, 3000은 0.3%입니다.

다음 단계는 풀에서 수수료 누적을 시작하는 것입니다. 이를 위해 두 개의 전역 수수료 누적기 변수를 추가합니다.
```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    ...
    uint24 public immutable fee;
    uint256 public feeGrowthGlobal0X128;
    uint256 public feeGrowthGlobal1X128;
}
```

인덱스 0은 `token0`에서 누적된 수수료를 추적하고, 인덱스 1은 `token1`에서 누적된 수수료를 추적합니다.

### 수수료 징수

이제 `SwapMath.computeSwapStep`을 업데이트해야 합니다. 이곳에서 스왑 금액을 계산하고 스왑 수수료를 계산하고 차감할 것입니다. 함수에서 `amountRemaining`이 나타나는 모든 항목을 `amountRemainingLessFee`로 바꿉니다.
```solidity
uint256 amountRemainingLessFee = PRBMath.mulDiv(
    amountRemaining,
    1e6 - fee,
    1e6
);
```

따라서 입력 토큰 금액에서 수수료를 차감하고 더 작은 입력 금액에서 출력 금액을 계산합니다.

이제 함수는 단계별로 징수된 수수료 금액도 반환합니다. 범위의 상한선에 도달했는지 여부에 따라 다르게 계산됩니다.
```solidity
bool max = sqrtPriceNextX96 == sqrtPriceTargetX96;
if (!max) {
    feeAmount = amountRemaining - amountIn;
} else {
    feeAmount = Math.mulDivRoundingUp(amountIn, fee, 1e6 - fee);
}
```
도달하지 않은 경우, 현재 가격 범위는 스왑을 충족하기에 충분한 유동성을 가지므로 충족해야 할 금액과 실제로 충족된 금액의 차이를 반환합니다. `amountRemainingLessFee`는 실제 최종 금액이 `amountIn`에서 계산되었으므로 (사용 가능한 유동성을 기반으로 계산됨) 여기서 관여하지 않습니다.

목표 가격에 도달하면 현재 가격 범위가 스왑을 충족하기에 충분한 유동성을 가지고 있지 않으므로 전체 `amountRemaining`에서 수수료를 차감할 수 없습니다. 따라서 수수료 금액은 현재 가격 범위가 충족한 금액 (`amountIn`)에서 차감됩니다.

`SwapMath.computeSwapStep`이 반환된 후, 스왑으로 누적된 수수료를 업데이트해야 합니다. 스왑을 시작할 때 입력 토큰을 이미 알고 있기 때문에 (스왑 중에 수수료는 `token0` 또는 `token1` 중 하나에서 징수되며 둘 다는 아님) 이를 추적하는 변수는 하나뿐입니다.
```solidity
SwapState memory state = SwapState({
    ...
    feeGrowthGlobalX128: zeroForOne
        ? feeGrowthGlobal0X128
        : feeGrowthGlobal1X128
});

(...) = SwapMath.computeSwapStep(...);

state.feeGrowthGlobalX128 += PRBMath.mulDiv(
    step.feeAmount,
    FixedPoint128.Q128,
    state.liquidity
);
```

여기서 나중에 유동성 공급자에게 공정한 방식으로 수수료를 분배하기 위해 누적된 수수료를 유동성 금액으로 조정합니다.

### 틱에서 수수료 추적기 업데이트

다음으로, 스왑 중에 틱이 교차된 경우 (틱 교차는 새로운 가격 범위에 진입하는 것을 의미함) 틱에서 수수료 추적기를 업데이트해야 합니다.
```solidity
if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
    int128 liquidityDelta = ticks.cross(
        step.nextTick,
        (
            zeroForOne
                ? state.feeGrowthGlobalX128
                : feeGrowthGlobal0X128
        ),
        (
            zeroForOne
                ? feeGrowthGlobal1X128
                : state.feeGrowthGlobalX128
        )
    );
    ...
}
```

이 시점에서 `feeGrowthGlobal0X128/feeGrowthGlobal1X128` 상태 변수를 아직 업데이트하지 않았으므로, 스왑 방향에 따라 수수료 매개변수 중 하나로 `state.feeGrowthGlobalX128`을 전달합니다. `cross` 함수는 위에서 논의한 대로 수수료 추적기를 업데이트합니다.
```solidity
// src/lib/Tick.sol
function cross(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128
) internal returns (int128 liquidityDelta) {
    Tick.Info storage info = self[tick];
    info.feeGrowthOutside0X128 =
        feeGrowthGlobal0X128 -
        info.feeGrowthOutside0X128;
    info.feeGrowthOutside1X128 =
        feeGrowthGlobal1X128 -
        info.feeGrowthOutside1X128;
    liquidityDelta = info.liquidityNet;
}
```

> `feeGrowthOutside0X128/feeGrowthOutside1X128` 변수의 초기화를 아직 추가하지 않았습니다. 이 부분은 나중에 추가하겠습니다.

### 전역 수수료 추적기 업데이트

마지막으로, 스왑이 완료된 후 전역 수수료 추적기를 업데이트할 수 있습니다.
```solidity
if (zeroForOne) {
    feeGrowthGlobal0X128 = state.feeGrowthGlobalX128;
} else {
    feeGrowthGlobal1X128 = state.feeGrowthGlobalX128;
}
```
반복적으로 말하지만, 스왑 중에는 스왑 방향에 따라 `token0` 또는 `token1` 중 하나인 입력 토큰에서 수수료가 징수되므로 이들 중 하나만 업데이트됩니다.

스왑에 대한 내용은 여기까지입니다! 이제 유동성이 추가될 때 수수료에 어떤 일이 발생하는지 살펴보겠습니다.

## 포지션 관리에서 수수료 추적

유동성을 추가하거나 제거할 때 (아직 후자는 구현하지 않음), 수수료를 초기화하거나 업데이트해야 합니다. 수수료는 틱 (틱 외부에서 누적된 수수료 - 방금 추가한 `feeGrowthOutside` 변수)과 포지션 (포지션 내부에서 누적된 수수료) 모두에서 추적해야 합니다. 포지션의 경우, 수수료로 징수된 토큰 금액도 추적하고 업데이트해야 합니다. 즉, 유동성당 수수료를 토큰 금액으로 변환해야 합니다. 후자는 유동성 공급자가 유동성을 제거할 때 스왑 수수료로 징수된 추가 토큰을 얻을 수 있도록 하기 위해 필요합니다.

다시 단계별로 진행해 보겠습니다.

### 틱에서 수수료 추적기 초기화

`Tick.update` 함수에서, 틱이 초기화될 때마다 (이전에 비어 있던 틱에 유동성을 추가할 때) 수수료 추적기를 초기화합니다. 그러나 틱이 현재 가격보다 낮을 때, 즉 현재 가격 범위 내에 있을 때만 초기화합니다.

```solidity
// src/lib/Tick.sol
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    int24 currentTick,
    int128 liquidityDelta,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128,
    bool upper
) internal returns (bool flipped) {
    ...
    if (liquidityBefore == 0) {
        // 관례상, 이전의 모든 수수료는 틱 아래에서 징수되었다고 가정합니다.
        // 틱 아래에서
        if (tick <= currentTick) {
            tickInfo.feeGrowthOutside0X128 = feeGrowthGlobal0X128;
            tickInfo.feeGrowthOutside1X128 = feeGrowthGlobal1X128;
        }

        tickInfo.initialized = true;
    }
    ...
}
```

현재 가격 범위 내에 있지 않으면 수수료 추적기는 0이 되며, 다음에 틱이 교차될 때 업데이트됩니다 (위에서 업데이트한 `cross` 함수 참조).

### 포지션 수수료 및 토큰 금액 업데이트

다음 단계는 포지션에서 누적된 수수료와 토큰을 계산하는 것입니다. 포지션은 두 틱 사이의 범위이므로, 이전 단계에서 틱에 추가한 수수료 추적기를 사용하여 이러한 값을 계산합니다. 다음 함수는 복잡해 보일 수 있지만, 앞에서 본 가격 범위 수수료 공식을 정확히 구현합니다.
```solidity
// src/lib/Tick.sol
function getFeeGrowthInside(
    mapping(int24 => Tick.Info) storage self,
    int24 lowerTick_,
    int24 upperTick_,
    int24 currentTick,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128
)
    internal
    view
    returns (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128)
{
    Tick.Info storage lowerTick = self[lowerTick_];
    Tick.Info storage upperTick = self[upperTick_];

    uint256 feeGrowthBelow0X128;
    uint256 feeGrowthBelow1X128;
    if (currentTick >= lowerTick_) {
        feeGrowthBelow0X128 = lowerTick.feeGrowthOutside0X128;
        feeGrowthBelow1X128 = lowerTick.feeGrowthOutside1X128;
    } else {
        feeGrowthBelow0X128 =
            feeGrowthGlobal0X128 -
            lowerTick.feeGrowthOutside0X128;
        feeGrowthBelow1X128 =
            feeGrowthGlobal1X128 -
            lowerTick.feeGrowthOutside1X128;
    }

    uint256 feeGrowthAbove0X128;
    uint256 feeGrowthAbove1X128;
    if (currentTick < upperTick_) {
        feeGrowthAbove0X128 = upperTick.feeGrowthOutside0X128;
        feeGrowthAbove1X128 = upperTick.feeGrowthOutside1X128;
    } else {
        feeGrowthAbove0X128 =
            feeGrowthGlobal0X128 -
            upperTick.feeGrowthOutside0X128;
        feeGrowthAbove1X128 =
            feeGrowthGlobal1X128 -
            upperTick.feeGrowthOutside1X128;
    }

    feeGrowthInside0X128 =
        feeGrowthGlobal0X128 -
        feeGrowthBelow0X128 -
        feeGrowthAbove0X128;
    feeGrowthInside1X128 =
        feeGrowthGlobal1X128 -
        feeGrowthBelow1X128 -
        feeGrowthAbove1X128;
}
```

여기서는 두 틱 사이에서 누적된 수수료 (가격 범위 내부)를 계산합니다. 이를 위해 먼저 하단 틱 아래에서 누적된 수수료를 계산한 다음 상단 틱 위에서 계산된 수수료를 계산합니다. 마지막으로, 이러한 수수료를 전역적으로 누적된 수수료에서 차감합니다. 이것이 앞에서 본 공식입니다.

$$f_{r} = f_{g} - f_{b}(i_{l}) - f_{a}(i_{u})$$

틱 위와 아래에서 징수된 수수료를 계산할 때, 가격 범위가 활성화되었는지 여부 (현재 가격이 가격 범위의 경계 틱 사이에 있는지 여부)에 따라 다르게 계산합니다. 활성화된 경우 틱의 현재 수수료 추적기를 사용하기만 하면 됩니다. 활성화되지 않은 경우 틱의 업데이트된 수수료 추적기를 가져와야 합니다. 이러한 계산은 위의 코드에서 두 개의 `else` 분기에서 확인할 수 있습니다.

포지션 내부에서 누적된 수수료를 찾은 후, 포지션의 수수료 및 토큰 금액 추적기를 업데이트할 준비가 되었습니다.
```solidity
// src/lib/Position.sol
function update(
    Info storage self,
    int128 liquidityDelta,
    uint256 feeGrowthInside0X128,
    uint256 feeGrowthInside1X128
) internal {
    uint128 tokensOwed0 = uint128(
        PRBMath.mulDiv(
            feeGrowthInside0X128 - self.feeGrowthInside0LastX128,
            self.liquidity,
            FixedPoint128.Q128
        )
    );
    uint128 tokensOwed1 = uint128(
        PRBMath.mulDiv(
            feeGrowthInside1X128 - self.feeGrowthInside1LastX128,
            self.liquidity,
            FixedPoint128.Q128
        )
    );

    self.liquidity = LiquidityMath.addLiquidity(
        self.liquidity,
        liquidityDelta
    );
    self.feeGrowthInside0LastX128 = feeGrowthInside0X128;
    self.feeGrowthInside1LastX128 = feeGrowthInside1X128;

    if (tokensOwed0 > 0 || tokensOwed1 > 0) {
        self.tokensOwed0 += tokensOwed0;
        self.tokensOwed1 += tokensOwed1;
    }
}
```

지불해야 할 토큰을 계산할 때, 스왑 중에 수행한 것의 반대로 포지션에서 누적된 수수료에 유동성을 곱합니다. 마지막으로, 수수료 추적기를 업데이트하고 토큰 금액을 이전에 추적된 금액에 추가합니다.

이제 포지션이 수정될 때마다 (유동성 추가 또는 제거 중), 포지션에서 징수된 수수료를 계산하고 포지션을 업데이트합니다.
```solidity
// src/UniswapV3Pool.sol
function mint(...) {
    ...
    bool flippedLower = ticks.update(params.lowerTick, ...);
    bool flippedUpper = ticks.update(params.upperTick, ...);
    ...
    (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) = ticks
        .getFeeGrowthInside(
            params.lowerTick,
            params.upperTick,
            slot0_.tick,
            feeGrowthGlobal0X128_,
            feeGrowthGlobal1X128_
        );

    position.update(
        params.liquidityDelta,
        feeGrowthInside0X128,
        feeGrowthInside1X128
    );
    ...
}
```

## 유동성 제거

이제 아직 구현하지 않은 유일한 핵심 기능인 유동성 제거를 추가할 준비가 되었습니다. 민팅과 반대로 이 함수를 `burn`이라고 부르겠습니다. 이 함수는 유동성 공급자가 이전에 유동성을 추가한 포지션에서 유동성의 일부 또는 전체를 제거할 수 있도록 합니다. 또한 유동성 공급자가 받을 수 있는 수수료 토큰도 계산합니다. 그러나 실제 토큰 전송은 별도의 함수인 `collect`에서 수행됩니다.

### 유동성 소각

유동성 소각은 민팅의 반대입니다. 현재 설계 및 구현은 번거로움 없는 작업으로 만듭니다. 유동성 소각은 단순히 음수 부호로 민팅하는 것과 같습니다. 음수 금액의 유동성을 추가하는 것과 같습니다.

> `burn`을 구현하기 위해, 코드 리팩토링이 필요했으며 포지션 관리 (틱 및 포지션 업데이트, 토큰 금액 계산)와 관련된 모든 것을 `_modifyPosition` 함수로 추출했습니다. 이 함수는 `mint` 및 `burn` 함수 모두에서 사용됩니다.

```solidity
function burn(
    int24 lowerTick,
    int24 upperTick,
    uint128 amount
) public returns (uint256 amount0, uint256 amount1) {
    (
        Position.Info storage position,
        int256 amount0Int,
        int256 amount1Int
    ) = _modifyPosition(
            ModifyPositionParams({
                owner: msg.sender,
                lowerTick: lowerTick,
                upperTick: upperTick,
                liquidityDelta: -(int128(amount))
            })
        );

    amount0 = uint256(-amount0Int);
    amount1 = uint256(-amount1Int);

    if (amount0 > 0 || amount1 > 0) {
        (position.tokensOwed0, position.tokensOwed1) = (
            position.tokensOwed0 + uint128(amount0),
            position.tokensOwed1 + uint128(amount1)
        );
    }

    emit Burn(msg.sender, lowerTick, upperTick, amount, amount0, amount1);
}
```

`burn` 함수에서 먼저 포지션을 업데이트하고 포지션에서 유동성 일부를 제거합니다. 그런 다음 포지션에서 지불해야 할 토큰 금액을 업데이트합니다. 이제 수수료를 통해 누적된 금액과 이전에 유동성으로 제공된 금액을 포함합니다. 포지션 유동성을 포지션에서 지불해야 할 토큰 금액으로 변환하는 것으로 볼 수도 있습니다. 이러한 금액은 더 이상 유동성으로 사용되지 않으며 `collect` 함수를 호출하여 자유롭게 상환할 수 있습니다.

```solidity
function collect(
    address recipient,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount0Requested,
    uint128 amount1Requested
) public returns (uint128 amount0, uint128 amount1) {
    Position.Info storage position = positions.get(
        msg.sender,
        lowerTick,
        upperTick
    );

    amount0 = amount0Requested > position.tokensOwed0
        ? position.tokensOwed0
        : amount0Requested;
    amount1 = amount1Requested > position.tokensOwed1
        ? position.tokensOwed1
        : amount1Requested;

    if (amount0 > 0) {
        position.tokensOwed0 -= amount0;
        IERC20(token0).transfer(recipient, amount0);
    }

    if (amount1 > 0) {
        position.tokensOwed1 -= amount1;
        IERC20(token1).transfer(recipient, amount1);
    }

    emit Collect(
        msg.sender,
        recipient,
        lowerTick,
        upperTick,
        amount0,
        amount1
    );
}
```

이 함수는 단순히 풀에서 토큰을 전송하고 유효한 금액만 전송할 수 있도록 합니다 (소각한 금액 + 획득한 수수료보다 더 많이 전송할 수 없음).

유동성을 소각하지 않고 수수료만 징수하는 방법도 있습니다. 유동성 0 금액을 소각한 다음 `collect`를 호출합니다. 소각하는 동안 포지션이 업데이트되고 포지션에서 지불해야 할 토큰 금액도 업데이트됩니다.

이것으로 끝입니다! 이제 풀 구현이 완료되었습니다!