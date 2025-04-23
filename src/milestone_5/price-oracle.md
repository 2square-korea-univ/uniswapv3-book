## 가격 오라클

DEX에 마지막으로 추가할 메커니즘은 *가격 오라클*입니다. DEX에 필수적인 기능은 아니지만 (일부 DEX는 가격 오라클을 구현하지 않음), Uniswap의 중요한 기능이며 학습할 가치가 있습니다.

## 가격 오라클이란 무엇인가?

가격 오라클은 블록체인에 자산 가격을 제공하는 메커니즘입니다. 블록체인은 고립된 생태계이기 때문에 외부 데이터를 직접 쿼리할 방법이 없습니다. 예를 들어 API를 통해 중앙화 거래소에서 자산 가격을 가져오는 것은 불가능합니다. 또 다른 매우 어려운 문제는 데이터 유효성과 진위성입니다. 거래소에서 가격을 가져올 때, 그 가격이 진짜인지 어떻게 알 수 있을까요? 출처를 신뢰해야 합니다. 하지만 인터넷은 종종 안전하지 않으며, 때로는 가격이 조작될 수 있고, DNS 레코드가 탈취될 수 있으며, API 서버가 다운될 수도 있습니다. 신뢰성 있고 정확한 온체인 가격을 확보하기 위해서는 이러한 모든 어려움을 해결해야 합니다.

