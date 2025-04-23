## 플래시 론

Uniswap V2와 V3 모두 플래시 론을 구현합니다. 플래시 론은 무제한이며 무담보 대출로, 동일한 트랜잭션 내에서 상환되어야 합니다. 풀은 사용자에게 요청하는 임의의 양의 토큰을 제공하지만, 호출이 끝나기 전에 소량의 수수료와 함께 해당 금액을 상환해야 합니다.

플래시 론은 동일한 트랜잭션 내에서 상환되어야 한다는 사실은 일반 사용자가 플래시 론을 이용할 수 없다는 것을 의미합니다. 일반 사용자는 트랜잭션 내에서 사용자 정의 로직을 프로그래밍할 수 없기 때문입니다. 플래시 론은 스마트 컨트랙트에 의해서만 실행 및 상환될 수 있습니다.

플래시 론은 DeFi에서 강력한 금융 도구입니다. 플래시 론은 종종 DeFi 프로토콜의 취약점을 악용하는 데 사용되지만 (풀 잔액을 부풀리고 결함 있는 상태 관리를 남용함으로써), 레버리지 포지션 관리와 같은 유용한 애플리케이션도 많습니다. – 이것이 유동성을 저장하는 DeFi 애플리케이션이 무허가형 플래시 론을 제공하는 이유입니다.

### 플래시 론 구현

Uniswap V2 플래시 론은 스왑 기능의 일부였습니다. 스왑 중에 토큰을 빌릴 수 있었지만, 동일한 트랜잭션 내에서 해당 토큰 또는 동일한 양의 다른 풀 토큰을 반환해야 했습니다. V3에서는 플래시 론이 스왑과 분리되었습니다. V3 플래시 론은 단순히 호출자에게 요청한 토큰 수를 제공하고, 호출자에 대한 콜백을 호출하며, 플래시 론이 상환되었는지 확인하는 함수입니다.

```solidity
function flash(
    uint256 amount0,
    uint256 amount1,
    bytes calldata data
) public {
    uint256 balance0Before = IERC20(token0).balanceOf(address(this));
    uint256 balance1Before = IERC20(token1).balanceOf(address(this));

    if (amount0 > 0) IERC20(token0).transfer(msg.sender, amount0);
    if (amount1 > 0) IERC20(token1).transfer(msg.sender, amount1);

    IUniswapV3FlashCallback(msg.sender).uniswapV3FlashCallback(data);

    require(IERC20(token0).balanceOf(address(this)) >= balance0Before);
    require(IERC20(token1).balanceOf(address(this)) >= balance1Before);

    emit Flash(msg.sender, amount0, amount1);
}
```

이 함수는 호출자에게 토큰을 보내고 호출자에게 `uniswapV3FlashCallback`을 호출합니다. – 여기가 호출자가 론을 상환해야 하는 곳입니다. 그런 다음 함수는 잔액이 감소하지 않았는지 확인합니다. 사용자 정의 데이터를 콜백으로 전달할 수 있다는 점에 유의하십시오.

다음은 콜백 구현의 예입니다.

```solidity
function uniswapV3FlashCallback(bytes calldata data) public {
    (uint256 amount0, uint256 amount1) = abi.decode(
        data,
        (uint256, uint256)
    );

    if (amount0 > 0) token0.transfer(msg.sender, amount0);
    if (amount1 > 0) token1.transfer(msg.sender, amount1);
}
```

이 구현에서는 단순히 토큰을 풀로 다시 보내고 있습니다 (저는 `flash` 함수 테스트에서 이 콜백을 사용했습니다). 실제로, 빌린 금액을 사용하여 다른 DeFi 프로토콜에서 특정 작업을 수행할 수 있습니다. 그러나 항상 이 콜백에서 론을 상환해야 합니다.

이것으로 끝입니다!