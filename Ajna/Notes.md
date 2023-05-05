# R.E.A.C.H - Ajna Protocol Notes
# @DedOhwale

*Disclaimer: Some of the @audit notes may be out of scope. I wanted to include them, just in case there is a way to use in-scope contracts to attack.*

## Whitepaper Notes

### What is Ajna protocol?

- Ajna facilitates peer-to-peer loans without centralized governance & external price feeds.
    - *Permissionless pool creation* - AMM functionality
        - @audit potential scam/malicious pools should be distinguished on the front-end
    - *Price specified lending* - Replaces oracles with lenders inputing the price at which they're willing to lend
        - Price is the amount of the underlying token they are willing to lend per unit of collateral
            - E.g., if a lender deposits at price 100, they are willing to lend 100 units of the underlying token per one unit of collateral
            - In the worst case scenario, a lender should receive collateral at the price they lent to the pool in lieu of their underlying token deposit
            - @audit Make sure this occurs, and that a lender cannot receive less collateral than the lent amount
            - @audit Malicious actors could manipulate the lending pool by specifying inaccurate prices
    - *Market-derived interest rates* - Deterministic rules to set interest rates based upon user actions in the pool
        - It is **assumed** that the aggregate average loan collateralization( *pool collateralization* ) is an accurate indicator of the volatility profile of an asset
            - This is assumed as borrowers are naturally averse to liquidations
            - @audit This seems dangerous - can we manipulate a pool by borrowing in a very risky or risk-averse manner & abuse this functionality?
        - Ajna uses *pool collateralization* to inform the desired utilization ratio( *target utilization* )
            - The equilibrium between utilized & unutilized deposits
        - Looks at *meaningful actual utilization* - short term EMA of the ratio of debt to total lender deposits
            - Interest rates adjust based on these values to move the pool towards equilibrium
        - When *meaningful actual utilization* is less than *target utilization*:
            - There are more lenders than borrowers, and rates can be lowered
        - In the other direction, there is a shortage of lenders & rates are increased
        - @audit Risk that the market forces driving these rates could be manipulated or exploited (large lender artificially driving up rates)
        - @audit Flash loan attacks may exagerate the above potential vulnerabilities as it is possible to borrow large sums, manipulate the pool, and abuse
    - *Liquidation Bonds* - No actual liquidation will occur unless an external user triggers one
        - This is done by pledging a *liquidation bond* - a bet on the outcome of a pay-as-bid dutch auction for collateral
            - @audit can this introduce inefficiencies in the liquidation process? Collusion among bidders to drive down prices during an auction?
        - No incentive to spuriously liquidate a borrower in Ajna, disincentivizes liquidations of loans that are well collateralized with respect to the market price 
        - @audit make sure this is true - can we benefit from **griefing** well collateralized loans?
    - Ultimately, a user can **lend, borrow, or trade.**


### The Pool

- The pool consists of:
    - A fungible quote(underlying) token & fungible collateral token
    - A NFT collection pool consists of a fungible quote token & an NFT collection
        - All NFTs from that collection are accepted as the collateral token
    - An NFT subset pool, consisting of a fungible quote token & a selected subset of NFTs (from one or many collections) as the collateral token
        - @audit Malicious or scam NFT subset pools need to be considered (think AAA mortgage loans from 2008 financial collapse, bundling fraudulent crap with a set of good assets).
- The collateral token is the numerator, quote token is the denominator
    - The factory contract only permits the creation of one pool per unique pairing (not including the NFT subset pool)
        - E.g., only one ETH/DAI pool where DAI is lent out against ETH, and only one DAI/ETH pool.
        - @audit Make sure this happens & that we cannot create multiple pools of the same type (this would exaggerate all of the potential vulnerabilities listed above).
        - @audit Make sure the factory contract is secure & properly validates input parameters for creating a pool.

### Pool Structure

