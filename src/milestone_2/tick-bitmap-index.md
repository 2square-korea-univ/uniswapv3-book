## 틱 비트맵 인덱스

동적 스왑을 향한 첫 번째 단계로서, 틱 인덱스를 구현해야 합니다. 이전 마일스톤에서는 스왑을 수행할 때 목표 틱을 계산하는 방식을 사용했습니다:

```solidity
function swap(address recipient, bytes calldata data)
    public
    returns (int256 amount0, int256 amount1)
{
  int24 nextTick = 85184;
  ...
}
```

다양한 가격 범위에 유동성이 제공될 때, 단순히 목표 틱을 계산할 수 없습니다. 우리는 그것을 **찾아야** 합니다. 따라서 유동성을 가진 모든 틱을 인덱싱하고, 이 인덱스를 사용하여 스왑에 충분한 유동성을 "주입"할 틱을 찾아야 합니다. 이번 단계에서는 이러한 인덱스를 구현할 것입니다.

## 비트맵

비트맵은 데이터를 압축적인 방식으로 인덱싱하는 데 널리 사용되는 기술입니다. 비트맵은 단순히 이진 시스템으로 표현된 숫자입니다. 예를 들어, 31337은 `111101001101001`입니다. 우리는 이것을 0과 1의 배열로 볼 수 있으며, 각 숫자는 인덱스를 가집니다. 여기서 0은 플래그가 설정되지 않았음을 의미하고 1은 설정되었음을 의미합니다. 따라서 매우 압축적인 인덱싱된 플래그 배열을 얻게 됩니다. 각 바이트는 8개의 플래그를 담을 수 있습니다. Solidity에서 우리는 최대 256비트의 정수를 가질 수 있으며, 이는 하나의 `uint256`이 256개의 플래그를 담을 수 있음을 의미합니다.

Uniswap V3는 초기화된 틱, 즉 유동성이 있는 틱에 대한 정보를 저장하기 위해 이 기술을 사용합니다. 플래그가 설정되면(1), 틱은 유동성을 가집니다. 플래그가 설정되지 않으면(0), 틱은 초기화되지 않은 것입니다. 구현을 살펴봅시다.

## TickBitmap 컨트랙트

풀 컨트랙트에서 틱 인덱스는 상태 변수에 저장됩니다:

```solidity
contract UniswapV3Pool {
    using TickBitmap for mapping(int16 => uint256);
    mapping(int16 => uint256) public tickBitmap;
    ...
}
```

이것은 키가 `int16`이고 값이 워드(`uint256`)인 매핑입니다. 무한한 연속적인 1과 0의 배열을 상상해보세요:



![틱 비트맵의 틱 인덱스](images/tick_bitmap.png)

이 배열의 각 요소는 틱에 해당합니다. 이 배열에서 틱의 위치를 찾기 위해, 우리는 그것을 256비트 길이의 서브 배열인 워드로 나눕니다. 이 배열에서 틱의 위치를 찾으려면 다음을 수행합니다:

```solidity
function position(int24 tick) private pure returns (int16 wordPos, uint8 bitPos) {
    wordPos = int16(tick >> 8);
    bitPos = uint8(uint24(tick % 256));
}
```

즉, 워드 위치를 찾은 다음 이 워드에서 비트 위치를 찾습니다. `>> 8`은 256으로 정수 나누기를 하는 것과 동일합니다. 따라서 워드 위치는 틱 인덱스를 256으로 나눈 정수 부분이고, 비트 위치는 나머지입니다.

예를 들어, 틱 중 하나인 틱 85176에 대한 워드 및 비트 위치를 계산해 봅시다:

```python
tick = 85176
word_pos = tick >> 8 # 또는 tick // 2**8
bit_pos = tick % 256
print(f"Word {word_pos}, bit {bit_pos}")
# Word 332, bit 184
```

### 플래그 뒤집기

