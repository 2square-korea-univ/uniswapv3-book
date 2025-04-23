## 사용자 인터페이스

이제 웹 앱을 실제 DEX처럼 작동하도록 만들어 보겠습니다. 하드코딩된 스왑 금액을 제거하고 사용자가 임의의 금액을 입력할 수 있도록 할 수 있습니다.  또한, 이제 사용자가 양방향으로 스왑할 수 있도록 토큰 입력을 스왑하는 버튼도 필요합니다. 업데이트 후 스왑 폼은 다음과 같습니다.

```jsx
<form className="SwapForm">
  <SwapInput
    amount={zeroForOne ? amount0 : amount1}
    disabled={!enabled || loading}
    readOnly={false}
    setAmount={setAmount_(zeroForOne ? setAmount0 : setAmount1, zeroForOne)}
    token={zeroForOne ? pair[0] : pair[1]} />
  <ChangeDirectionButton zeroForOne={zeroForOne} setZeroForOne={setZeroForOne} disabled={!enabled || loading} />
  <SwapInput
    amount={zeroForOne ? amount1 : amount0}
    disabled={!enabled || loading}
    readOnly={true}
    token={zeroForOne ? pair[1] : pair[0]} />
  <button className='swap' disabled={!enabled || loading} onClick={swap_}>Swap</button>
</form>
```

각 입력 필드에는 `zeroForOne` 상태 변수에 의해 제어되는 스왑 방향에 따라 금액이 할당됩니다. 하단 입력 필드는 항상 읽기 전용인데, 이는 해당 값이 Quoter 컨트랙트에 의해 계산되기 때문입니다.

`setAmount_` 함수는 두 가지 작업을 수행합니다. 상단 입력 필드의 값을 업데이트하고 Quoter 컨트랙트를 호출하여 하단 입력 필드의 값을 계산합니다.

```js
const updateAmountOut = debounce((amount) => {
  if (amount === 0 || amount === "0") {
    return;
  }

  setLoading(true);

  quoter.callStatic
    .quote({ pool: config.poolAddress, amountIn: ethers.utils.parseEther(amount), zeroForOne: zeroForOne })
    .then(({ amountOut }) => {
      zeroForOne ? setAmount1(ethers.utils.formatEther(amountOut)) : setAmount0(ethers.utils.formatEther(amountOut));
      setLoading(false);
    })
    .catch((err) => {
      zeroForOne ? setAmount1(0) : setAmount0(0);
      setLoading(false);
      console.error(err);
    })
})

const setAmount_ = (setAmountFn) => {
  return (amount) => {
    amount = amount || 0;
    setAmountFn(amount);
    updateAmountOut(amount)
  }
}
```

`quoter`에서 호출되는 `callStatic`에 주목하십시오. 이것이 이전 장에서 논의했던 내용입니다. Ethers.js가 정적 호출을 하도록 강제해야 합니다. `quote`는 `pure` 또는 `view` 함수가 아니기 때문에 Ethers.js는 트랜잭션 내에서 `quote`를 호출하려고 시도합니다.

이것으로 끝입니다! 이제 UI를 통해 임의의 금액을 지정하고 양방향으로 스왑할 수 있습니다!