- Contains an index of prices - discretized into *price buckets*
    - Spacing & bucket range chosen to be inclusive of a wide range of asset prices - 7388 price buckets, spaced 0.5% apart with 1 being the price of bucket 3232
        - 3232 price buckkets below 1, 4155 price buckets above 1
        - Price = 1.005^n where n is the bucket index of the price bucket being calculated
        - A price bucket consists of:
            - Quote token (deposit)
            - Claimable collateral
            - Total liquidation provider balance (Total LPB)
    - @audit A specific bucket choice & spacing may potentially lead to situations where lenders or borrowers cannot find a suitable range and experience slippage
    - @audit Front-running could be lucrative here, with a malicious actor front-running transactions to take advantage of price discrepencies between the buckets
- A *deposit* is when a user provides liquidity to a pool, specifying a price bucket.
    - This is a liability of the contract
- Users may also place collateral into a bucket via a trade or as a result of a liquidation auction
    - Each bucket maintains a *claimable collateral* account to reflect the presence of this collateral in the bucket
    - Collateral is exchangable with quote tokkens at the price of the buckket
    - A record is kept of which portion of the contents of the buckets is owed to each liquidity provider
        - E.g., if there are two depositors in a bucket, and each contributes 100 quote tokens, each is entitled to 50% of the contents of the bucket (quote token, collateral, or both).
        - Users redeeming *LPB* have the choice of which form to withdraw, with the price of the bucket itself being the rate of exchange between deposit & claimable collateral
- Exchange rates with *p* being price, *QT* quote token, *CT* collateral token, *TotalLPB* total liquidity provider balance:
    - $\frac{LPB_p}{QT} = \frac{deposit_p+p*claimableCollateral_p}{TotalLPB_p}$
    - $\frac{LPB_p}{CT} = \frac{deposit_p + p*claimableCollateral_p}{p*TotalLPB_p}$
    - If a bucket is empty, these equations yield 0/0 & the exchange rate could be initialized to anything. In this case, the exchange rate between *LPB* and quote token *LPB_p/QT* is set to 1.0
        - E.g., Alice deposits 50 DAI into an empty pool, and is issues 50 LPB, as the exchange rate is 1.0
        - @audit Can we do something similar to H-01 within our EigenLayer audit?
        - @audit There are way too many things depending on the total assets in the pool, this leads to price discrepancy & arbitrage opportunity
        - @audit If there is a discrepancy between the deposit, claimable collateral, and total LPB values - users could receeive more or less than they should when withdrawing or exchanging LPB
        - @audit Front-running can be a big issue with this type of math

### Loans

- When a user pledges collateral & borrows quote token, their debt is a *loan*
    - A user may add or withdraw collateral at any time, unless it would leave their loan insufficiently collateralized
        - @audit Can we bypass this and withdraw collateral while maintaining the loan?
- Debt/collateral pledged is the loan's *threshold price, TP* or the price at which the value of the collateral equals the value of the debt
    - This value, *TP*, is used extensively in the system.
    - @audit Make sure this is flawless
- Loan creation triggers an origination fee - immediately increases the borrowers debt
    - Further, lenders are faced with deposit penalties under certain conditions
    - The pool collects *net interest margin, NIM* from the interest collected on loans
        - E.g., Bob wants to borrow DAI with his ETH. He places 20 ETH in the ETH/DAI contract and withdraws 18000DAI. His debt is 18000 & collateral pledged is 20. His TP is 18000/20=900 (this is ignoring fees)

### Reserves

- Origination fees, deposit penalties, and net interest margin( *NIM* ) on loans are accumulated within the pool's reserves
    - The reserves are mainly used to buy and burn AJNA tokens
        -@audit make sure this is implemented correctly - is this front-runnable? Can we reliably know when they will be buying or burning?

### Lending

- Lenders deposit quote tokens & are issued LPB in a specific bucket
    - LPB can be received in the form of an NFT which represents a transferable version of their LPB in the pool
