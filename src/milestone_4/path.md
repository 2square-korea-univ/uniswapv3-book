# 스왑 경로

WETH/USDC, USDC/USDT, WBTC/USDT 풀만 있다고 가정해 봅시다. WETH를 WBTC로 스왑하고 싶지만 WETH/WBTC 풀이 없다면, 여러 번의 스왑(WETH→USDC→USDT→WBTC)을 거쳐야 합니다. 이러한 스왑을 수동으로 처리할 수도 있지만, 컨트랙트를 개선하여 체인 스왑, 즉 다중 풀 스왑을 처리하도록 할 수 있습니다. 물론 후자를 선택할 것입니다!

다중 풀 스왑을 수행할 때는 이전 스왑의 결과물을 다음 스왑의 입력값으로 전달합니다. 예를 들면 다음과 같습니다.

1. WETH/USDC 풀에서 WETH를 판매하고 USDC를 구매합니다.
2. USDC/USDT 풀에서 이전 스왑에서 얻은 USDC를 판매하고 USDT를 구매합니다.
3. WBTC/USDT 풀에서 이전 풀에서 얻은 USDT를 판매하고 WBTC를 구매합니다.

이러한 일련의 과정을 경로로 나타낼 수 있습니다.

```
WETH/USDC,USDC/USDT,WBTC/USDT
```

그리고 컨트랙트에서 이 경로를 반복하여 단일 트랜잭션으로 여러 스왑을 수행할 수 있습니다. 하지만 이전 장에서 풀 주소를 알 필요 없이 풀 파라미터로부터 풀 주소를 유도할 수 있다는 것을 기억하십시오. 따라서 위의 경로는 다음과 같은 토큰 시리즈로 바뀔 수 있습니다.

```
WETH, USDC, USDT, WBTC
```

틱 간격은 토큰 외에 풀을 식별하는 또 다른 파라미터입니다. 따라서 위의 경로는 다음과 같이 됩니다.

```
WETH, 60, USDC, 10, USDT, 60, WBTC
```

여기서 60과 10은 틱 간격입니다. 변동성이 큰 페어(예: ETH/USDC, WBTC/USDT)에는 60을 사용하고, 스테이블 코인 페어(USDC/USDT)에는 10을 사용합니다.

이제 이러한 경로를 통해 각 풀에 대한 풀 파라미터를 구성하기 위해 반복할 수 있습니다.

1. `WETH, 60, USDC`
2. `USDC, 10, USDT`
3. `USDT, 60, WBTC`

이러한 파라미터를 알면 이전 장에서 구현한 `PoolAddress.computeAddress`를 사용하여 풀 주소를 유도할 수 있습니다.

> 또한 단일 풀 내에서 스왑을 수행할 때도 이 개념을 사용할 수 있습니다. 경로는 단순히 단일 풀의 파라미터를 포함하게 됩니다. 따라서 스왑 경로를 모든 스왑에서 보편적으로 사용할 수 있습니다.

스왑 경로 작업을 위한 라이브러리를 구축해 보겠습니다.

## 경로 라이브러리

코드에서 스왑 경로는 바이트 시퀀스입니다. Solidity에서 경로는 다음과 같이 구축할 수 있습니다.
```solidity
bytes.concat(
    bytes20(address(weth)),
    bytes3(uint24(60)),
    bytes20(address(usdc)),
    bytes3(uint24(10)),
    bytes20(address(usdt)),
    bytes3(uint24(60)),
    bytes20(address(wbtc))
);
```

다음과 같은 형태입니다.
```shell
0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 # weth 주소
  00003c                                   # 60
  A0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 # usdc 주소
  00000a                                   # 10
  dAC17F958D2ee523a2206206994597C13D831ec7 # usdt 주소
  00003c                                   # 60
  2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599 # wbtc 주소
```

구현해야 할 함수는 다음과 같습니다.
1. 경로에 있는 풀의 수 계산
2. 경로에 여러 풀이 있는지 확인
3. 경로에서 첫 번째 풀 파라미터 추출
4. 경로에서 다음 페어로 진행
5. 첫 번째 풀 파라미터 디코딩

### 경로에 있는 풀의 수 계산
경로에 있는 풀의 수를 계산하는 것부터 시작해 보겠습니다.
```solidity
// src/lib/Path.sol
library Path {
    /// @dev 바이트로 인코딩된 주소의 길이
    uint256 private constant ADDR_SIZE = 20;
    /// @dev 바이트로 인코딩된 틱 간격의 길이
    uint256 private constant TICKSPACING_SIZE = 3;

    /// @dev 단일 토큰 주소 + 틱 간격의 오프셋
    uint256 private constant NEXT_OFFSET = ADDR_SIZE + TICKSPACING_SIZE;
    /// @dev 인코딩된 풀 키의 오프셋 (tokenIn + 틱 간격 + tokenOut)
    uint256 private constant POP_OFFSET = NEXT_OFFSET + ADDR_SIZE;
    /// @dev 2개 이상의 풀을 포함하는 경로의 최소 길이
    uint256 private constant MULTIPLE_POOLS_MIN_LENGTH =
        POP_OFFSET + NEXT_OFFSET;

    ...
```

