# BAL Liquidity Mining

## BAL Distribution Proportional to Liquidity on Balancer <a id="353e"></a>

To make the token distribution as fair as possible, we distribute BAL tokens proportional to the amount of liquidity each address contributed, relative to the total liquidity on Balancer. Since there is liquidity in several different tokens, we use the USD value as the common measure.

In practice, every week Balancer Labs has to:

* Define the starting and ending block of the week. Both are chosen as the block with the closest timestamp to a fixed weekly time \(e.g. Sunday 1:00pm UTC\). For example, the starting block for a given week might be \#10,100,000 and the ending block \#10,140,000.
* Define snapshot blocks, every 256 blocks \(roughly hourly\) counting backwards from the ending block until the starting block. For the example above, the snapshot blocks would be \#10,140,000, \#10,139,744, \#10,139,488, and so on.
* For each snapshot block, and for each Balancer pool, get the USD price of the tokens in the pool from [CoinGecko](https://www.coingecko.com/api/documentations/v3#/contract/get_coins__id__contract__contract_address__market_chart_), and calculate the total USD liquidity.
* Since liquidity in pools that have lower trading fees contribute more to protocol usage than liquidity in pools with higher fees, we multiply the USD pool liquidity by a feeFactor that down-weights pools according to their fee percentage:

![](../../.gitbook/assets/fee_factor_calc.png)

The constant, `k`, was initially set to 0.5, but beginning in week 8 \(July 20th 00:00 UTC\), the community approved a [proposal to to change `k` to 0.25](https://forum.balancer.finance/t/modifying-feefactor-toward-reducing-the-mining-penalty-for-high-fee-pools/103) in order to ease the penalty for higher-fee pools. This creates the following bell-shaped curve for feeFactor, which means, for example, that a pool with a 0.5% fee has a feeFactor of ~0.98, a pool with a 1% fee has a feeFactor of ~0.94, and a pool with a 2% fee has a feeFactor of ~0.78:

![](../../.gitbook/assets/fee_factor_plot.png)

* **UPDATED for week 2** \(starting June 8th 00:00 UTC\), multiply the pool liquidity by a `ratioFactor`. Since pools that are imbalanced contribute less to trading volume \(because the slippage is higher\), the community approved a [proposal to add a ratioFactor](https://forum.balancer.finance/t/introduction-of-a-weight-ratio-factor-in-liquidity-mining/15). This way highly imbalanced pools \(such as those with 98%/2% weights\) have a much lower weight in the final BAL distribution.
* **UPDATED for week 3** \(starting June 15th 00:00 UTC\), multiply the pool liquidity by a `wrapFactor`. Since pools containing pairs of tokens that have a hard peg \(e.g. DAI and cDAI\) do not contribute much trading volume \(because traders can wrap DAI for cDAI and vice-versa\), the community approved a [proposal to add a wrapFactor](https://forum.balancer.finance/t/wrapfactor-penalizing-pairs-of-equivalent-tokens-in-liquidity-mining/28/3). This way, liquidity in such pairs \(like cETH/ WETH\) has a 0.1 wrapFactor \(i.e. counts 10 times less than for other regular pairs\), reducing the amount of BAL received by their liquidity providers. In week 8 \(starting July 20th 00:00 UTC\), the community approved a [proposal to add a wrapFactor for soft pegged pairs](https://forum.balancer.finance/t/modifying-wrapfactor-applying-a-0-7-factor-to-soft-pegged-pairs/108). While for hard pegs, the factor is 0.1, for soft pegs it is 0.7 \(much less harsh\). A soft pegged pair is one in which the two assets are not directly convertible, but they do track the same underlying asset's price by design. Examples include a pair of USD stable coins \(e.g. DAI and USDC\) or a synthetic paired with its real-world asset \(e.g. sETH and WETH\).
* **UPDATED for week 8** \(starting July 20th 00:00 UTC\), multiply the pool liquidity by a `balFactor`. To incentivize liquidity of the native governance token, BAL, the community approved a [proposal to add a balFactor](https://forum.balancer.finance/t/balfactor-incentivizing-bal-liquidity-on-balancer/102/4). Deep liquidity of the BAL token ensures a more equitable distribution of governance power, by allowing new users to buy BAL with lower slippage, and discouraging existing holders from hoarding their tokens. The `balFactor` implementation applies a 2x multiplier to the BAL portion of each trading pair containing both BAL and one of the uncapped tokens                               \(`ETH, DAI, USDC, WBTC`\). This, combined with `ratioFactor`, creates a maximum possible total factor of ~1.54 for a pool containing 58% BAL and 42% of one of the uncapped tokens.
* **UPDATED for week 9** \(starting June 29th 00:00 UTC\), after the liquidity of all tokens is adjusted as usual by the currently active factors \(ratio, wrap, fee\), a `capFactor` has [been proposed](https://forum.balancer.finance/t/capfactor-capping-eligible-liquidity-to-10m-per-token/56), which is calculated such that every capped token is **limited to a maximum of $10M in adjusted liquidity**. `capFactor` is then applied to the liquidity of each affected capped token, resulting in an adjusted liquidity for every pool containing those tokens.`ETH, DAI, USDC, WBTC, BAL` is the list of uncapped tokens. Please read the proposal linked above for further details and examples.
* Calculate the proportional, adjusted, and capped \(see `capFactor` above\) liquidity USD value that each liquidity provider has in the pool. The table below shows an example for a pool that has 100$ worth of liquidity \(already adjusted by all factors\):

![](https://miro.medium.com/max/1472/1*2EM2KXgvt48qVK8FKQRmcw@2x.png)

* Divide the weekly amount of BALs distributed by the number of snapshot blocks. Considering blocks lasting 15s, a week would have a total of 40,320 blocks \(=7\*24\*60\*60/15\). Of these, there would be 158 snapshot blocks \(=40,320/256\). With 145,000 BAL distributed per week, the number of BAL distributed per snapshot block would be approximately 918 \(=145,000/158\).
* For each snapshot block, calculate the number of BAL tokens allocated to each address. This is calculated for each address proportional to the total liquidity of that account \(considering all pools they've contributed to\), divided by the total protocol liquidity. The table below shows an example of the final distribution for a snapshot block.

![](https://miro.medium.com/max/1492/1*MvfWrMI2PovCLJiwaQr6EQ@2x.png)

All the calculations described above depend exclusively on on-chain data and historical token prices openly accessible on CoinGecko. This whole calculation process is fully auditable via [an open source script](https://github.com/balancer-labs), and BAL distribution results will be posted to IPFS and referenced on Balancer’s [twitter account](https://twitter.com/BalancerLabs).

## Token Whitelist and Eligible Pools for Liquidity Mining <a id="84fc"></a>

**UPDATED for week 9** \(starting June 29th 00:00 UTC\): All tokens present in the whitelist created by the community \(see whitelist proposal\) are eligible for BAL liquidity mining. The [most up to date list](https://github.com/balancer-labs/assets/blob/master/lists/eligible.json) is maintained on Balancer's Github. All tokens listed under the `"homestead"` list in the json file linked are eligible. This list will evolve over time with input from the community.

Only Balancer pools containing two or more whitelisted tokens will be eligible for BAL liquidity mining.

## Liquidity Mining and Smart Pools

**UPDATED for week 21** \(starting October 19th 00:00 UTC\): Starting this week, BAL earned by LPs of [Smart Pools](../../smart-contracts/configurable-rights-pool.md) eligible for liquidity mining will be distributed directly to the LPs.
