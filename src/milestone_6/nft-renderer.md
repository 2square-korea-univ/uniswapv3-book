# NFT 렌더러

이제 NFT 렌더러를 구축해야 합니다. NFT 렌더러는 NFT 관리자 컨트랙트에서 `tokenURI` 호출을 처리하는 라이브러리입니다. 이는 각 발행된 토큰에 대한 JSON 메타데이터와 SVG를 렌더링합니다. 앞서 논의했듯이 데이터 URI 형식을 사용할 것이며, 이는 base64 인코딩이 필요합니다. 즉, 솔리디티에 base64 인코더가 필요합니다. 하지만 먼저 토큰이 어떻게 생겼는지 살펴보겠습니다.

## SVG 템플릿

저는 유니스왑 V3 NFT의 단순화된 변형을 만들었습니다:



![NFT 토큰용 SVG 템플릿](images/nft_template.png)

다음은 해당 코드입니다.
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 300 480">
  <style>
    .tokens {
      font: bold 30px sans-serif;
    }

    .fee {
      font: normal 26px sans-serif;
    }

    .tick {
      font: normal 18px sans-serif;
    }
  </style>

  <rect width="300" height="480" fill="hsl(330,40%,40%)" />
  <rect x="30" y="30" width="240" height="420" rx="15" ry="15" fill="hsl(330,90%,50%)" stroke="#000" />

  <rect x="30" y="87" width="240" height="42" />
  <text x="39" y="120" class="tokens" fill="#fff">
    WETH/USDC
  </text>

  <rect x="30" y="132" width="240" height="30" />
  <text x="39" y="120" dy="36" class="fee" fill="#fff">
    0.05%
  </text>

  <rect x="30" y="342" width="240" height="24" />
  <text x="39" y="360" class="tick" fill="#fff">
    Lower tick: 123456
  </text>

  <rect x="30" y="372" width="240" height="24" />
  <text x="39" y="360" dy="30" class="tick" fill="#fff">
    Upper tick: 123456
  </text>
