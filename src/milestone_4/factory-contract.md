# 팩토리 컨트랙트

유니스왑은 각각의 풀 컨트랙트가 하나의 토큰 쌍의 스왑을 처리하는, 다수의 분리된 풀 컨트랙트가 있다고 가정하는 방식으로 설계되었습니다. 이는 풀이 없는 두 토큰 간에 스왑하려는 경우 문제가 될 수 있습니다. 풀이 없다면 스왑이 불가능하기 때문입니다. 그러나 여전히 중간 스왑을 할 수 있습니다. 첫째, 두 토큰 중 하나와 쌍을 이루는 토큰으로 스왑한 다음, 이 토큰을 대상 토큰으로 스왑합니다. 이는 더 깊어지고 더 많은 중간 토큰을 가질 수도 있습니다. 그러나 수동으로 이 작업을 수행하는 것은 번거롭고, 다행히 스마트 컨트랙트에 구현하여 프로세스를 더 쉽게 만들 수 있습니다.

팩토리 컨트랙트는 여러 목적을 수행하는 컨트랙트입니다.
1. 풀 컨트랙트의 중앙 집중식 레지스트리 역할을 합니다. 팩토리를 사용하면 배포된 모든 풀, 해당 토큰 및 주소를 찾을 수 있습니다.
2. 풀 컨트랙트의 배포를 단순화합니다. EVM은 스마트 컨트랙트에서 스마트 컨트랙트 배포를 허용합니다. 팩토리는 이 기능을 사용하여 풀 배포를 간편하게 만듭니다.
3. 풀 주소를 예측 가능하게 만들고 레지스트리에 호출하지 않고도 계산할 수 있도록 합니다. 이를 통해 풀을 쉽게 찾을 수 있습니다.

팩토리 컨트랙트를 구축해 보겠습니다! 하지만 그 전에 새로운 것을 배워야 합니다.

## `CREATE` 및 `CREATE2` Opcode

EVM에는 `CREATE` 또는 `CREATE2` opcode를 통해 컨트랙트를 배포하는 두 가지 방법이 있습니다. 이 둘의 유일한 차이점은 새 컨트랙트 주소가 생성되는 방식입니다.
1. `CREATE`는 배포자의 계정 `nonce`를 사용하여 컨트랙트 주소를 생성합니다 (의사 코드).
    ```
    KECCAK256(deployer.address, deployer.nonce)
    ```
    `nonce`는 계정별 트랜잭션 카운터입니다. 새 컨트랙트 주소 생성에 `nonce`를 사용하면 다른 컨트랙트 또는 오프체인 앱에서 주소를 계산하기가 어려워집니다. 주로 컨트랙트가 배포된 nonce를 찾으려면 과거 계정 트랜잭션을 스캔해야 하기 때문입니다.
2. `CREATE2`는 사용자 정의 *솔트*를 사용하여 컨트랙트 주소를 생성합니다. 이것은 개발자가 선택한 임의의 바이트 시퀀스이며, 주소 생성을 결정적으로 만들고 (충돌 가능성을 줄입니다) 데 사용됩니다.
    ```
    KECCAK256(deployer.address, salt, contractCodeHash)
    ```

팩토리가 풀 컨트랙트를 배포할 때 `CREATE2`를 사용하므로 풀이 다른 컨트랙트 및 오프체인 앱에서 계산할 수 있는 고유하고 결정적인 주소를 얻게 되므로 차이점을 알아야 합니다. 특히 솔트의 경우 팩토리는 다음 풀 매개변수를 사용하여 해시를 계산합니다.
```solidity
keccak256(abi.encodePacked(token0, token1, tickSpacing))
```

`token0` 및 `token1`은 풀 토큰의 주소이고, `tickSpacing`은 다음에 배울 내용입니다.

## 틱 간격

`swap` 함수의 루프를 회상해 봅시다.
```solidity
while (
    state.amountSpecifiedRemaining > 0 &&
    state.sqrtPriceX96 != sqrtPriceLimitX96
) {
    ...
    (step.nextTick, ) = tickBitmap.nextInitializedTickWithinOneWord(...);
    (state.sqrtPriceX96, step.amountIn, step.amountOut) = SwapMath.computeSwapStep(...);
    ...
}
```

