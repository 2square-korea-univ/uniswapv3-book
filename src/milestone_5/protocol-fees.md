# 프로토콜 수수료

Uniswap 구현을 작업하면서, "Uniswap은 어떻게 돈을 벌까?"라는 질문을 스스로에게 던져봤을 것입니다. 음, (적어도 2022년 9월 현재로는) 그렇지 않습니다.

지금까지 구축한 구현에서, 거래자들은 유동성을 제공한 것에 대한 대가로 유동성 공급자에게 수수료를 지불하며, DEX를 개발한 회사인 Uniswap Labs는 이 과정에 참여하지 않습니다. 거래자나 유동성 공급자 모두 Uniswap DEX를 사용하는 것에 대해 Uniswap Labs에 비용을 지불하지 않습니다. 어째서일까요?

Uniswap Labs가 DEX에서 수익을 창출하기 시작할 수 있는 방법이 있습니다. 하지만, 그 메커니즘은 아직 활성화되지 않았습니다 (다시 한번, 2022년 9월 현재). 각 Uniswap 풀에는 *프로토콜 수수료* 징수 메커니즘이 있습니다. 프로토콜 수수료는 스왑 수수료에서 징수됩니다: 스왑 수수료의 작은 부분이 차감되어 프로토콜 수수료로 저장되며, 나중에 팩토리 컨트랙트 소유자 (Uniswap Labs)가 징수합니다. 프로토콜 수수료의 크기는 UNI 토큰 보유자가 결정할 것으로 예상되지만, 스왑 수수료의 $1/4$에서 $1/10$ (포함) 사이여야 합니다.

간결성을 위해, 저희는 프로토콜 수수료를 구현에 추가하지 않을 것입니다. 하지만 Uniswap에서 어떻게 구현되었는지 살펴봅시다.

프로토콜 수수료 크기는 `slot0`에 저장됩니다:

```solidity
// UniswapV3Pool.sol
struct Slot0 {
    ...
    // 인출 시 스왑 수수료의 백분율로 적용되는 현재 프로토콜 수수료
    // 정수 분모 (1/x)%로 표현
    uint8 feeProtocol;
    ...
}
```

축적된 수수료를 추적하기 위한 글로벌 누적기가 필요합니다:
```solidity
// token0/token1 단위로 축적된 프로토콜 수수료
struct ProtocolFees {
    uint128 token0;
    uint128 token1;
}
ProtocolFees public override protocolFees;
```

프로토콜 수수료는 `setFeeProtocol` 함수에서 설정됩니다:

```solidity
function setFeeProtocol(uint8 feeProtocol0, uint8 feeProtocol1) external override lock onlyFactoryOwner {
    require(
        (feeProtocol0 == 0 || (feeProtocol0 >= 4 && feeProtocol0 <= 10)) &&
            (feeProtocol1 == 0 || (feeProtocol1 >= 4 && feeProtocol1 <= 10))
    );
    uint8 feeProtocolOld = slot0.feeProtocol;
    slot0.feeProtocol = feeProtocol0 + (feeProtocol1 << 4);
    emit SetFeeProtocol(feeProtocolOld % 16, feeProtocolOld >> 4, feeProtocol0, feeProtocol1);
}
```

보시다시피, 각 토큰에 대해 프로토콜 수수료를 개별적으로 설정하는 것이 허용됩니다. 값은 두 개의 `uint8`이며, 하나의 `uint8`에 저장되도록 패킹됩니다: `feeProtocol1`은 왼쪽으로 4비트 시프트(16을 곱하는 것과 동일)되고 `feeProtocol0`에 더해집니다. `feeProtocol0`을 언패킹하려면, `slot0.feeProtocol`을 16으로 나눈 나머지를 취합니다. `feeProtocol1`은 단순히 `slot0.feeProtocol`을 오른쪽으로 4비트 시프트합니다. 이러한 패킹은 `feeProtocol0`나 `feeProtocol1` 모두 10보다 클 수 없기 때문에 작동합니다.

스왑을 시작하기 전에, 스왑 방향에 따라 프로토콜 수수료 중 하나를 선택해야 합니다 (스왑 및 프로토콜 수수료는 입력 토큰에 대해 징수됩니다):

```solidity
function swap(...) {
    ...
    uint8 feeProtocol = zeroForOne ? (slot0_.feeProtocol % 16) : (slot0_.feeProtocol >> 4);
    ...
```

프로토콜 수수료를 축적하기 위해, 스왑 단계 금액을 계산한 직후 스왑 수수료에서 차감합니다:

```solidity
...
while (...) {
    (..., step.feeAmount) = SwapMath.computeSwapStep(...);

    if (cache.feeProtocol > 0) {
        uint256 delta = step.feeAmount / cache.feeProtocol;
        step.feeAmount -= delta;
        state.protocolFee += uint128(delta);
    }

    ...
}
...
```

스왑이 완료된 후, 글로벌 프로토콜 수수료 누적기를 업데이트해야 합니다:
```solidity
if (zeroForOne) {
    if (state.protocolFee > 0) protocolFees.token0 += state.protocolFee;
} else {
    if (state.protocolFee > 0) protocolFees.token1 += state.protocolFee;
}
```

마지막으로, 팩토리 컨트랙트 소유자는 `collectProtocol`을 호출하여 축적된 프로토콜 수수료를 수집할 수 있습니다:

```solidity
function collectProtocol(
    address recipient,
    uint128 amount0Requested,
    uint128 amount1Requested
) external override lock onlyFactoryOwner returns (uint128 amount0, uint128 amount1) {
    amount0 = amount0Requested > protocolFees.token0 ? protocolFees.token0 : amount0Requested;
    amount1 = amount1Requested > protocolFees.token1 ? protocolFees.token1 : amount1Requested;

    if (amount0 > 0) {
        if (amount0 == protocolFees.token0) amount0--;
        protocolFees.token0 -= amount0;
        TransferHelper.safeTransfer(token0, recipient, amount0);
    }
    if (amount1 > 0) {
        if (amount1 == protocolFees.token1) amount1--;
        protocolFees.token1 -= amount1;
        TransferHelper.safeTransfer(token1, recipient, amount1);
    }

    emit CollectProtocol(msg.sender, recipient, amount0, amount1);
}
```