위에서 언급한 문제에 대한 최초의 실용적인 해결책 중 하나는 [Chainlink](https://chain.link/)였습니다. Chainlink는 API를 통해 중앙화 거래소에서 자산 가격을 가져와 평균을 내고, 위변조 방지 방식으로 온체인에 제공하는 탈중앙화 오라클 네트워크를 운영합니다. Chainlink는 누구나 (다른 컨트랙트 또는 사용자) 읽을 수 있지만 오라클만 쓸 수 있는 상태 변수, 즉 자산 가격을 가진 컨트랙트 집합입니다.

이것이 가격 오라클을 바라보는 한 가지 방식입니다. 다른 방식도 있습니다.

기본적인 온체인 거래소가 있다면 왜 외부에서 가격을 가져와야 할까요? 이것이 Uniswap 가격 오라클의 작동 방식입니다. 차익 거래와 높은 유동성 덕분에 Uniswap의 자산 가격은 중앙화 거래소의 가격과 거의 비슷합니다. 따라서 중앙화 거래소를 자산 가격의 진실 공급원으로 사용하는 대신, Uniswap을 사용할 수 있으며, 온체인으로 데이터를 전달하는 것과 관련된 문제를 해결할 필요가 없습니다 (데이터 제공자를 신뢰할 필요도 없습니다).

## Uniswap 가격 오라클 작동 방식

Uniswap은 단순히 모든 이전 스왑 가격 기록을 보관합니다. 그게 전부입니다. 하지만 실제 가격을 추적하는 대신, Uniswap은 *누적 가격*을 추적합니다. 누적 가격은 풀 컨트랙트의 역사에서 매 초마다의 가격 합계입니다.

$$a_{i} = \sum_{i=1}^t p_{i}$$

이 접근 방식을 통해 두 시점 ($t_1$과 $t_2$) 사이의 *시간 가중 평균 가격*을 간단히 찾을 수 있습니다. 해당 시점의 누적 가격 ($a_{t_1}$과 $a_{t_2}$)을 가져와서 하나에서 다른 하나를 빼고, 두 시점 사이의 초 단위 시간으로 나누면 됩니다.

$$p_{t_1,t_2} = \frac{a_{t_2} - a_{t_1}}{t_2 - t_1}$$

이것이 Uniswap V2에서 작동하는 방식이었습니다. V3에서는 약간 다릅니다. 누적 가격은 현재 틱입니다 (틱은 가격의 $log_{1.0001}$ 값).

$$a_{i} = \sum_{i=1}^t log_{1.0001}P(i)$$

그리고 가격을 평균하는 대신, *기하 평균*이 사용됩니다.

$$ P_{t_1,t_2} = \left( \prod_{i=t_1}^{t_2} P_i \right) ^ \frac{1}{t_2-t_1} $$

두 시점 사이의 시간 가중 기하 평균 가격을 찾으려면, 해당 시점의 누적 값을 가져와서 하나에서 다른 하나를 빼고, 두 시점 사이의 초 단위 시간으로 나누고, $1.0001^{x}$를 계산합니다.

$$ log_{1.0001}{(P_{t_1,t_2})} = \frac{\sum_{i=t_1}^{t_2} log_{1.0001}(P_i)}{t_2-t_1}$$
$$ = \frac{a_{t_2} - a_{t_1}}{t_2-t_1}$$

$$P_{t_1,t_2} = 1.0001^{\frac{a_{t_2} - a_{t_1}}{t_2-t_1}}$$

Uniswap V2는 과거 누적 가격을 저장하지 않았기 때문에, 평균 가격을 계산할 때 과거 가격을 찾기 위해 타사 블록체인 데이터 인덱싱 서비스를 참조해야 했습니다. 반면 Uniswap V3는 최대 65,535개의 과거 누적 가격을 저장할 수 있어, 과거 시간 가중 기하 평균 가격을 훨씬 쉽게 계산할 수 있습니다.

## 가격 조작 완화

또 다른 중요한 주제는 가격 조작과 Uniswap에서 가격 조작을 완화하는 방법입니다.

이론적으로 풀의 가격을 유리하게 조작하는 것이 가능합니다. 예를 들어, 토큰을 대량으로 구매하여 가격을 올리고 Uniswap 가격 오라클을 사용하는 타사 DeFi 서비스에서 이익을 얻은 다음, 토큰을 다시 실제 가격으로 거래하는 것입니다. 이러한 공격을 완화하기 위해 Uniswap은 블록 **마지막 거래 *후에*** 블록 **끝에서** 가격을 추적합니다. 이는 블록 내 가격 조작 가능성을 제거합니다.

기술적으로 Uniswap 오라클의 가격은 각 블록 시작 시에 업데이트되며, 각 가격은 블록의 첫 번째 스왑 전에 계산됩니다.

## 가격 오라클 구현

자, 이제 코드로 넘어가 보겠습니다.

### 관측 및 카디널리티

`Oracle` 라이브러리 컨트랙트와 `Observation` 구조체를 생성하는 것으로 시작하겠습니다.

```solidity
// src/lib/Oracle.sol
library Oracle {
    struct Observation {
        uint32 timestamp;
        int56 tickCumulative;
        bool initialized;
    }
    ...
}
```

*관측*은 기록된 가격을 저장하는 슬롯입니다. 관측은 가격, 가격이 기록된 타임스탬프, 그리고 관측이 활성화되었을 때 `true`로 설정되는 `initialized` 플래그를 저장합니다 (모든 관측이 기본적으로 활성화되는 것은 아님). 풀 컨트랙트는 최대 65,535개의 관측을 저장할 수 있습니다.

```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    using Oracle for Oracle.Observation[65535];
    ...
    Oracle.Observation[65535] public observations;
}
```

하지만 이렇게 많은 `Observation` 인스턴스를 저장하려면 많은 가스가 필요하기 때문에 (누군가가 각 인스턴스를 컨트랙트 스토리지에 쓰는 데 비용을 지불해야 함), 풀은 기본적으로 1개의 관측만 저장할 수 있으며, 새 가격이 기록될 때마다 덮어쓰여집니다. 활성화된 관측 수, 즉 관측의 *카디널리티*는 비용을 지불하려는 사람이 언제든지 늘릴 수 있습니다. 카디널리티를 관리하려면 몇 가지 추가 상태 변수가 필요합니다.
```solidity
    ...
    struct Slot0 {
        // 현재 sqrt(P)
        uint160 sqrtPriceX96;
        // 현재 틱
        int24 tick;
        // 가장 최근 관측 인덱스
        uint16 observationIndex;
        // 최대 관측 수
        uint16 observationCardinality;
        // 다음 최대 관측 수
        uint16 observationCardinalityNext;
    }
    ...
```

- `observationIndex`는 가장 최근 관측의 인덱스를 추적합니다.
- `observationCardinality`는 활성화된 관측 수를 추적합니다.
- `observationCardinalityNext`는 관측 배열이 확장될 수 있는 다음 카디널리티를 추적합니다.

관측은 고정 길이 배열에 저장되며, 새 관측이 저장되고 `observationCardinalityNext`가 `observationCardinality`보다 클 때 확장됩니다 (이는 카디널리티를 확장할 수 있음을 나타냄). 배열을 확장할 수 없는 경우 (다음 카디널리티 값이 현재 값과 같은 경우), 가장 오래된 관측이 덮어쓰여집니다. 즉, 관측은 인덱스 0에 저장되고, 다음 관측은 인덱스 1에 저장되는 식으로 진행됩니다.

풀이 생성될 때, `observationCardinality`와 `observationCardinalityNext`는 1로 설정됩니다.
```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    function initialize(uint160 sqrtPriceX96) public {
        ...

        (uint16 cardinality, uint16 cardinalityNext) = observations.initialize(
            _blockTimestamp()
        );

        slot0 = Slot0({
            sqrtPriceX96: sqrtPriceX96,
            tick: tick,
            observationIndex: 0,
            observationCardinality: cardinality,
            observationCardinalityNext: cardinalityNext
        });
    }
}
```

```solidity
// src/lib/Oracle.sol
library Oracle {
    ...
    function initialize(Observation[65535] storage self, uint32 time)
        internal
        returns (uint16 cardinality, uint16 cardinalityNext)
    {
        self[0] = Observation({
            timestamp: time,
            tickCumulative: 0,
            initialized: true
        });

        cardinality = 1;
        cardinalityNext = 1;
    }
    ...
}
```

### 관측 기록

`swap` 함수에서 현재 가격이 변경되면 관측이 observations 배열에 기록됩니다.

```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    function swap(...) public returns (...) {
        ...
        if (state.tick != slot0_.tick) {
            (
                uint16 observationIndex,
                uint16 observationCardinality
            ) = observations.write(
                    slot0_.observationIndex,
                    _blockTimestamp(),
                    slot0_.tick,
                    slot0_.observationCardinality,
                    slot0_.observationCardinalityNext
                );
            
            (
                slot0.sqrtPriceX96,
                slot0.tick,
                slot0.observationIndex,
                slot0.observationCardinality
            ) = (
                state.sqrtPriceX96,
                state.tick,
                observationIndex,
                observationCardinality
            );
        }
        ...
    }
}
```

여기서 관측되는 틱은 `slot0_.tick` ( `state.tick` 아님), 즉 스왑 전 가격입니다! 다음 구문에서 새 가격으로 업데이트됩니다. 이것이 앞에서 논의한 가격 조작 완화 방법입니다. Uniswap은 블록의 첫 번째 거래 **전**과 이전 블록의 마지막 거래 **후**의 가격을 추적합니다.

또한 각 관측은 `_blockTimestamp()`, 즉 현재 블록 타임스탬프로 식별됩니다. 이는 현재 블록에 대한 관측이 이미 있는 경우 가격이 기록되지 않음을 의미합니다. 현재 블록에 대한 관측이 없는 경우 (즉, 블록의 첫 번째 스왑인 경우), 가격이 기록됩니다. 이것은 가격 조작 완화 메커니즘의 일부입니다.

```solidity
// src/lib/Oracle.sol
function write(
    Observation[65535] storage self,
    uint16 index,
    uint32 timestamp,
    int24 tick,
    uint16 cardinality,
    uint16 cardinalityNext
) internal returns (uint16 indexUpdated, uint16 cardinalityUpdated) {
    Observation memory last = self[index];

    if (last.timestamp == timestamp) return (index, cardinality);

    if (cardinalityNext > cardinality && index == (cardinality - 1)) {
        cardinalityUpdated = cardinalityNext;
    } else {
        cardinalityUpdated = cardinality;
    }

    indexUpdated = (index + 1) % cardinalityUpdated;
    self[indexUpdated] = transform(last, timestamp, tick);
}
```

여기서 현재 블록에서 이미 관측이 이루어진 경우 관측이 건너뛰는 것을 볼 수 있습니다. 하지만 그러한 관측이 없는 경우, 새 관측을 저장하고 가능한 경우 카디널리티를 확장하려고 시도합니다. 모듈로 연산자 (`%`)는 관측 인덱스가 $[0, cardinality)$ 범위 내에 유지되도록 하고, 상한에 도달하면 0으로 재설정되도록 합니다.

이제 `transform` 함수를 살펴보겠습니다.

```solidity
function transform(
    Observation memory last,
    uint32 timestamp,
    int24 tick
) internal pure returns (Observation memory) {
    uint56 delta = timestamp - last.timestamp;

    return
        Observation({
            timestamp: timestamp,
            tickCumulative: last.tickCumulative +
                int56(tick) *
                int56(delta),
            initialized: true
        });
}
```

여기서 계산하는 것은 누적 가격입니다. 현재 틱에 마지막 관측 이후의 초 수를 곱하고, 마지막 누적 가격에 더합니다.

### 카디널리티 증가

이제 카디널리티가 어떻게 확장되는지 살펴보겠습니다.

누구나 언제든지 풀의 관측 카디널리티를 늘리고, 그렇게 하는 데 필요한 가스 비용을 지불할 수 있습니다. 이를 위해 풀 컨트랙트에 새로운 공개 함수를 추가할 것입니다.

```solidity
// src/UniswapV3Pool.sol
function increaseObservationCardinalityNext(
    uint16 observationCardinalityNext
) public {
    uint16 observationCardinalityNextOld = slot0.observationCardinalityNext;
    uint16 observationCardinalityNextNew = observations.grow(
        observationCardinalityNextOld,
        observationCardinalityNext
    );

    if (observationCardinalityNextNew != observationCardinalityNextOld) {
        slot0.observationCardinalityNext = observationCardinalityNextNew;
        emit IncreaseObservationCardinalityNext(
            observationCardinalityNextOld,
            observationCardinalityNextNew
        );
    }
}
```

그리고 Oracle에 새로운 함수를 추가합니다.

```solidity
// src/lib/Oracle.sol
function grow(
    Observation[65535] storage self,
    uint16 current,
    uint16 next
) internal returns (uint16) {
    if (next <= current) return current;

    for (uint16 i = current; i < next; i++) {
        self[i].timestamp = 1;
    }

    return next;
}
```

`grow` 함수에서 각 관측의 `timestamp` 필드를 0이 아닌 값으로 설정하여 새로운 관측을 할당합니다. `self`는 스토리지 변수이며, 요소에 값을 할당하면 배열 카운터가 업데이트되고 값이 컨트랙트 스토리지에 기록됩니다.

### 관측 읽기

드디어 이 장에서 가장 까다로운 부분인 관측 읽기에 도달했습니다. 계속 진행하기 전에 관측이 어떻게 저장되는지 다시 검토하여 더 나은 그림을 그려보겠습니다.

관측은 확장 가능한 고정 길이 배열에 저장됩니다.



![관측 배열](images/observations.png)

위에서 언급했듯이, 관측은 오버플로가 예상됩니다. 새 관측이 배열에 맞지 않으면 인덱스 0부터 시작하여 기록이 계속됩니다. 즉, 가장 오래된 관측이 덮어쓰여집니다.



![관측 래핑](images/observations_wrapping.png)

스왑이 모든 블록에서 발생하는 것은 아니기 때문에 모든 블록에 대해 관측이 저장된다는 보장은 없습니다. 따라서 관측이 기록되지 않은 블록이 있을 수 있으며, 이러한 관측 누락 기간이 길어질 수 있습니다. 물론 오라클이 보고하는 가격에 간격이 생기는 것을 원하지 않으므로, 시간 가중 평균 가격 (TWAP)을 사용하고 있습니다. TWAP을 사용하면 관측이 없었던 기간의 평균 가격을 가질 수 있습니다. TWAP을 사용하면 가격을 *보간*할 수 있습니다. 즉, 두 관측 사이에 선을 그릴 수 있습니다. 선의 각 점은 두 관측 사이의 특정 타임스탬프에서의 가격이 됩니다.



![보간된 가격](images/interpolated_prices.png)

따라서 관측을 읽는다는 것은 타임스탬프로 관측을 찾고, 관측 배열이 오버플로될 수 있다는 점을 고려하여 누락된 관측을 보간하는 것을 의미합니다 (예: 가장 오래된 관측이 배열에서 가장 최근 관측보다 뒤에 올 수 있음). 가스를 절약하기 위해 타임스탬프로 관측을 인덱싱하지 않으므로, 효율적인 검색을 위해 [이진 검색 알고리즘](https://en.wikipedia.org/wiki/Binary_search_algorithm)을 사용해야 합니다. 하지만 항상 그런 것은 아닙니다.

더 작은 단계로 나누어 `Oracle`에서 `observe` 함수를 구현하는 것으로 시작하겠습니다.

```solidity
function observe(
    Observation[65535] storage self,
    uint32 time,
    uint32[] memory secondsAgos,
    int24 tick,
    uint16 index,
    uint16 cardinality
) internal view returns (int56[] memory tickCumulatives) {
    tickCumulatives = new int56[](secondsAgos.length);

    for (uint256 i = 0; i < secondsAgos.length; i++) {
        tickCumulatives[i] = observeSingle(
            self,
            time,
            secondsAgos[i],
            tick,
            index,
            cardinality
        );
    }
}
```

이 함수는 현재 블록 타임스탬프, 가격을 가져오려는 시점 목록 (`secondsAgo`), 현재 틱, 관측 인덱스 및 카디널리티를 사용합니다.

`observeSingle` 함수로 넘어가겠습니다.

```solidity
function observeSingle(
    Observation[65535] storage self,
    uint32 time,
    uint32 secondsAgo,
    int24 tick,
    uint16 index,
    uint16 cardinality
) internal view returns (int56 tickCumulative) {
    if (secondsAgo == 0) {
        Observation memory last = self[index];
        if (last.timestamp != time) last = transform(last, time, tick);
        return last.tickCumulative;
    }
    ...
}
```

가장 최근 관측이 요청되면 (0초 경과), 즉시 반환할 수 있습니다. 현재 블록에 기록되지 않은 경우, 현재 블록과 현재 틱을 고려하도록 변환합니다.

더 오래된 시점이 요청되면, 이진 검색 알고리즘으로 전환하기 전에 몇 가지 검사를 수행해야 합니다.
1. 요청된 시점이 마지막 관측인 경우, 최신 관측에서 누적 가격을 반환할 수 있습니다.
2. 요청된 시점이 마지막 관측 이후인 경우, `transform`을 호출하여 이 시점의 누적 가격을 찾을 수 있습니다. 마지막으로 관측된 가격과 현재 가격을 알고 있습니다.
3. 요청된 시점이 마지막 관측 이전인 경우, 이진 검색을 사용해야 합니다.

세 번째 요점으로 바로 넘어가겠습니다.
```solidity
function binarySearch(
    Observation[65535] storage self,
    uint32 time,
    uint32 target,
    uint16 index,
    uint16 cardinality
)
    private
    view
    returns (Observation memory beforeOrAt, Observation memory atOrAfter)
{
    ...
```

이 함수는 현재 블록 타임스탬프 (`time`), 요청된 가격 시점의 타임스탬프 (`target`), 그리고 현재 관측 인덱스와 카디널리티를 사용합니다. 요청된 시점이 위치한 두 관측 사이의 범위를 반환합니다.

이진 검색 알고리즘을 초기화하기 위해, 경계를 설정합니다.
```solidity
uint256 l = (index + 1) % cardinality; // 가장 오래된 관측
uint256 r = l + cardinality - 1; // 가장 최신 관측
uint256 i;
```

관측 배열은 오버플로가 예상되므로, 여기서 모듈로 연산자를 사용하고 있습니다.

그런 다음 무한 루프를 시작합니다. 루프에서 범위의 중간 지점을 확인합니다. 중간 지점이 초기화되지 않은 경우 (관측이 없는 경우), 다음 지점으로 계속 진행합니다.

```solidity
while (true) {
    i = (l + r) / 2;

    beforeOrAt = self[i % cardinality];

    if (!beforeOrAt.initialized) {
        l = i + 1;
        continue;
    }

    ...
```

지점이 초기화된 경우, 요청된 시점이 포함될 범위의 왼쪽 경계라고 부릅니다. 그리고 오른쪽 경계 (`atOrAfter`)를 찾으려고 시도합니다.

```solidity
    ...
    atOrAfter = self[(i + 1) % cardinality];

    bool targetAtOrAfter = lte(time, beforeOrAt.timestamp, target);

    if (targetAtOrAfter && lte(time, target, atOrAfter.timestamp))
        break;
    ...
```
경계를 찾았으면 반환합니다. 그렇지 않으면 검색을 계속합니다.

```solidity
    ...
    if (!targetAtOrAfter) r = i - 1;
    else l = i + 1;
}
```

요청된 시점이 속하는 관측 범위를 찾은 후, 요청된 시점의 가격을 계산해야 합니다.
```solidity
// function observeSingle() {
    ...
    uint56 observationTimeDelta = atOrAfter.timestamp -
        beforeOrAt.timestamp;
    uint56 targetDelta = target - beforeOrAt.timestamp;
    return
        beforeOrAt.tickCumulative +
        ((atOrAfter.tickCumulative - beforeOrAt.tickCumulative) /
            int56(observationTimeDelta)) *
        int56(targetDelta);
    ...
```

이것은 범위 내 평균 변화율을 찾고, 범위의 하한과 필요한 시점 사이에 경과한 초 수를 곱하는 것만큼 간단합니다. 이것이 앞에서 논의한 보간입니다.

여기서 마지막으로 구현해야 할 것은 풀 컨트랙트에서 관측을 읽고 반환하는 공개 함수입니다.

```solidity
// src/UniswapV3Pool.sol
function observe(uint32[] calldata secondsAgos)
    public
    view
    returns (int56[] memory tickCumulatives)
{
    return
        observations.observe(
            _blockTimestamp(),
            secondsAgos,
            slot0.tick,
            slot0.observationIndex,
            slot0.observationCardinality
        );
}
```

### 관측 해석

이제 관측을 해석하는 방법을 살펴보겠습니다.

방금 추가한 `observe` 함수는 누적 가격 배열을 반환하며, 이를 실제 가격으로 변환하는 방법을 알고 싶습니다. `observe` 함수 테스트에서 이를 시연하겠습니다.

테스트에서 여러 방향과 다른 블록에서 여러 스왑을 실행했습니다.

```solidity
function testObserve() public {
    ...
    pool.increaseObservationCardinalityNext(3);

    vm.warp(2);
    pool.swap(address(this), false, swapAmount, sqrtP(6000), extra);

    vm.warp(7);
    pool.swap(address(this), true, swapAmount2, sqrtP(4000), extra);

    vm.warp(20);
    pool.swap(address(this), false, swapAmount, sqrtP(6000), extra);
    ...
```

> `vm.warp`는 Foundry에서 제공하는 치트 코드입니다. 지정된 타임스탬프가 있는 블록으로 포워딩합니다. 2, 7, 20은 블록 타임스탬프입니다.

첫 번째 스왑은 타임스탬프 2의 블록에서 이루어지고, 두 번째 스왑은 타임스탬프 7에서 이루어지고, 세 번째 스왑은 타임스탬프 20에서 이루어집니다. 그런 다음 관측을 읽을 수 있습니다.

```solidity
    ...
    secondsAgos = new uint32[](4);
    secondsAgos[0] = 0;
    secondsAgos[1] = 13;
    secondsAgos[2] = 17;
    secondsAgos[3] = 18;

    int56[] memory tickCumulatives = pool.observe(secondsAgos);
    assertEq(tickCumulatives[0], 1607059);
    assertEq(tickCumulatives[1], 511146);
    assertEq(tickCumulatives[2], 170370);
    assertEq(tickCumulatives[3], 85176);
    ...
```

1. 가장 먼저 관측된 가격은 0입니다. 풀이 배포될 때 설정되는 초기 관측입니다. 하지만 카디널리티가 3으로 설정되고 3번의 스왑을 수행했기 때문에 마지막 관측으로 덮어쓰여졌습니다.
2. 첫 번째 스왑 중에 틱 85176이 관측되었습니다. 이는 풀의 초기 가격입니다. 스왑 전 가격이 관측된다는 것을 기억하십시오. 가장 첫 번째 관측이 덮어쓰여졌기 때문에, 이것이 현재 가장 오래된 관측입니다.
3. 다음으로 반환된 누적 가격은 170370입니다. 이는 `85176 + 85194`입니다. 전자는 이전 누적기 값이고, 후자는 두 번째 스왑 중에 관측된 첫 번째 스왑 후의 가격입니다.
4. 다음으로 반환된 누적 가격은 511146입니다. 이는 `(511146 - 170370) / (17 - 13) = 85194`입니다. 두 번째 스왑과 세 번째 스왑 사이의 누적 가격입니다.
5. 마지막으로 가장 최근 관측은 1607059입니다. 이는 `(1607059 - 511146) / (20 - 7) = 84301`입니다. 이는 ~4581 USDC/ETH, 즉 세 번째 스왑 중에 관측된 두 번째 스왑 후의 가격입니다.

다음은 보간이 포함된 예입니다. 요청된 시점은 스왑 시점이 아닙니다.

```solidity
secondsAgos = new uint32[](5);
secondsAgos[0] = 0;
secondsAgos[1] = 5;
secondsAgos[2] = 10;
secondsAgos[3] = 15;
secondsAgos[4] = 18;

tickCumulatives = pool.observe(secondsAgos);
assertEq(tickCumulatives[0], 1607059);
assertEq(tickCumulatives[1], 1185554);
assertEq(tickCumulatives[2], 764049);
assertEq(tickCumulatives[3], 340758);
assertEq(tickCumulatives[4], 85176);
```

이는 4581.03, 4581.03, 4747.6, 5008.91의 가격으로 이어집니다. 요청된 간격 내 평균 가격입니다.

> 다음은 Python에서 해당 값을 계산하는 방법입니다.
> ```python
> vals = [1607059, 1185554, 764049, 340758, 85176]
> secs = [0, 5, 10, 15, 18]
> [1.0001**((vals[i] - vals[i+1]) / (secs[i+1] - secs[i])) for i in range(len(vals)-1)]
> ```