풀에 유동성을 추가할 때, 비트맵에서 몇 개의 틱 플래그를 설정해야 합니다: 하나는 하위 틱에 대한 것이고 다른 하나는 상위 틱에 대한 것입니다. 우리는 비트맵 매핑의 `flipTick` 메소드에서 이를 수행합니다:

```solidity
function flipTick(
    mapping(int16 => uint256) storage self,
    int24 tick,
    int24 tickSpacing
) internal {
    require(tick % tickSpacing == 0); // 틱이 간격을 유지하는지 확인
    (int16 wordPos, uint8 bitPos) = position(tick / tickSpacing);
    uint256 mask = 1 << bitPos;
    self[wordPos] ^= mask;
}
```

> 이 책의 뒷부분까지 `tickSpacing`은 항상 1입니다. 이 값은 초기화할 수 있는 틱에 영향을 미친다는 점을 명심하세요. 값이 1일 때는 모든 틱을 뒤집을 수 있지만, 다른 값으로 설정되면 해당 값으로 나누어 떨어지는 틱만 뒤집을 수 있습니다.

워드 및 비트 위치를 찾은 후, 마스크를 만들어야 합니다. 마스크는 틱의 비트 위치에 단일 1 플래그가 설정된 숫자입니다. 마스크를 찾기 위해, 단순히 `2**bit_pos` (또는 `1 << bit_pos`와 동일)를 계산합니다:

```python
mask = 2**bit_pos # 또는 1 << bit_pos
print(format(mask, '#0258b'))                                             ↓ 여기
#0b00000000000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

다음으로, 플래그를 뒤집기 위해 비트 XOR을 통해 마스크를 틱의 워드에 적용합니다:

```python
word = (2**256) - 1 # 워드를 모두 1로 설정
print(format(word ^ mask, '#0258b'))                                      ↓ 여기
#0b1111111111111111111111111111111111111111111111111111111111111111111111101111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111
```

184번째 비트(오른쪽에서 0부터 시작하여 계산)가 0으로 뒤집힌 것을 볼 수 있습니다.

비트가 0이면 1로 설정합니다:

```python
word = 0
print(format(word ^ mask, '#0258b'))                                      ↓ 여기
#0b00000000000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

### 다음 틱 찾기

다음 단계는 비트맵 인덱스를 사용하여 유동성을 가진 틱을 찾는 것입니다.

