## NFT 포지션

이것은 이 책의 **정점**입니다. 이번 중요한 단계에서는 유니스왑 컨트랙트가 어떻게 확장되어 제3자 프로토콜에 통합될 수 있는지 배우게 됩니다. 이러한 가능성은 핵심 컨트랙트가 필수적인 기능만을 갖도록 설계되었기 때문에 직접적으로 비롯된 결과입니다. 이는 핵심 컨트랙트에 새로운 기능을 추가할 필요 없이 다른 컨트랙트로 통합할 수 있도록 합니다.

유니스왑 V3의 보너스 기능은 유동성 포지션을 NFT 토큰으로 전환하는 기능이었습니다. 다음은 그러한 토큰의 예시입니다.

<p align="center">
<img src="images/nft_example.png" alt="Uniswap V3 NFT example"/>
</p>

토큰 심볼, 풀 수수료, 포지션 ID, 하위 및 상위 틱, 토큰 주소, 그리고 포지션이 제공되는 곡선 구간을 보여줍니다.

> 모든 유니스왑 V3 NFT 포지션은 [이 OpenSea 컬렉션](https://opensea.io/collection/uniswap-v3-positions)에서 확인할 수 있습니다.

이번 단계에서는 유동성 포지션의 NFT 토큰화를 추가할 것입니다!

자, 시작해 봅시다!

> 이번 챕터의 전체 코드는 [이 Github 브랜치](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_6)에서 찾을 수 있습니다.
>
> 이번 단계는 기존 컨트랙트에 많은 코드 변경 사항을 도입합니다. [여기에서 이전 단계 이후의 모든 변경 사항을 확인할 수 있습니다](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_5...milestone_6)

> 질문이 있으시면 [이번 단계의 GitHub Discussion](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-6-nft-positions)에 자유롭게 질문해주세요!