</svg>
```

이것은 간단한 SVG 템플릿이며, 이 템플릿의 필드를 채우고 `tokenURI`로 반환하는 솔리디티 컨트랙트를 만들 것입니다. 각 토큰마다 고유하게 채워질 필드는 다음과 같습니다:
1. 배경색, 이는 처음 두 개의 `rect`에서 설정됩니다. 템플릿에서 색상 구성 요소(330)는 각 토큰마다 고유합니다.
2. 포지션이 속한 풀의 토큰 이름 (템플릿의 WETH/USDC).
3. 풀의 수수료 (0.05%).
4. 포지션 경계의 틱 값 (123456).

다음은 컨트랙트가 생성할 수 있는 NFT의 예시입니다:



![NFT 예시 1](images/nft_example_2.png)


![NFT 예시 2](images/nft_example_3.png)


## 의존성

솔리디티는 기본 Base64 인코딩 도구를 제공하지 않으므로 타사 도구를 사용할 것입니다. 특히, [OpenZeppelin의 도구](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Base64.sol)를 사용할 것입니다.

솔리디티의 또 다른 지루한 점은 문자열 연산에 대한 지원이 매우 부족하다는 것입니다. 예를 들어, 정수를 문자열로 변환할 방법이 없습니다. 하지만 SVG 템플릿에서 풀 수수료와 포지션 틱을 렌더링하려면 필요합니다. 이를 위해 [OpenZeppelin의 Strings 라이브러리](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Strings.sol)를 사용할 것입니다.

## 결과 형식

렌더러가 생성하는 데이터는 다음과 같은 형식을 가집니다:

```
data:application/json;base64,BASE64_ENCODED_JSON
```

JSON은 다음과 같은 형태일 것입니다:
```json
{
  "name": "Uniswap V3 Position",
  "description": "USDC/DAI 0.05%, Lower tick: -520, Upper text: 490",
  "image": "BASE64_ENCODED_SVG"
}
```

이미지는 위 SVG 템플릿에 포지션 데이터를 채우고 Base64로 인코딩한 것입니다.

## 렌더러 구현

NFT 관리자 컨트랙트가 너무 복잡해지는 것을 막기 위해 렌더러를 별도의 라이브러리 컨트랙트에 구현할 것입니다:

```solidity
library NFTRenderer {
    struct RenderParams {
        address pool;
        address owner;
        int24 lowerTick;
        int24 upperTick;
        uint24 fee;
    }

    function render(RenderParams memory params) {
        ...
    }
}
```

`render` 함수에서 먼저 SVG를 렌더링한 다음 JSON을 렌더링합니다. 코드를 더 깔끔하게 유지하기 위해 각 단계를 더 작은 단계로 나눌 것입니다.

토큰 심볼을 가져오는 것으로 시작합니다:
```solidity
function render(RenderParams memory params) {
    IUniswapV3Pool pool = IUniswapV3Pool(params.pool);
    IERC20 token0 = IERC20(pool.token0());
    IERC20 token1 = IERC20(pool.token1());
    string memory symbol0 = token0.symbol();
    string memory symbol1 = token1.symbol();

    ...
```

### SVG 렌더링

그런 다음 SVG 템플릿을 렌더링할 수 있습니다:
```solidity
string memory image = string.concat(
    "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 300 480'>",
    "<style>.tokens { font: bold 30px sans-serif; }",
    ".fee { font: normal 26px sans-serif; }",
    ".tick { font: normal 18px sans-serif; }</style>",
    renderBackground(params.owner, params.lowerTick, params.upperTick),
    renderTop(symbol0, symbol1, params.fee),
    renderBottom(params.lowerTick, params.upperTick),
    "</svg>"
);
```

템플릿은 여러 단계로 나뉩니다:
1. 첫 번째는 CSS 스타일을 포함하는 헤더입니다.
2. 그 다음은 배경이 렌더링됩니다.
3. 그 다음은 상단 포지션 정보 (토큰 심볼 및 수수료)가 렌더링됩니다.
4. 마지막으로 하단 정보 (포지션 틱)가 렌더링됩니다.

배경은 단순히 두 개의 `rect` 요소입니다. 이를 렌더링하려면 이 토큰의 고유한 색조를 찾은 다음 모든 조각들을 함께 연결해야 합니다:
```solidity
function renderBackground(
    address owner,
    int24 lowerTick,
    int24 upperTick
) internal pure returns (string memory background) {
    bytes32 key = keccak256(abi.encodePacked(owner, lowerTick, upperTick));
    uint256 hue = uint256(key) % 360;

    background = string.concat(
        '<rect width="300" height="480" fill="hsl(',
        Strings.toString(hue),
        ',40%,40%)"/>',
        '<rect x="30" y="30" width="240" height="420" rx="15" ry="15" fill="hsl(',
        Strings.toString(hue),
        ',100%,50%)" stroke="#000"/>'
    );
}
```

상단 템플릿은 토큰 심볼과 풀 수수료를 렌더링합니다:
```solidity
function renderTop(
    string memory symbol0,
    string memory symbol1,
    uint24 fee
) internal pure returns (string memory top) {
    top = string.concat(
        '<rect x="30" y="87" width="240" height="42"/>',
        '<text x="39" y="120" class="tokens" fill="#fff">',
        symbol0,
        "/",
        symbol1,
        "</text>"
        '<rect x="30" y="132" width="240" height="30"/>',
        '<text x="39" y="120" dy="36" class="fee" fill="#fff">',
        feeToText(fee),
        "</text>"
    );
}
```

수수료는 소수 부분이 있는 숫자로 렌더링됩니다. 가능한 모든 수수료를 미리 알 수 있으므로 정수를 소수로 변환할 필요 없이 값을 하드코딩할 수 있습니다:
```solidity
function feeToText(uint256 fee)
    internal
    pure
    returns (string memory feeString)
{
    if (fee == 500) {
        feeString = "0.05%";
    } else if (fee == 3000) {
        feeString = "0.3%";
    }
}
```

하단 부분에서는 포지션 틱을 렌더링합니다:
```solidity
function renderBottom(int24 lowerTick, int24 upperTick)
    internal
    pure
    returns (string memory bottom)
{
    bottom = string.concat(
        '<rect x="30" y="342" width="240" height="24"/>',
        '<text x="39" y="360" class="tick" fill="#fff">Lower tick: ',
        tickToText(lowerTick),
        "</text>",
        '<rect x="30" y="372" width="240" height="24"/>',
        '<text x="39" y="360" dy="30" class="tick" fill="#fff">Upper tick: ',
        tickToText(upperTick),
        "</text>"
    );
}
```

틱은 양수와 음수가 될 수 있으므로 적절하게 렌더링해야 합니다 (마이너스 부호 유무에 따라):
```solidity
function tickToText(int24 tick)
    internal
    pure
    returns (string memory tickString)
{
    tickString = string.concat(
        tick < 0 ? "-" : "",
        tick < 0
            ? Strings.toString(uint256(uint24(-tick)))
            : Strings.toString(uint256(uint24(tick)))
    );
}
```

### JSON 렌더링

이제 `render` 함수로 돌아가서 JSON을 렌더링해 보겠습니다. 먼저 토큰 설명을 렌더링해야 합니다:
```solidity
function render(RenderParams memory params) {
    ... SVG 렌더링 ...

    string memory description = renderDescription(
        symbol0,
        symbol1,
        params.fee,
        params.lowerTick,
        params.upperTick
    );

    ...
```

토큰 설명은 토큰의 SVG에서 렌더링하는 모든 동일한 정보를 포함하는 텍스트 문자열입니다:
```solidity
function renderDescription(
    string memory symbol0,
    string memory symbol1,
    uint24 fee,
    int24 lowerTick,
    int24 upperTick
) internal pure returns (string memory description) {
    description = string.concat(
        symbol0,
        "/",
        symbol1,
        " ",
        feeToText(fee),
        ", Lower tick: ",
        tickToText(lowerTick),
        ", Upper text: ",
        tickToText(upperTick)
    );
}
```

이제 JSON 메타데이터를 조립할 수 있습니다:
```solidity
function render(RenderParams memory params) {
    string memory image = ...SVG 렌더링...
    string memory description = ...description 렌더링...

    string memory json = string.concat(
        '{"name":"Uniswap V3 Position",',
        '"description":"',
        description,
        '",',
        '"image":"data:image/svg+xml;base64,',
        Base64.encode(bytes(image)),
        '"}'
    );
```

마지막으로 결과를 반환할 수 있습니다:

```solidity
return
    string.concat(
        "data:application/json;base64,",
        Base64.encode(bytes(json))
    );
```

### `tokenURI`의 빈 공간 채우기

이제 NFT 관리자 컨트랙트의 `tokenURI` 함수로 돌아가서 실제 렌더링을 추가할 준비가 되었습니다:

```solidity
function tokenURI(uint256 tokenId)
    public
    view
    override
    returns (string memory)
{
    TokenPosition memory tokenPosition = positions[tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);

    return
        NFTRenderer.render(
            NFTRenderer.RenderParams({
                pool: tokenPosition.pool,
                owner: address(this),
                lowerTick: tokenPosition.lowerTick,
                upperTick: tokenPosition.upperTick,
                fee: pool.fee()
            })
        );
}
```

# 가스 비용

온체인에 데이터를 저장하는 것은 모든 이점에도 불구하고 큰 단점이 있습니다: 컨트랙트 배포 비용이 매우 비싸집니다. 컨트랙트를 배포할 때 컨트랙트 크기에 대한 비용을 지불하며, 모든 문자열과 템플릿은 가스 소비를 크게 증가시킵니다. SVG가 더 복잡해질수록 상황은 더욱 악화됩니다: 도형, CSS 스타일, 애니메이션 등이 많을수록 비용이 더 많이 듭니다.

위에서 구현한 NFT 렌더러는 가스 최적화되지 않았다는 점을 명심하십시오: 반복되는 `rect` 및 `text` 태그 문자열이 내부 함수로 추출될 수 있는 것을 볼 수 있습니다. 저는 컨트랙트의 가독성을 위해 가스 효율성을 희생했습니다. 실제 모든 데이터를 온체인에 저장하는 NFT 프로젝트에서는 가스 비용 최적화 때문에 코드 가독성이 일반적으로 매우 낮습니다.

# 테스팅

여기서 마지막으로 집중하고 싶었던 것은 NFT 이미지를 테스트하는 방법입니다. NFT 이미지의 모든 변경 사항을 추적하여 어떤 변경 사항도 렌더링을 손상시키지 않도록 하는 것이 매우 중요합니다. 이를 위해 `tokenURI`의 출력과 다양한 변형을 테스트하는 방법이 필요합니다 (전체 컬렉션을 미리 렌더링하고 개발 중에 이미지가 손상되지 않도록 테스트할 수도 있습니다).

`tokenURI`의 출력을 테스트하기 위해 다음과 같은 사용자 정의 어설션을 추가했습니다:

```solidity
assertTokenURI(
    nft.tokenURI(tokenId0),
    "tokenuri0",
    "invalid token URI"
);
```

첫 번째 인수는 실제 출력이고 두 번째 인수는 예상 출력을 저장하는 파일의 이름입니다. 어설션은 파일의 내용을 로드하고 실제 출력과 비교합니다:

```solidity
function assertTokenURI(
    string memory actual,
    string memory expectedFixture,
    string memory errMessage
) internal {
    string memory expected = vm.readFile(
        string.concat("./test/fixtures/", expectedFixture)
    );

    assertEq(actual, string(expected), errMessage);
}
```

Forge와 함께 제공되는 헬퍼 라이브러리인 `forge-std` 라이브러리에서 제공하는 `vm.readFile()` 치트 코드 덕분에 솔리디티에서 이를 수행할 수 있습니다. 이것은 간단하고 편리할 뿐만 아니라 안전하기도 합니다: 허용된 파일 작업만 허용하도록 파일 시스템 권한을 구성할 수 있습니다. 특히, 위의 테스트가 작동하도록 하려면 `foundry.toml`에 다음과 같은 `fs_permissions` 규칙을 추가해야 합니다:
```toml
fs_permissions = [{access='read',path='.'}]
```

다음은 `tokenURI` 픽스처에서 SVG를 읽는 방법입니다:
```shell
$ cat test/fixtures/tokenuri0 \
    | awk -F ',' '{print $2}' \
    | base64 -d - \
    | jq -r .image \
    | awk -F ',' '{print $2}' \
    | base64 -d - > nft.svg \
    && open nft.svg
```

> [jq tool](https://stedolan.github.io/jq/)이 설치되어 있는지 확인하십시오.