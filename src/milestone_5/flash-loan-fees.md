# 플래시 론 수수료

이전 장에서 우리는 플래시 론을 구현했고 무료로 만들었습니다. 하지만, Uniswap은 플래시 론에 대해 스왑 수수료를 징수합니다. 그리고 우리는 이것을 우리 구현에 추가할 것입니다: 플래시 론 차용자가 상환하는 금액은 수수료를 포함해야 합니다.

업데이트된 `flash` 함수는 다음과 같습니다:
```solidity
function flash(
    uint256 amount0,
    uint256 amount1,
    bytes calldata data
) public {
    uint256 fee0 = Math.mulDivRoundingUp(amount0, fee, 1e6);
    uint256 fee1 = Math.mulDivRoundingUp(amount1, fee, 1e6);

    uint256 balance0Before = IERC20(token0).balanceOf(address(this));
    uint256 balance1Before = IERC20(token1).balanceOf(address(this));

    if (amount0 > 0) IERC20(token0).transfer(msg.sender, amount0);
    if (amount1 > 0) IERC20(token1).transfer(msg.sender, amount1);

    IUniswapV3FlashCallback(msg.sender).uniswapV3FlashCallback(
        fee0,
        fee1,
        data
    );

    if (IERC20(token0).balanceOf(address(this)) < balance0Before + fee0)
        revert FlashLoanNotPaid();
    if (IERC20(token1).balanceOf(address(this)) < balance1Before + fee1)
        revert FlashLoanNotPaid();

    emit Flash(msg.sender, amount0, amount1);
}
```

변경된 점은 이제 호출자가 요청한 금액에 대해 수수료를 계산하고, 풀 잔액이 수수료 금액만큼 증가할 것으로 예상한다는 것입니다.