스왑 중에, 우리는 현재 틱 이전 또는 이후(즉, 왼쪽 또는 오른쪽)에 유동성이 있는 틱을 찾아야 합니다. 이전 마일스톤에서는 [계산하고 하드 코딩하여 사용했지만](https://github.com/Jeiwan/uniswapv3-code/blob/85b8605c37a9065c141a234ee2c18d9507eeba22/src/UniswapV3Pool.sol#L142), 이제 비트맵 인덱스를 사용하여 그러한 틱을 찾아야 합니다. 우리는 `TickBitmap.nextInitializedTickWithinOneWord` 함수에서 이를 수행할 것입니다. 이 함수에서는 두 가지 시나리오를 구현해야 합니다:

1. 토큰 $x$(여기서는 ETH)를 판매할 때, 현재 틱의 워드에서 현재 틱의 **오른쪽**에 있는 다음 초기화된 틱을 찾습니다.
2. 토큰 $y$(여기서는 USDC)를 판매할 때, 다음 (현재 + 1) 틱의 워드에서 현재 틱의 **왼쪽**에 있는 다음 초기화된 틱을 찾습니다.

이는 어느 방향으로든 스왑을 할 때의 가격 변동에 해당합니다:



![스왑 중 다음 초기화된 틱 찾기](images/find_next_tick.png)

> 코드에서는 방향이 뒤집힌다는 점에 유의하세요. 토큰 $x$를 구매할 때, 현재 틱의 **왼쪽**에 있는 초기화된 틱을 검색합니다. 토큰 $x$를 판매할 때는 **오른쪽**에 있는 틱을 검색합니다. 그러나 이것은 워드 내에서만 해당됩니다. 워드는 왼쪽에서 오른쪽으로 정렬됩니다.

현재 워드에 초기화된 틱이 없으면, 다음 루프 사이클에서 인접한 워드에서 검색을 계속합니다.

이제 구현을 살펴봅시다:

```solidity
function nextInitializedTickWithinOneWord(
    mapping(int16 => uint256) storage self,
    int24 tick,
    int24 tickSpacing,
    bool lte
) internal view returns (int24 next, bool initialized) {
    int24 compressed = tick / tickSpacing;
    ...
```

1. 첫 번째 인수는 이 함수를 `mapping(int16 => uint256)`의 메소드로 만듭니다.
2. `tick`은 현재 틱입니다.
3. `tickSpacing`은 마일스톤 4에서 사용하기 시작할 때까지 항상 1입니다.
4. `lte`는 방향을 설정하는 플래그입니다. `true`이면 토큰 $x$를 판매하고 현재 틱의 오른쪽에 있는 다음 초기화된 틱을 검색합니다. `false`이면 반대 방향입니다. `lte`는 스왑 방향과 같습니다: 토큰 $x$를 판매할 때 `true`, 그렇지 않으면 `false`입니다.

```solidity
if (lte) {
    (int16 wordPos, uint8 bitPos) = position(compressed);
    uint256 mask = (1 << bitPos) - 1 + (1 << bitPos);
    uint256 masked = self[wordPos] & mask;
    ...
```

$x$를 판매할 때, 우리는:
1. 현재 틱의 워드 및 비트 위치를 가져옵니다.
2. 현재 비트 위치를 포함하여 오른쪽의 모든 비트가 1인 마스크를 만듭니다(`mask`는 모두 1이고, 길이는 `bitPos`입니다).
3. 마스크를 현재 틱의 워드에 적용합니다.

```solidity
    ...
    initialized = masked != 0;
    next = initialized
        ? (compressed - int24(uint24(bitPos - BitMath.mostSignificantBit(masked)))) * tickSpacing
        : (compressed - int24(uint24(bitPos))) * tickSpacing;
    ...
```

다음으로, `masked`는 적어도 하나의 비트가 1로 설정되어 있으면 0이 되지 않습니다. 그렇다면 초기화된 틱이 있습니다. 그렇지 않으면 없습니다(현재 워드에 없습니다). 결과에 따라 다음 초기화된 틱의 인덱스 또는 다음 워드의 가장 왼쪽 비트를 반환합니다. 이렇게 하면 다른 루프 사이클 중에 워드에서 초기화된 틱을 검색할 수 있습니다.

```solidity
    ...
} else {
    (int16 wordPos, uint8 bitPos) = position(compressed + 1);
    uint256 mask = ~((1 << bitPos) - 1);
    uint256 masked = self[wordPos] & mask;
    ...
```

마찬가지로, $y$를 판매할 때, 우리는:
1. 현재 틱의 워드 및 비트 위치를 가져옵니다.
2. 현재 틱 비트 위치의 왼쪽에 있는 모든 비트는 1이고 오른쪽에 있는 모든 비트는 0인 다른 마스크를 만듭니다.
3. 마스크를 현재 틱의 워드에 적용합니다.

다시 말하지만, 왼쪽에 초기화된 틱이 없으면 이전 워드의 가장 오른쪽 비트가 반환됩니다:

```solidity
    ...
    initialized = masked != 0;
    // 오버플로우/언더플로우가 발생할 수 있지만, tickSpacing과 tick 모두 제한되어 외부적으로 방지됨
    next = initialized
        ? (compressed + 1 + int24(uint24((BitMath.leastSignificantBit(masked) - bitPos)))) * tickSpacing
        : (compressed + 1 + int24(uint24((type(uint8).max - bitPos)))) * tickSpacing;
}
```

이것으로 끝입니다!

보시다시피, `nextInitializedTickWithinOneWord`는 멀리 떨어진 경우 정확한 틱을 찾지 못합니다. 검색 범위는 현재 또는 다음 틱의 워드입니다. 실제로, 우리는 무한한 비트맵 인덱스를 반복하고 싶지 않습니다.