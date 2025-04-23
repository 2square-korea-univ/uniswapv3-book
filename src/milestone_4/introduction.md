# 멀티 풀 스왑

틱 간 스왑을 구현한 후, 우리는 실제 Uniswap V3 스왑에 거의 근접했습니다. 우리 구현의 중요한 한계점은 풀 내에서의 스왑만 허용한다는 것입니다. 토큰 쌍에 대한 풀이 없다면, 이 토큰 간의 스왑은 불가능합니다. Uniswap에서는 그렇지 않습니다. 멀티 풀 스왑을 허용하기 때문입니다. 이번 장에서는 멀티 풀 스왑을 우리 구현에 추가할 것입니다.

계획은 다음과 같습니다:

1. 먼저, Factory 컨트랙트에 대해 배우고 구현할 것입니다;
2. 그 다음으로, 체인 스왑 또는 멀티 풀 스왑이 어떻게 작동하는지 살펴보고 Path 라이브러리를 구현할 것입니다;
3. 그 후, 프론트엔드 앱을 업데이트하여 멀티 풀 스왑을 지원하도록 할 것입니다;
4. 두 토큰 간의 경로를 찾는 기본적인 라우터를 구현할 것입니다;
5. 진행하면서 스왑 최적화 방법인 틱 간격에 대해서도 배울 것입니다.

이번 장을 마치면, 우리 구현은 멀티 풀 스왑을 처리할 수 있게 될 것입니다. 예를 들어, 다양한 스테이블 코인을 거쳐 WBTC를 WETH로 스왑하는 것과 같습니다: WETH → USDC → USDT → WBTC.

시작해 봅시다!

> 이번 장의 전체 코드는 [이 Github 브랜치](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_4)에서 찾을 수 있습니다.
>
> 이번 마일스톤에서는 기존 컨트랙트에 많은 코드 변경이 도입되었습니다. [여기에서 지난 마일스톤 이후의 모든 변경 사항을 볼 수 있습니다](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_3...milestone_4)

> 질문이 있으시면 [이 마일스톤의 GitHub 토론](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-4-multi-pool-swaps)에서 자유롭게 질문해주세요!