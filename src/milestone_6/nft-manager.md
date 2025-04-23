## NFT ManagerContract

우리는 풀 컨트랙트에 NFT 관련 기능을 추가하지 않을 것입니다. 대신 NFT와 유동성 포지션을 병합할 별도의 컨트랙트가 필요합니다. 구현 작업을 진행하면서 풀 컨트랙트와의 상호 작용을 용이하게 하기 위해 `UniswapV3Manager` 컨트랙트를 구축했던 것을 상기해 보십시오 (일부 계산을 더 간단하게 만들고 다중 풀 스왑을 가능하게 하기 위해). 이 컨트랙트는 핵심 Uniswap 컨트랙트가 어떻게 확장될 수 있는지 보여주는 좋은 예시였습니다. 그리고 우리는 이 아이디어를 조금 더 발전시킬 것입니다.

우리는 ERC721 표준을 구현하고 유동성 포지션을 관리할 관리자 컨트랙트가 필요합니다. 이 컨트랙트는 표준 NFT 기능 (발행, 소각, 전송, 잔액 및 소유권 추적 등)을 가지며 풀에 유동성을 공급하고 제거할 수 있도록 합니다. 이 컨트랙트는 풀 유동성의 실제 소유자여야 합니다. 왜냐하면 사용자가 토큰을 발행하지 않고 유동성을 추가하거나 토큰을 소각하지 않고 전체 유동성을 제거하는 것을 원치 않기 때문입니다. 우리는 모든 유동성 포지션이 NFT 토큰에 연결되기를 원하며, 그것들이 동기화되기를 원합니다.

새로운 컨트랙트에 어떤 기능이 있을지 살펴봅시다:
1. NFT 컨트랙트이므로 NFT 토큰 이미지의 URI를 반환하는 `tokenURI`를 포함한 모든 ERC721 기능이 있습니다.
1. 유동성과 NFT 토큰을 동시에 발행하고 소각하는 `mint` 및 `burn`;
1. 기존 포지션에 유동성을 추가 및 제거하는 `addLiquidity` 및 `removeLiquidity`;
1. 유동성 제거 후 토큰을 수집하는 `collect`.

자, 이제 코딩을 시작해 봅시다.

## 최소 컨트랙트