이 루프는 어느 방향으로든 틱을 반복하여 일부 유동성을 가진 초기화된 틱을 찾습니다. 그러나 이 반복은 비용이 많이 드는 작업입니다. 틱이 멀리 떨어져 있으면 코드는 현재 틱과 대상 틱 사이의 모든 틱을 통과해야 하므로 가스가 소모됩니다. 이 루프를 가스 효율적으로 만들기 위해 유니스왑 풀에는 이름에서 알 수 있듯이 틱 사이의 거리를 설정하는 `tickSpacing` 설정이 있습니다. 거리가 넓을수록 스왑이 가스 효율적입니다.

그러나 틱 간격이 넓을수록 정밀도가 낮아집니다. 낮은 변동성 쌍 (예 : 스테이블 코인 쌍) 은 가격 변동이 좁기 때문에 더 높은 정밀도가 필요합니다. 중간 및 높은 변동성 쌍은 가격 변동이 넓기 때문에 더 낮은 정밀도가 필요합니다. 이러한 다양성을 처리하기 위해 유니스왑은 쌍이 배포될 때 틱 간격을 선택할 수 있도록 합니다. 유니스왑은 10, 60 또는 200 중에서 선택할 수 있도록 합니다. 그리고 단순성을 위해 10과 60만 사용할 것입니다.

기술적으로 틱 인덱스는 `tickSpacing`의 배수여야 합니다. `tickSpacing`이 10이면 10의 배수만 틱 인덱스로 유효합니다 (10, 20, 5000, 5010, 그러나 8, 12, 5001 등은 아님). 그러나 중요하게도 현재 가격에는 적용되지 않습니다. 가능한 한 정확하게 유지하고 싶기 때문에 여전히 어떤 틱이든 될 수 있습니다. `tickSpacing`은 가격 범위에만 적용됩니다.

따라서 각 풀은 다음 매개변수 집합으로 고유하게 식별됩니다.
1. `token0`,
2. `token1`,
3. `tickSpacing`;

> 그렇습니다. 동일한 토큰이지만 틱 간격이 다른 풀이 있을 수 있습니다.

팩토리 컨트랙트는 이 매개변수 집합을 풀의 고유 식별자로 사용하고 새 풀 컨트랙트 주소를 생성하기 위해 솔트로 전달합니다.

> 이제부터 모든 풀에 대해 60의 틱 간격을 가정하고 스테이블 코인 쌍에는 10을 사용합니다. 이러한 값으로 나눌 수 있는 틱만 틱 비트맵에서 초기화된 것으로 플래그가 지정될 수 있습니다. 예를 들어 틱 간격이 60일 때 -120, -60, 0, 60, 120 등의 틱만 초기화되어 유동성 범위에 사용될 수 있습니다.

## 팩토리 구현

팩토리의 생성자에서 지원되는 틱 간격을 초기화해야 합니다.
```solidity
// src/UniswapV3Factory.sol
contract UniswapV3Factory is IUniswapV3PoolDeployer {
    mapping(uint24 => bool) public tickSpacings;
    constructor() {
        tickSpacings[10] = true;
        tickSpacings[60] = true;
    }

    ...
```

> 상수로 만들 수도 있었지만, 나중에 마일스톤을 위해 매핑으로 만들어야 합니다 (틱 간격은 다른 스왑 수수료 금액을 갖게 됩니다).

팩토리 컨트랙트는 `createPool`이라는 하나의 함수만 있는 컨트랙트입니다. 함수는 풀을 만들기 전에 필요한 필수 검사로 시작합니다.
```solidity
// src/UniswapV3Factory.sol
contract UniswapV3Factory is IUniswapV3PoolDeployer {
    PoolParameters public parameters;
    mapping(address => mapping(address => mapping(uint24 => address)))
        public pools;

    ...

    function createPool(
        address tokenX,
        address tokenY,
        uint24 tickSpacing
    ) public returns (address pool) {
        if (tokenX == tokenY) revert TokensMustBeDifferent();
        if (!tickSpacings[tickSpacing]) revert UnsupportedTickSpacing();

        (tokenX, tokenY) = tokenX < tokenY
            ? (tokenX, tokenY)
            : (tokenY, tokenX);

        if (tokenX == address(0)) revert TokenXCannotBeZero();
        if (pools[tokenX][tokenY][tickSpacing] != address(0))
            revert PoolAlreadyExists();
        
        ...
```

