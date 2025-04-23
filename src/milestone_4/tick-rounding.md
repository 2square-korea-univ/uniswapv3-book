# 틱 반올림

다양한 틱 간격을 지원하기 위해 필요한 다른 변경 사항들을 살펴보겠습니다.

틱 간격이 1보다 클 경우, 사용자는 임의의 가격 범위를 자유롭게 선택할 수 없습니다. 틱 인덱스는 반드시 틱 간격의 배수여야만 합니다. 예를 들어, 틱 간격이 60이라면 틱은 0, 60, 120, 180 등과 같이 틱 간격의 배수 값만 가능합니다. 따라서 사용자가 가격 범위를 설정할 때, 풀의 틱 간격의 배수가 되도록 경계를 "반올림"해야 합니다.

## JavaScript의 `nearestUsableTick`

[Uniswap V3 SDK](https://github.com/Uniswap/v3-sdk)에서 이 기능을 수행하는 함수는 [nearestUsableTick](https://github.com/Uniswap/v3-sdk/blob/b6cd73a71f8f8ec6c40c130564d3aff12c38e693/src/utils/nearestUsableTick.ts)입니다.

```javascript
/**
 * 주어진 틱과 틱 간격에 대해 가장 가까운 사용 가능한 틱을 반환합니다.
 * @param tick 목표 틱
 * @param tickSpacing 풀의 틱 간격
 */
export function nearestUsableTick(tick: number, tickSpacing: number) {
  invariant(Number.isInteger(tick) && Number.isInteger(tickSpacing), '정수여야 합니다')
  invariant(tickSpacing > 0, 'TICK_SPACING')
  invariant(tick >= TickMath.MIN_TICK && tick <= TickMath.MAX_TICK, 'TICK_BOUND')
  const rounded = Math.round(tick / tickSpacing) * tickSpacing
  if (rounded < TickMath.MIN_TICK) return rounded + tickSpacing
  else if (rounded > TickMath.MAX_TICK) return rounded - tickSpacing
  else return rounded
}
```

핵심 코드는 다음과 같습니다:

```javascript
Math.round(tick / tickSpacing) * tickSpacing
```

여기서 `Math.round`는 가장 가까운 정수로 반올림하는 함수입니다. 소수점 아래 값이 0.5 미만일 때는 내림하고, 0.5 이상일 때는 올림합니다. 특히, 정확히 0.5일 경우에도 올림합니다.

따라서 웹 애플리케이션에서는 `mint` 매개변수를 생성할 때 `nearestUsableTick` 함수를 사용합니다:

```javascript
const mintParams = {
  tokenA: pair.token0.address,
  tokenB: pair.token1.address,
  tickSpacing: pair.tickSpacing,
  lowerTick: nearestUsableTick(lowerTick, pair.tickSpacing),
  upperTick: nearestUsableTick(upperTick, pair.tickSpacing),
  amount0Desired, amount1Desired, amount0Min, amount1Min
}
```

> 실제로는 사용자가 가격 범위를 조정할 때마다 이 함수를 호출해야 합니다. 이는 사용자가 실제로 생성될 가격 범위를 실시간으로 확인할 수 있도록 하기 위함입니다. 하지만 여기서는 단순화된 앱을 위해 사용자 편의성을 다소 희생하여 구현했습니다.

하지만 Solidity 테스트 환경에서도 이와 유사한 함수가 필요합니다. 하지만 현재 사용 중인 어떤 수학 라이브러리에서도 이 함수를 제공하지 않습니다.

## Solidity의 `nearestUsableTick`

스마트 컨트랙트 테스트 환경에서는 틱을 반올림하고, 반올림된 틱 값을 $\sqrt{P}$ 값으로 변환하는 기능이 필요합니다. 이전 장에서 테스트 환경에서 고정 소수점 연산을 처리하기 위해 [ABDKMath64x64](https://github.com/abdk-consulting/abdk-libraries-solidity) 라이브러리를 사용하기로 결정했습니다. 그러나 이 라이브러리에는 `nearestUsableTick` 함수와 동일한 반올림 기능이 없으므로, 직접 구현해야 합니다.

```solidity
function divRound(int128 x, int128 y)
    internal
    pure
    returns (int128 result)
{
    int128 quot = ABDKMath64x64.div(x, y);
    result = quot >> 64;

    // Check if remainder is greater than 0.5
    if (quot % 2**64 >= 0x8000000000000000) {
        result += 1;
    }
}
```

이 `divRound` 함수는 다음과 같은 단계를 거쳐 동작합니다:

1. 두 개의 Q64.64 형식 숫자를 나눕니다.
2. 나눗셈 결과에서 정수 부분만을 취하여 `result`에 저장합니다 (`result = quot >> 64`). 이 과정은 소수점 이하를 버리는 것과 같습니다 (즉, 내림 연산과 유사).
3. 몫을 $2^{64}$으로 나눈 나머지를 계산하고, 이 나머지를 `0x8000000000000000` (Q64.64 형식에서 0.5에 해당)와 비교합니다.
4. 나머지가 0.5 (Q64.64 형식) 이상이면 `result` 값을 1 증가시켜 올림합니다.

이 함수를 통해 JavaScript의 `Math.round` 함수와 동일한 방식으로 정수 반올림을 수행할 수 있습니다. 이제 이 `divRound` 함수를 사용하여 Solidity 환경에서 `nearestUsableTick` 함수를 구현할 수 있습니다:

```solidity
function nearestUsableTick(int24 tick_, uint24 tickSpacing)
    internal
    pure
    returns (int24 result)
{
    result =
        int24(divRound(int128(tick_), int128(int24(tickSpacing)))) *
        int24(tickSpacing);

    if (result < TickMath.MIN_TICK) {
        result += int24(tickSpacing);
    } else if (result > TickMath.MAX_TICK) {
        result -= int24(tickSpacing);
    }
}
```

이것으로 충분합니다!