ERC721 표준을 처음부터 구현하고 싶지 않으므로 라이브러리를 사용할 것입니다. 이미 의존성에 [Solmate](https://github.com/transmissions11/solmate)가 있으므로 [Solmate의 ERC721 구현](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC721.sol)을 사용할 것입니다.

> [OpenZeppelin의 ERC721 구현](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/token/ERC721)을 사용하는 것도 선택 사항이지만, 저는 Solmate의 가스 최적화 컨트랙트를 선호합니다.

다음은 NFT 관리자 컨트랙트의 최소한의 모습입니다:

```solidity
contract UniswapV3NFTManager is ERC721 {
    address public immutable factory;

    constructor(address factoryAddress)
        ERC721("UniswapV3 NFT Positions", "UNIV3")
    {
        factory = factoryAddress;
    }

    function tokenURI(uint256 tokenId)
        public
        view
        override
        returns (string memory)
    {
        return "";
    }
}
```

`tokenURI`는 메타데이터 및 SVG 렌더러를 구현할 때까지 빈 문자열을 반환합니다. Solidity 컴파일러가 컨트랙트의 나머지 부분을 작업하는 동안 실패하지 않도록 스텁을 추가했습니다 (Solmate ERC721 컨트랙트의 `tokenURI` 함수는 가상이므로 구현해야 합니다).

## 발행 (Minting)

앞서 논의한 바와 같이 발행은 풀에 유동성을 추가하고 NFT를 발행하는 두 가지 작업을 포함합니다.

풀 유동성 포지션과 NFT 간의 연결을 유지하려면 매핑과 구조체가 필요합니다:

```solidity
struct TokenPosition {
    address pool;
    int24 lowerTick;
    int24 upperTick;
}
mapping(uint256 => TokenPosition) public positions;
```

포지션을 찾으려면 다음이 필요합니다:
1. 풀 주소;
2. 소유자 주소;
3. 포지션 경계 (하위 및 상위 틱).

NFT 관리자 컨트랙트는 이를 통해 생성된 모든 포지션의 소유자가 되므로 포지션의 소유자 주소를 저장할 필요가 없으며 나머지 데이터만 저장할 수 있습니다. `positions` 매핑의 키는 토큰 ID입니다. 이 매핑은 NFT ID를 유동성 포지션을 찾는 데 필요한 포지션 데이터에 연결합니다.

발행을 구현해 봅시다:

```solidity
struct MintParams {
    address recipient;
    address tokenA;
    address tokenB;
    uint24 fee;
    int24 lowerTick;
    int24 upperTick;
    uint256 amount0Desired;
    uint256 amount1Desired;
    uint256 amount0Min;
    uint256 amount1Min;
}

function mint(MintParams calldata params) public returns (uint256 tokenId) {
    ...
}
```

발행 파라미터는 `UniswapV3Manager`의 파라미터와 동일하며, NFT를 다른 주소로 발행할 수 있도록 하는 `recipient`가 추가되었습니다.

`mint` 함수에서 먼저 풀에 유동성을 추가합니다:

```solidity
IUniswapV3Pool pool = getPool(params.tokenA, params.tokenB, params.fee);

(uint128 liquidity, uint256 amount0, uint256 amount1) = _addLiquidity(
    AddLiquidityInternalParams({
        pool: pool,
        lowerTick: params.lowerTick,
        upperTick: params.upperTick,
        amount0Desired: params.amount0Desired,
        amount1Desired: params.amount1Desired,
        amount0Min: params.amount0Min,
        amount1Min: params.amount1Min
    })
);
```

`_addLiquidity`는 `UniswapV3Manager` 컨트랙트의 `mint` 함수 본문과 동일합니다. 틱을 $\sqrt(P)$로 변환하고, 유동성 양을 계산하고, `pool.mint()`를 호출합니다.

다음으로 NFT를 발행합니다:

```solidity
tokenId = nextTokenId++;
_mint(params.recipient, tokenId);
totalSupply++;
```

`tokenId`는 현재 `nextTokenId`로 설정되고, `nextTokenId`는 증가됩니다. `_mint` 함수는 Solmate의 ERC721 컨트랙트에서 제공됩니다. 새 토큰을 발행한 후 `totalSupply`를 업데이트합니다.

마지막으로 새 토큰과 새 포지션에 대한 정보를 저장해야 합니다:

```solidity
TokenPosition memory tokenPosition = TokenPosition({
    pool: address(pool),
    lowerTick: params.lowerTick,
    upperTick: params.upperTick
});

positions[tokenId] = tokenPosition;
```

이렇게 하면 나중에 토큰 ID로 유동성 포지션을 찾는 데 도움이 됩니다.

## 유동성 추가 (Adding Liquidity)

다음으로 기존 포지션에 유동성을 추가하는 함수를 구현합니다. 이미 유동성이 있는 포지션에 더 많은 유동성을 추가하려는 경우입니다. 이러한 경우 NFT를 발행하는 것이 아니라 기존 포지션의 유동성 양만 늘리고 싶습니다. 이를 위해 토큰 ID와 토큰 양만 제공하면 됩니다:

```solidity
function addLiquidity(AddLiquidityParams calldata params)
    public
    returns (
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    TokenPosition memory tokenPosition = positions[params.tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    (liquidity, amount0, amount1) = _addLiquidity(
        AddLiquidityInternalParams({
            pool: IUniswapV3Pool(tokenPosition.pool),
            lowerTick: tokenPosition.lowerTick,
            upperTick: tokenPosition.upperTick,
            amount0Desired: params.amount0Desired,
            amount1Desired: params.amount1Desired,
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min
        })
    );
}
```

이 함수는 기존 토큰이 있는지 확인하고 기존 포지션의 파라미터로 `pool.mint()`를 호출합니다.

## 유동성 제거 (Remove Liquidity)

`UniswapV3Manager` 컨트랙트에서는 사용자가 유동성 포지션의 소유자가 되기를 원했기 때문에 `burn` 함수를 구현하지 않았던 것을 상기해 보십시오. 이제 NFT 관리자가 소유자가 되기를 원합니다. 그리고 유동성 소각을 구현할 수 있습니다:

```solidity
struct RemoveLiquidityParams {
    uint256 tokenId;
    uint128 liquidity;
}

function removeLiquidity(RemoveLiquidityParams memory params)
    public
    isApprovedOrOwner(params.tokenId)
    returns (uint256 amount0, uint256 amount1)
{
    TokenPosition memory tokenPosition = positions[params.tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);

    (uint128 availableLiquidity, , , , ) = pool.positions(
        poolPositionKey(tokenPosition)
    );
    if (params.liquidity > availableLiquidity) revert NotEnoughLiquidity();

    (amount0, amount1) = pool.burn(
        tokenPosition.lowerTick,
        tokenPosition.upperTick,
        params.liquidity
    );
}
```

여기서도 제공된 토큰 ID가 유효한지 확인합니다. 그리고 포지션에 소각할 충분한 유동성이 있는지 확인해야 합니다.

## 토큰 수집 (Collecting Tokens)

NFT 관리자 컨트랙트는 유동성을 소각한 후 토큰을 수집할 수도 있습니다. 수집된 토큰은 호출자를 대신하여 컨트랙트가 유동성을 관리하므로 `msg.sender`에게 전송됩니다:

```solidity
struct CollectParams {
    uint256 tokenId;
    uint128 amount0;
    uint128 amount1;
}

function collect(CollectParams memory params)
    public
    isApprovedOrOwner(params.tokenId)
    returns (uint128 amount0, uint128 amount1)
{
    TokenPosition memory tokenPosition = positions[params.tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);

    (amount0, amount1) = pool.collect(
        msg.sender,
        tokenPosition.lowerTick,
        tokenPosition.upperTick,
        params.amount0,
        params.amount1
    );
}
```

## 소각 (Burning)

마지막으로 소각입니다. 컨트랙트의 다른 기능과는 달리 이 함수는 풀과 관련된 작업을 수행하지 않습니다. NFT만 소각합니다. NFT를 소각하려면 기본 포지션이 비어 있어야 하고 토큰이 수집되어야 합니다. 따라서 NFT를 소각하려면 다음을 수행해야 합니다:
1. `removeLiquidity`를 호출하여 전체 포지션 유동성을 제거합니다.
2. `collect`를 호출하여 포지션을 소각한 후 토큰을 수집합니다.
3. `burn`을 호출하여 토큰을 소각합니다.

```solidity
function burn(uint256 tokenId) public isApprovedOrOwner(tokenId) {
    TokenPosition memory tokenPosition = positions[tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);
    (uint128 liquidity, , , uint128 tokensOwed0, uint128 tokensOwed1) = pool
        .positions(poolPositionKey(tokenPosition));

    if (liquidity > 0 || tokensOwed0 > 0 || tokensOwed1 > 0)
        revert PositionNotCleared();

    delete positions[tokenId];
    _burn(tokenId);
    totalSupply--;
}
```

이것으로 끝입니다!