토큰을 정렬하는 것은 이번이 처음입니다.
```solidity
(tokenX, tokenY) = tokenX < tokenY
    ? (tokenX, tokenY)
    : (tokenY, tokenX);
```

이제부터 풀 토큰 주소도 정렬될 것으로 예상합니다. 즉, 정렬될 때 `token0`이 `token1`보다 먼저 옵니다. 솔트 (및 풀 주소) 계산을 일관성 있게 만들기 위해 이를 시행할 것입니다.

> 이 변경 사항은 테스트 및 배포 스크립트에서 토큰을 배포하는 방식에도 영향을 미칩니다. 솔리디티에서 가격 계산을 더 간단하게 만들기 위해 WETH가 항상 `token0`인지 확인해야 합니다 (그렇지 않으면 1/5000과 같은 분수 가격을 사용해야 합니다). 테스트에서 WETH가 `token0`이 아니면 토큰 배포 순서를 변경하십시오.

그 후 풀 매개변수를 준비하고 풀을 배포합니다.
```solidity
parameters = PoolParameters({
    factory: address(this),
    token0: tokenX,
    token1: tokenY,
    tickSpacing: tickSpacing
});

pool = address(
    new UniswapV3Pool{
        salt: keccak256(abi.encodePacked(tokenX, tokenY, tickSpacing))
    }()
);

delete parameters;
```

