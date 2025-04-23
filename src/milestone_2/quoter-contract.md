# Quoter 컨트랙트

업데이트된 Pool 컨트랙트를 프론트엔드 앱에 통합하기 위해서는 스왑을 실행하지 않고 스왑 양을 계산하는 방법이 필요합니다. 사용자가 판매하고자 하는 양을 입력하면, 교환으로 받게 될 양을 계산하여 사용자에게 보여주고자 합니다. 이를 위해 Quoter 컨트랙트를 사용할 것입니다.

Uniswap V3의 유동성은 여러 가격 범위에 분산되어 있기 때문에 (Uniswap V2에서 가능했던 것처럼) 공식을 사용하여 스왑 양을 계산할 수 없습니다. Uniswap V3의 설계는 다른 접근 방식을 강제합니다. 스왑 양을 계산하기 위해 실제 스왑을 시작하고 콜백 함수에서 이를 중단하여 Pool 컨트랙트에 의해 계산된 양을 확보할 것입니다. 즉, 출력량을 계산하기 위해 실제 스왑을 시뮬레이션해야 합니다!

다시 말하지만, 이를 위한 헬퍼 컨트랙트를 만들 것입니다.

```solidity
contract UniswapV3Quoter {
    struct QuoteParams {
        address pool;
        uint256 amountIn;
        bool zeroForOne;
    }

    function quote(QuoteParams memory params)
        public
        returns (
            uint256 amountOut,
            uint160 sqrtPriceX96After,
            int24 tickAfter
        )
    {
        ...
```

Quoter는 오직 하나의 public 함수인 `quote`만을 구현하는 컨트랙트입니다. Quoter는 모든 풀에서 작동하는 범용 컨트랙트이므로 풀 주소를 매개변수로 받습니다. 다른 매개변수(`amountIn` 및 `zeroForOne`)는 스왑을 시뮬레이션하는 데 필요합니다.

```solidity
try
    IUniswapV3Pool(params.pool).swap(
        address(this),
        params.zeroForOne,
        params.amountIn,
        abi.encode(params.pool)
    )
{} catch (bytes memory reason) {
    return abi.decode(reason, (uint256, uint160, int24));
}
```

컨트랙트가 하는 유일한 작업은 풀의 `swap` 함수를 호출하는 것입니다. 이 호출은 되돌려질(즉, 에러를 발생시킬) 것으로 예상됩니다. 스왑 콜백에서 이를 수행할 것입니다. 되돌림의 경우, 되돌림 사유가 디코딩되어 반환됩니다. `quote`는 절대 되돌려지지 않습니다. 추가 데이터에서 풀 주소만 전달하는 것을 주목하세요. 스왑 콜백에서 스왑 후 풀의 `slot0`를 얻기 위해 이를 사용할 것입니다.

```solidity
function uniswapV3SwapCallback(
    int256 amount0Delta,
    int256 amount1Delta,
    bytes memory data
) external view {
    address pool = abi.decode(data, (address));

    uint256 amountOut = amount0Delta > 0
        ? uint256(-amount1Delta)
        : uint256(-amount0Delta);

    (uint160 sqrtPriceX96After, int24 tickAfter) = IUniswapV3Pool(pool)
        .slot0();
```

스왑 콜백에서 필요한 값인 출력량, 새로운 가격, 그리고 해당 틱을 수집하고 있습니다. 다음으로, 이 값들을 저장하고 되돌려야 합니다.

```solidity
assembly {
    let ptr := mload(0x40)
    mstore(ptr, amountOut)
    mstore(add(ptr, 0x20), sqrtPriceX96After)
    mstore(add(ptr, 0x40), tickAfter)
    revert(ptr, 96)
}
```

