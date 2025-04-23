# Output Amount 계산

저희 Uniswap 수학 공식 모음의 마지막 부분은 ETH 판매 (토큰 $x$ 판매) 시 output amount 계산 공식입니다. 이전 단계에서는 ETH 구매 (토큰 $x$ 구매) 시나리오에 대한 유사한 공식이 있었습니다:

$$\Delta \sqrt{P} = \frac{\Delta y}{L}$$

이 공식은 토큰 $y$를 판매할 때 가격 변화를 찾습니다. 그런 다음 목표 가격을 찾기 위해 이 변화량을 현재 가격에 더했습니다:

$$\sqrt{P_{target}} = \sqrt{P_{current}} + \Delta \sqrt{P}$$

이제 토큰 $x$ (저희의 경우 ETH)를 판매하고 토큰 $y$ (저희의 경우 USDC)를 구매할 때 목표 가격을 찾는 유사한 공식이 필요합니다.

토큰 $x$의 변화량은 다음과 같이 계산됨을 기억하십시오:

$$\Delta x = \Delta \frac{1}{\sqrt{P}}L$$

이 공식으로부터 목표 가격을 찾을 수 있습니다:

$$\Delta x = (\frac{1}{\sqrt{P_{target}}} - \frac{1}{\sqrt{P_{current}}}) L$$
$$= \frac{L}{\sqrt{P_{target}}} - \frac{L}{\sqrt{P_{current}}}$$

이로부터 기본적인 대수 변환을 사용하여 $\sqrt{P_{target}}$를 찾을 수 있습니다:

$$\sqrt{P_{target}} = \frac{\sqrt{P}L}{\Delta x \sqrt{P} + L}$$

목표 가격을 알면 이전 단계에서 output amount를 찾았던 방식과 유사하게 output amount를 찾을 수 있습니다.

새로운 공식으로 Python 스크립트를 업데이트해 봅시다:
```python
# Swap ETH for USDC
amount_in = 0.01337 * eth

print(f"\nSelling {amount_in/eth} ETH")

price_next = int((liq * q96 * sqrtp_cur) // (liq * q96 + amount_in * sqrtp_cur))

print("New price:", (price_next / q96) ** 2)
print("New sqrtP:", price_next)
print("New tick:", price_to_tick((price_next / q96) ** 2))

amount_in = calc_amount0(liq, price_next, sqrtp_cur)
amount_out = calc_amount1(liq, price_next, sqrtp_cur)

print("ETH in:", amount_in / eth)
print("USDC out:", amount_out / eth)
```

출력 결과:
```shell
Selling 0.01337 ETH
New price: 4993.777388290041
New sqrtP: 5598789932670289186088059666432
New tick: 85163
ETH in: 0.013369999999998142
USDC out: 66.80838889019013
```

이는 이전 단계에서 제공한 유동성을 사용하여 0.01337 ETH를 판매할 때 66.8 USDC를 얻게 된다는 것을 의미합니다.

좋아 보이네요, 하지만 Python은 충분합니다! 이제 모든 수학 계산을 Solidity로 구현할 것입니다.