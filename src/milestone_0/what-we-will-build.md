# 우리가 구축할 것

이 책의 목표는 Uniswap V3의 클론을 구축하는 것입니다. 하지만 완벽히 동일한 복사본을 만들지는 않을 것입니다. 주된 이유는 Uniswap은 복잡하고 다양한 부가 기능들을 포함하는 거대한 프로젝트이기 때문입니다. 이 모든 것을 책에서 다루는 것은 책을 너무 방대하게 만들고 독자들이 완독하기 어렵게 할 것입니다. 대신, Uniswap의 핵심 기능, 즉 가장 핵심적이고 중요한 메커니즘들을 구축할 것입니다. 여기에는 유동성 관리, 스왑, 수수료, 주변 컨트랙트, 견적 컨트랙트, 그리고 NFT 컨트랙트가 포함됩니다. 이후에는, 독자 여러분이 Uniswap V3의 소스 코드를 읽고 이 책의 범위에서 제외된 모든 기능들을 이해할 수 있을 것이라고 확신합니다.

## 스마트 컨트랙트

이 책을 마치고 나면, 독자 여러분은 다음 컨트랙트들을 구현하게 됩니다:
1. `UniswapV3Pool` – 유동성 관리와 스왑을 구현하는 핵심 풀 컨트랙트입니다. 이 컨트랙트는 [원본 컨트랙트](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol)와 매우 유사하지만, 구현 세부 사항은 약간 다르며, 단순화를 위해 몇 가지 부분은 생략되었습니다. 예를 들어, 저희 구현에서는 "정확한 입력" 스왑, 즉 입력량이 알려진 스왑만 처리할 것입니다. 원본 구현은 알려진 *출력* 금액(예: 특정량의 토큰을 구매하려는 경우)을 가진 스왑도 지원합니다.
1. `UniswapV3Factory` – 새로운 풀을 배포하고 배포된 모든 풀의 기록을 유지하는 레지스트리 컨트랙트입니다. 이 컨트랙트는 소유자와 수수료를 변경하는 기능을 제외하면 [원본 컨트랙트](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Factory.sol)와 대부분 동일합니다.
1. `UniswapV3Manager` – 풀 컨트랙트와 더 쉽게 상호 작용할 수 있도록 하는 주변 컨트랙트입니다. 이것은 [SwapRouter](https://github.com/Uniswap/v3-periphery/blob/main/contracts/SwapRouter.sol)의 매우 단순화된 구현입니다. 다시 말씀드리지만, 보시다시피, 저는 "정확한 입력" 및 "정확한 출력" 스왑을 구별하지 않고 전자의 스왑만 구현합니다.
1. `UniswapV3Quoter` – 온체인에서 스왑 가격을 계산할 수 있도록 하는 멋진 컨트랙트입니다. 이것은 [Quoter](https://github.com/Uniswap/v3-periphery/blob/main/contracts/lens/Quoter.sol)와 [QuoterV2](https://github.com/Uniswap/v3-periphery/blob/main/contracts/lens/QuoterV2.sol)의 최소한의 복사본입니다. 다시 한번, "정확한 입력" 스왑만 지원됩니다.
1. `UniswapV3NFTManager` – 유동성 포지션을 NFT로 전환할 수 있도록 합니다. 이것은 [NonfungiblePositionManager](https://github.com/Uniswap/v3-periphery/blob/main/contracts/NonfungiblePositionManager.sol)의 단순화된 구현입니다.

## 프론트엔드 애플리케이션

이 책에서는, [Uniswap UI](https://app.uniswap.org/)의 단순화된 클론 또한 만들었습니다. 이것은 매우 기본적인 클론이며, 제 React 및 프론트엔드 기술은 매우 부족하지만, 프론트엔드 애플리케이션이 Ethers.js와 MetaMask를 사용하여 스마트 컨트랙트와 상호 작용하는 방법을 보여줍니다.