- High priced buckets offer highest valuations on collateral, and are thus the first source of liquidity to borrowers
    - First buckets to purchase collateral if a loan were to be liquidated
        - A buckets deposit is *utilized* if the sum of all deposits is priced higher than the total debt of all borrowers in the pool
            - Lowest utilized price *LUP* is the lowest price among utilized buckets
            - If we match highest priced lenders deposits with borrowers debt in equal quantities, *LUP* would be the price of the marginal lender (lowest priced & least aggressive)
            - @audit *LUP* plays a critical role in Ajna - a borrower who is undercollateralizzzed with respect to the *LUP* is eligible for liquidation. Make sure this is done right.
- In order to remove a deposit - the user must ensure that this action does not cause *LUP* to move below *highest threshold price, HTP* - threshold price of the least collateralized loan
    - In this event, the user would need to liquidate the loan(s) that sit between *LUP* and *HTP*
    - @audit Is this process automated? If not, what can happen if we try to execute this type of transaction?
- If there are active liquidations in thhe pool, deposits that are withing `liquidation_debt` of the top of the book cannot withdraw until liquidations are complete. Deposits are frozen.
    - @audit Again, reminiscent of EigenLayer, H-02 this time. Can we skirt around the frozen mechanisms.
- Lenders are subject to a small fee for making a deposit below the *LUP* - mitigation of MEV attacks on the pool. The unutilized deposit fees go to the reserves.
    - @audit They are worried about MEV attacks here - lets find out why and see if this mitigation is enough
- The lender/bucket accounting is very gas intensive with the following four operations:
    - Evaluate total deposit over a range of buckets (used for evaluation of how much deposit is in a specific bucket or above the *HTP* for lender interest calculations)
    - Searching for the highest price that has at least a given amount of deposit above it ( *LUP* calculations)
    - Incrementing/decrementing deposit quantity in a bucket (for lending or withdrawing)
    - Multiply all deposits in a range by scalar quantity(accruing interest to lenders)
    - @audit This seems open to gas optimizations, and gas attacks.

### Borrow

- A borrower may takke out a loan by pledging collateral to a pool
    - An NFT must be pledged in its entirety - no fractionalizations
    - If debt is less than collateral, it is fully collateralized
- Each borrower is subject to interest rate penalties on their debt, at the same rate for each lender
    - A global inflator variable is tracked by looking at total debt
    - @audit Make sure this works correctly, can we make *faux* pools and impact the global inflator?
- Interest rates change dynamically based on *meaningful actual utilization, MAU* and *target utilization, TU* in response to market forces
- When borrowing quote token, a borrower is subject to an origination fee - greater of the current annualized borrower interest rate divided by 52 (on week of interest) or 5bps
- The minimum borrow size is 10% of the average loan size - This check is only enforced when the number of loans exceeds 10

### Liquidation

- Loans which are not fully collateralized wrt the *LUP* are eligible for liquidation.
    - `collateral_loan * LUP < debt_loan`
- Actors calling the liquidate method are *kickers* & required to post a liquidation bond
    - Quantity of quote token that, depending on the results of the auction, could either earn additional quote token as a reward, or be forfeited in whole or in part as a penalty
    - @audit Make sure this is done fairly
- Once the method has been successfully called, the loan has its debt increased by 90 days of interest
- There is a grace period of 1 hour to recapitalize or pay back the loan
    - Afterward, anyonne can purchase portions of the collateral via a pay-as-bid dutch auction
- To disincentivize malicious liquidations - a percentage of the debt of the loan must be posted
    - If the liquidation auction yields a value over the *neutral price, NP*, the kicker forfeits a portion or all of their bond
        - @audit Can this be front-ran? E.g., Alice has a value under NP, Bob calls liquidate, Alice sees this & front-runs collateralization, Bob forfeits his bond

### Grants