가스 최적화를 위해 이 부분은 Solidity의 인라인 어셈블리 언어인 [Yul](https://docs.soliditylang.org/en/latest/assembly.html)로 구현됩니다. 자세히 살펴보겠습니다.
1. `mload(0x40)`은 다음 사용 가능한 메모리 슬롯의 포인터를 읽습니다 (EVM의 메모리는 32바이트 슬롯으로 구성됨).
2. 해당 메모리 슬롯에 `mstore(ptr, amountOut)`은 `amountOut`을 씁니다.
3. `mstore(add(ptr, 0x20), sqrtPriceX96After)`은 `amountOut` 바로 뒤에 `sqrtPriceX96After`를 씁니다.
4. `mstore(add(ptr, 0x40), tickAfter)`은 `sqrtPriceX96After` 뒤에 `tickAfter`를 씁니다.
5. `revert(ptr, 96)`은 호출을 되돌리고 주소 `ptr` (위에서 쓴 데이터의 시작점)에서 96바이트 (우리가 메모리에 쓴 값들의 총 길이)의 데이터를 반환합니다.

따라서, 필요한 값들의 바이트 표현을 연결하고 있습니다 (정확히 `abi.encode()`가 하는 것). 오프셋은 항상 32바이트인 것을 주목하세요. `sqrtPriceX96After`는 20바이트(`uint160`)를 차지하고 `tickAfter`는 3바이트(`int24`)를 차지하더라도 그렇습니다. 이는 `abi.decode()`를 사용하여 데이터를 디코딩할 수 있도록 하기 위함입니다. 그 대응 함수인 `abi.encode()`는 모든 정수를 32바이트 워드로 인코딩합니다.

자, 이제... ~끝났습니다~! 완료되었습니다.

## 요약

알고리즘을 더 잘 이해하기 위해 요약해 봅시다.
1. `quote`는 입력량과 스왑 방향으로 풀의 `swap`을 호출합니다.
2. `swap`은 실제 스왑을 수행하며, 사용자가 지정한 입력량을 채우기 위해 루프를 실행합니다.
3. 사용자로부터 토큰을 얻기 위해 `swap`은 호출자에게 스왑 콜백을 호출합니다.
4. 호출자 (Quoter 컨트랙트)는 콜백을 구현하며, 콜백에서 출력량, 새로운 가격, 그리고 새로운 틱으로 되돌립니다.
5. 되돌림은 초기 `quote` 호출까지 버블업됩니다.
6. `quote`에서 되돌림이 포착되고, 되돌림 사유가 디코딩되어 `quote` 호출의 결과로 반환됩니다.

이해가 되셨기를 바랍니다!

## Quoter의 한계

이 설계에는 한 가지 중요한 한계가 있습니다. `quote`가 Pool 컨트랙트의 `swap` 함수를 호출하고, `swap` 함수는 (컨트랙트 상태를 수정하기 때문에) 순수 함수 또는 view 함수가 아니므로, `quote` 또한 순수 함수 또는 view 함수가 될 수 없습니다. `swap`은 상태를 수정하고 `quote`도 마찬가지입니다. 비록 Quoter 컨트랙트에서 직접 수정하는 것은 아니지만 말입니다. 그러나 우리는 `quote`를 컨트랙트 데이터를 읽기만 하는 getter 함수로 취급합니다. 이러한 불일치는 `quote`가 호출될 때 EVM이 [STATICCALL](https://www.evm.codes/#fa) opcode 대신 [CALL](https://www.evm.codes/#f1) opcode를 사용하게 된다는 것을 의미합니다. 이는 큰 문제는 아닙니다. Quoter가 스왑 콜백에서 되돌아가고, 되돌림이 호출 중에 수정된 상태를 리셋하기 때문입니다. 이는 `quote`가 Pool 컨트랙트의 상태를 수정하지 않음을 (실제 거래가 발생하지 않음을) 보장합니다.

이 문제에서 비롯되는 또 다른 불편함은 클라이언트 라이브러리 (Ethers.js, Web3.js 등)에서 `quote`를 호출하면 트랜잭션이 트리거된다는 것입니다. 이를 해결하기 위해 라이브러리가 static call을 강제하도록 해야 합니다. 이 마일스톤의 뒷부분에서 Ethers.js에서 이를 수행하는 방법을 알아볼 것입니다.