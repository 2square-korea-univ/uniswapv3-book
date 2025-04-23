## 틱 범위 간 스왑

지금까지 훌륭한 진전을 이루었으며, 저희 Uniswap V3 구현이 원본에 상당히 근접했습니다! 그러나 저희 구현은 가격 범위 내에서만 스왑을 지원하며, 이것이 이번 마일스톤에서 개선할 부분입니다.

이번 마일스톤에서는 다음을 수행할 것입니다:
1. 다양한 가격 범위에서 유동성을 제공하도록 `mint` 함수를 업데이트합니다.
2. 현재 가격 범위에 충분한 유동성이 없을 때 가격 범위를 교차하도록 `swap` 함수를 업데이트합니다.
3. 스마트 컨트랙트에서 유동성을 계산하는 방법을 배웁니다.
4. `mint` 및 `swap` 함수에 슬리피지 보호를 구현합니다.
5. 다양한 가격 범위에서 유동성을 추가할 수 있도록 UI 애플리케이션을 업데이트합니다.
6. 고정 소수점 숫자에 대해 조금 더 배웁니다.

이번 마일스톤에서는 Uniswap의 핵심 기능인 스왑을 완료할 것입니다!

시작해 봅시다!

> 이 챕터의 전체 코드는 [이 Github 브랜치](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_3)에서 찾을 수 있습니다.
>
> 이번 마일스톤은 기존 컨트랙트에 많은 코드 변경 사항을 도입합니다. [여기에서 마지막 마일스톤 이후의 모든 변경 사항을 볼 수 있습니다](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_2...milestone_3)

> 질문이 있으시면 [이 마일스톤의 GitHub 토론](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-3-cross-tick-swaps)에서 자유롭게 질문해주세요!