- Premised upon the idea that AJNA token has value from the buy & burn mechanism
    - Upon launch, the community will be given a fixed portion of the total supply
        - This will be held & distributed by the *grant coordination fund, GCF*
            - Emits tokens through primary funding & extraordinary funding mechanisms
- Ajna also permits vote delegation & delegate rewards
    - A tokenholder may delegate to themselves if they wish to capture this reward
    - In the primary funding mechanism, 10% of the quarterly distribution is awarded to delegates. The remaining 90% goes through the *PFM* and distributes
        - Delegate rewards do not apply to the *EFM*

#### Primary Funding Mechanism(PFM)

- Up to 2% of the treasury (30% of the AJNA token supply on launch) is distributed
    - Projects submit proposals for funding, denominated in AJNA tokens
    - Ajna holders vote on the proposal - either a win & is funded or fails and receives nothing, binary
    - Voting system has two stages:
        1. Screening stage - list of possible winning proposals is culled down to 10 candidates using a 1-token 1-vote method
            - @audit Evaluate the implementation, ensure the top 10 proposals are selected correctly & passed to the funding stage
        2. Funding stage - quadratic voting is used to determine which proposals are funded once they have made it through the screening stage
            - @audit Double check the quadratic voting implementation, verifying that the constraint provided is properly enforced & the system remains resistant to Sybil attacks
    - Once the voting period ends - the winning proposals that can be executed are decided through a one week challenge period.
        - Anyone can submit a set of winning proposals to execute
        - If it is more optimal than the previously submitted slate of proposals, then it becomes the new winning slate
        - @audit Make sure this is adhered to

Here is the process according to the whitepaper:

1. Each quarter (90 days), up to a 2% of the treasury can be distributed to projects that win a
competitive bidding process5. This is the global budgetary constraint, GBC.
a. The following formula allows us to calculate the amount of AJNA tokens that
may be distributed per quarter. “Amount Distributable,” D, is the number of
AJNA tokens that may be distributed in that cycle, GBC is percentage referenced
above, and “Amount Treasury,” T, is the number of AJNA tokens remaining in
the treasury contract at the time of the distribution.

$amount_D = GBC*amount_T$

2. Any project can submit a proposal. A proposal consists of a budget together with the
address to send the tokens to. A budget is simply a quantity of Ajna tokens requested.
Proposals can include multiple different addresses, with variable portions of the total
proposal budget.

3. Voting on proposals is conducted by voters, with a voter’s voting power determined by
the number of tokens delegated. Ajna token holders can delegate their voting power to
themselves, or to any other address of their choosing.

4. To avoid an overwhelming number of proposals, the slate of projects is filtered down to
10 projects during a screening stage. Voting power in the screening stage is based upon a
snapshot of an address' voting power 33 blocks prior to the screening stage’s start block,
where one token is equal to one vote. Votes can be split across an arbitrary number of
proposals, and voters can only vote once in the screening stage. For example, if a holder
has 100 AJNA they may vote 10 AJNA on 10 proposals, 50 AJNA on 2 proposals, or 100
AJNA on 1 proposal, etc... The screening stage lasts for the first 80 days of the cycle.
At the end of the screening stage, the 10 proposals with the most support are deemed
eligible to be funded.

5. These 10 projects are then voted upon in the funding stage. In the funding stage, Ajna
token holders vote on the proposals using a quadratic system. Each address with voting
power from Ajna tokens can vote as much as they want on every proposal, including
negative votes, subject to the constraint that the sum of the squares of their votes cannot
exceed the square of the number of Ajna tokens delegated to the voter:

$\sum_{i=1}^10 \gamma_{i, addr}^2 \leq A_{addr}^2$

where $A_{addr}$ is the number of Ajna tokens delegated to an address addr, and $\gamma_{i,addr}$ is the 
vote by address addr on proposal i. Voting power in the funding stage is based upon a
snapshot of an address's delegated token balance 33 blocks prior to the funding stage’s
start block. Note that this quadratic system is not vulnerable to sybil attacks, as splitting
tokens across multiple addresses results in a corresponding decrease in the “vote budget”
of the voter.