먼저 몇 가지 상수를 정의합니다.
1. `ADDR_SIZE`는 주소의 크기로, 20바이트입니다.
2. `TICKSPACING_SIZE`는 틱 간격의 크기로, 3바이트(`uint24`)입니다.
3. `NEXT_OFFSET`는 다음 토큰 주소의 오프셋입니다. 이를 얻으려면 주소와 틱 간격을 건너뜁니다.
4. `POP_OFFSET`는 풀 키(토큰 주소 + 틱 간격 + 토큰 주소)의 오프셋입니다.
5. `MULTIPLE_POOLS_MIN_LENGTH`는 2개 이상의 풀(하나의 풀 파라미터 세트 + 틱 간격 + 토큰 주소)을 포함하는 경로의 최소 길이입니다.

경로에 있는 풀의 수를 세려면 경로의 길이에서 주소의 크기(경로의 첫 번째 또는 마지막 토큰)를 빼고 남은 부분을 `NEXT_OFFSET`(주소 + 틱 간격)으로 나눕니다.

```solidity
function numPools(bytes memory path) internal pure returns (uint256) {
    return (path.length - ADDR_SIZE) / NEXT_OFFSET;
}
```

### 경로에 여러 풀이 있는지 확인
경로에 여러 풀이 있는지 확인하려면 경로의 길이를 `MULTIPLE_POOLS_MIN_LENGTH`와 비교해야 합니다.

```solidity
function hasMultiplePools(bytes memory path) internal pure returns (bool) {
    return path.length >= MULTIPLE_POOLS_MIN_LENGTH;
}
```

### 경로에서 첫 번째 풀 파라미터 추출

다른 함수를 구현하려면 Solidity에 기본 바이트 조작 함수가 없으므로 헬퍼 라이브러리가 필요합니다. 특히 바이트 배열에서 하위 배열을 추출하는 함수, 바이트를 `address` 및 `uint24`로 변환하는 몇 가지 함수가 필요합니다.

다행히 [solidity-bytes-utils](https://github.com/GNSPS/solidity-bytes-utils)라는 훌륭한 오픈 소스 라이브러리가 있습니다. 라이브러리를 사용하려면 `Path` 라이브러리에서 `bytes` 유형을 확장해야 합니다.
```solidity
library Path {
    using BytesLib for bytes;
    ...
}
```

이제 `getFirstPool`을 구현할 수 있습니다.
```solidity
function getFirstPool(bytes memory path)
    internal
    pure
    returns (bytes memory)
{
    return path.slice(0, POP_OFFSET);
}
```

이 함수는 바이트로 인코딩된 첫 번째 "토큰 주소 + 틱 간격 + 토큰 주소" 세그먼트를 반환합니다.

### 경로에서 다음 페어로 진행

다음 함수는 경로를 반복하고 처리된 풀을 버릴 때 사용합니다. 다음 풀 주소를 계산하려면 다른 토큰 주소가 필요하므로 전체 풀 파라미터가 아닌 "토큰 주소 + 틱 간격"만 제거합니다.

```solidity
function skipToken(bytes memory path) internal pure returns (bytes memory) {
    return path.slice(NEXT_OFFSET, path.length - NEXT_OFFSET);
}
```

### 첫 번째 풀 파라미터 디코딩

마지막으로 경로에서 첫 번째 풀의 파라미터를 디코딩해야 합니다.

```solidity
function decodeFirstPool(bytes memory path)
    internal
    pure
    returns (
        address tokenIn,
        address tokenOut,
        uint24 tickSpacing
    )
{
    tokenIn = path.toAddress(0);
    tickSpacing = path.toUint24(ADDR_SIZE);
    tokenOut = path.toAddress(NEXT_OFFSET);
}
```

안타깝게도 `BytesLib`은 `toUint24` 함수를 구현하지 않지만 직접 구현할 수 있습니다! `BytesLib`에는 여러 개의 `toUintXX` 함수가 있으므로 그 중 하나를 가져와서 `uint24` 함수로 변환할 수 있습니다.
```solidity
library BytesLibExt {
    function toUint24(bytes memory _bytes, uint256 _start)
        internal
        pure
        returns (uint24)
    {
        require(_bytes.length >= _start + 3, "toUint24_outOfBounds");
        uint24 tempUint;

        assembly {
            tempUint := mload(add(add(_bytes, 0x3), _start))
        }

        return tempUint;
    }
}
```

새로운 라이브러리 컨트랙트에서 이를 수행하고, 그런 다음 `BytesLib`과 함께 Path 라이브러리에서 사용할 수 있습니다.

```solidity
library Path {
    using BytesLib for bytes;
    using BytesLibExt for bytes;
    ...
}
```