`parameters`가 사용되지 않기 때문에 이 부분은 이상하게 보입니다. 유니스왑은 [제어 역전](https://en.wikipedia.org/wiki/Inversion_of_control)을 사용하여 배포 중에 매개변수를 풀에 전달합니다. 업데이트된 풀 컨트랙트 생성자를 살펴 보겠습니다.
```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    ...
    constructor() {
        (factory, token0, token1, tickSpacing) = IUniswapV3PoolDeployer(
            msg.sender
        ).parameters();
    }
    ..
}
```

아하! 풀은 배포자가 `IUniswapV3PoolDeployer` 인터페이스 ( `parameters()` getter 만 정의) 를 구현할 것으로 예상하고 배포 중에 생성자에서 호출하여 매개변수를 가져옵니다. 이것이 흐름입니다.
1. `Factory`: `parameters` 상태 변수 ( `IUniswapV3PoolDeployer` 구현) 를 정의하고 풀을 배포하기 전에 설정합니다.
2. `Factory`: 풀을 배포합니다.
3. `Pool`: 생성자에서 배포자에서 `parameters()` 함수를 호출하고 풀 매개변수가 반환될 것으로 예상합니다.
4. `Factory`: `delete parameters;` 를 호출하여 `parameters` 상태 변수의 슬롯을 정리하고 가스 소비를 줄입니다. 이것은 `createPool()` 호출 중에만 값을 갖는 임시 상태 변수입니다.

풀이 생성된 후에는 `pools` 매핑에 보관하고 (토큰으로 찾을 수 있도록) 이벤트를 발생시킵니다.
```solidity
    pools[tokenX][tokenY][tickSpacing] = pool;
    pools[tokenY][tokenX][tickSpacing] = pool;

    emit PoolCreated(tokenX, tokenY, tickSpacing, pool);
}
```

## 풀 초기화

위의 코드에서 알 수 있듯이 더 이상 풀의 생성자에서 `sqrtPriceX96` 및 `tick`을 설정하지 않습니다. 이제 별도의 함수인 `initialize`에서 수행되며 풀이 배포된 후 호출해야 합니다.

```solidity
// src/UniswapV3Pool.sol
function initialize(uint160 sqrtPriceX96) public {
    if (slot0.sqrtPriceX96 != 0) revert AlreadyInitialized();

    int24 tick = TickMath.getTickAtSqrtRatio(sqrtPriceX96);

    slot0 = Slot0({sqrtPriceX96: sqrtPriceX96, tick: tick});
}
```

이것이 이제 풀을 배포하는 방법입니다.

```solidity
UniswapV3Factory factory = new UniswapV3Factory();
UniswapV3Pool pool = UniswapV3Pool(factory.createPool(token0, token1, tickSpacing));
pool.initialize(sqrtP(currentPrice));
```

## `PoolAddress` 라이브러리

이제 다른 컨트랙트에서 풀 컨트랙트 주소를 계산하는 데 도움이 되는 라이브러리를 구현해 보겠습니다. 이 라이브러리에는 `computeAddress`라는 하나의 함수만 있습니다.
```solidity
// src/lib/PoolAddress.sol
library PoolAddress {
    function computeAddress(
        address factory,
        address token0,
        address token1,
        uint24 tickSpacing
    ) internal pure returns (address pool) {
        require(token0 < token1);
        ...
```

이 함수는 풀 매개변수 (솔트를 빌드하는 데 사용됨) 와 팩토리 컨트랙트 주소를 알아야 합니다. 위에서 설명한 대로 토큰이 정렬될 것으로 예상합니다.

이제 함수의 핵심입니다.
```solidity
pool = address(
    uint160(
        uint256(
            keccak256(
                abi.encodePacked(
                    hex"ff",
                    factory,
                    keccak256(
                        abi.encodePacked(token0, token1, tickSpacing)
                    ),
                    keccak256(type(UniswapV3Pool).creationCode)
                )
            )
        )
    )
);
```

이것이 `CREATE2`가 새로운 컨트랙트 주소를 계산하기 위해 내부적으로 수행하는 작업입니다. 자세히 살펴 보겠습니다.

1. 먼저 솔트 ( `abi.encodePacked(token0, token1, tickSpacing)` ) 를 계산하고 해시합니다.
2. 그런 다음 풀 컨트랙트 코드 ( `type(UniswapV3Pool).creationCode` ) 를 얻고 해시합니다.
3. 그런 다음 `0xff`, 팩토리 컨트랙트 주소, 해시된 솔트 및 해시된 풀 컨트랙트 코드를 포함하는 바이트 시퀀스를 빌드합니다.
4. 그런 다음 시퀀스를 해시하고 주소로 변환합니다.

이러한 단계는 `CREATE2` opcode를 추가한 EIP 인 [EIP-1014](https://eips.ethereum.org/EIPS/eip-1014) 에 정의된 대로 컨트랙트 주소 생성을 구현합니다. 해시된 바이트 시퀀스를 구성하는 값을 자세히 살펴 보겠습니다.
1. `0xff`는 EIP에 정의된 대로 `CREATE` 및 `CREATE2`로 생성된 주소를 구별하는 데 사용됩니다.
2. `factory`는 배포자의 주소입니다. 이 경우 팩토리 컨트랙트입니다.
3. 솔트는 앞에서 설명했습니다. 풀을 고유하게 식별합니다.
4. 해시된 컨트랙트 코드는 충돌로부터 보호하는 데 필요합니다. 다른 컨트랙트가 동일한 솔트를 가질 수 있지만 코드 해시는 다릅니다.

따라서 이 체계에 따르면 컨트랙트 주소는 배포자, 코드 및 고유 매개변수를 포함하여 이 컨트랙트를 고유하게 식별하는 값의 해시입니다. 외부 호출을 하거나 팩토리를 쿼리하지 않고도 어디에서나 이 함수를 사용하여 풀 주소를 찾을 수 있습니다.

## 매니저 및 쿼터의 단순화된 인터페이스

매니저 및 쿼터 컨트랙트에서는 더 이상 사용자에게 풀 주소를 요청할 필요가 없습니다! 이렇게 하면 사용자가 풀 주소를 알 필요가 없고 토큰만 알면 되므로 컨트랙트와의 상호 작용이 더 쉬워집니다. 그러나 사용자는 풀의 솔트에 포함되어 있으므로 틱 간격도 지정해야 합니다.

또한 토큰 정렬 덕분에 항상 알 수 있으므로 더 이상 사용자에게 `zeroForOne` 플래그를 요청할 필요가 없습니다. 풀의 `token0`이 항상 `token1`보다 작으므로 "from token"이 "to token"보다 작으면 `zeroForOne`은 참입니다. 마찬가지로 "from token"이 "to token"보다 크면 `zeroForOne`은 항상 거짓입니다.

> 주소는 해시이고 해시는 숫자이므로 주소를 비교할 때 "보다 작음" 또는 "보다 큼"이라고 말할 수 있습니다.