6. Once the voting period ends, the winning proposals that can be executed are decided
through a one week challenge period. In the challenge period, anyone can submit a set of
winning proposals to execute. If it is more optimal than the previously submitted slate of
proposals, then it becomes the new winning slate. The optimal proposal slate maximizes
the sum of the net number of votes cast in favor of the proposals, subject to the sum of
the proposals’ budgets must not exceed the GBC. Example:
a. There are 3 proposals that make it through screening and funding stages: proposal
1, or proposals 2+3 combined can make it through the GBC, not all 3 together.
Since 1 has more votes than 2+3, 1 should be selected and funded but not 2+3
even though they were also theoretica

#### Extraordinary Funding Mechanism(EFM)

- The PFM only allows for a small fraction of the treasury to be distributed on a fixed schedule
    - EFM was made in case more capital is needed for funding
    - By meeting a certain quorum of non-treasury tokens, tokenholders may take tokens from the treasury outside of the PFM
    - @audit Ensure it handles voting, token distribution, minimum threshold increases, and proposal requirements correctly
    - @audit Confirm that the EFM votihng period is correctly set by the proposer & adheres to the max duration of one month

Here is the process taken from the whitepaper:

1. By meeting a certain quorum of non-treasury tokens, tokenholders may take tokens from
the treasury outside of the PFM

2. Voting occurs similarly to the PFM, with token owners able to delegate to themselves or
others, and one token equaling one vote based upon delegated voting power at a snapshot
33 blocks prior to the EFM proposal’s start. In the EFM, votes can only be cast in support
of the proposal, with voters only able to vote on a proposal once.

3. This mechanism works by allowing up to the percentage of non-treasury tokens,
minimum threshold, that vote affirmatively to be removed from the treasury – the cap on
this mechanism is therefore 100% minus the minimum threshold (50% in this case).
Examples:
a. If 51% of non-treasury tokens vote affirmatively for a proposal, up to 1% of the
treasury may be withdrawn by the proposal
b. If 65% of non-treasury tokens vote affirmatively for a proposal, up to 15% of the
treasury may be withdrawn by the proposal
c. If 50% or less of non-treasury tokens vote affirmatively for a proposal, 0% of the
treasury may be withdrawn by the proposal

4. When submitting a proposal the proposer must include the exact number of tokens they
would like to extract, proposal threshold. This proposal threshold is added to the
minimum threshold to provide a required votes received threshold. If the vote fails to
reach this threshold it will fail and no tokens will be distributed. Example:
a. A proposer requests tokens equivalent to 10% of the treasury
i. 50% + 10% = 60%
ii. If 65% of non-treasury tokens vote affirmatively, 10% of the treasury is
released
iii. If 59.9% of non-treasury tokens vote affirmatively, 0% of the treasury is
released

5. Also, a proposal must not exceed the minimum threshold in what it withdraws. For
example, if a proposal specifies that it would like to take 100,000,000 tokens from the
treasury, which represents 60% of the treasury supply, and the minimum threshold is
55%, this proposal will automatically fail.
EF1 requests: 40% (120,000,000)
EF2 requests: 40% (120,000,000)
Minimum threshold is 50% at time both proposals are submitted
EF1 passes
- Minimum threshold: 55%
- treasury: 180,000,000
EF2 should now fail since `120,000,000 < ((1 - minimum threshold) * treasury)`
- 120,00,000 < 81,000,000

6. Each time the emergency mechanism is successfully used the minimum threshold is
increased by 5%, meaning it can only be used 9 times in total and becomes successively
more difficult to use. Example:
a. A proposal is placed and meets its threshold, the minimum threshold was 50%
b. The next proposal will have a minimum threshold of 55%, and the following will
have a minimum threshold of 60%

7. The voting period for the EFM is set by the proposer and has a maximum duration of 1
month
