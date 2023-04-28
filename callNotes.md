# Eigenlayer Call Notes

Conducted 2 audits - **Eigenvalue Diligence & Sigma Prime**

Only 1 medium severity & 1 high severity found in both of them.

## Areas of Focus:

### General

- Loss of user funds
- Enforced delays on withdraw

### Beacon Chain, Execution Layer

- Eigenpods - connects beacon chain to execution layer for "native restaking" of ETH. 
   - Existing/new ETH validators who don't hold liquid staked tokens, but just directly stake eth on Beacon Chain can restake on eigenlayer. Pointing withdraw credentials to one of the eigenpod contracts. Oracle later, out of scope. 
   - Still, lots of functionality in eigenpods to check merkalized proofs against the oracle state roots - want assurance that they've done this correctly. 


## Restaking

Restaking is the process of taking a staked asset, and staking it again. E.g., a user may stake ETH for stETH and then stake stETH within EigenLayer.

- Restaked assets are controlled by EigenLayer
   - This can be used by rollups, bridges, etc.
        - Essentially acts as a backing of collateral.

Slashing can occur when a user is acting maliciously, enabling services to slash some of the users deposited funds.

- Eigenlayer is permissionless
    - Anyone can be a staker, consume a service, or create their own service
    - Being an operator for a service is **opt-in**.
        - Being a staker is **opt-in** as well, and can commit to choosen services (1 or more)
            Stakers may also delegate their stake to an operator that they trust.

- New services built on EigenLayer can determine their own slashing conditions
    - On-chain verifiable conditions

## Actors
**Stakers** are the party with the asset. Mix of ERC20 tokens, ETH, etc.

- \textcolor{red}{@audit} We need to check for:
    - Correctness with asset depositing
        - Contract handles staked deposits correctly
    - Delegation mechanism
        - Secure, intended functionality

**Operators** are the users who run the software built on EigenLayer. Stakers may delegate their assets to operators, which selects certain services to service.

- \textcolor{red}{@audit} We need to check for:
    - Registration is non-tamperable
        - Someone else cannot register you
        - Cannot take anothers registration
    - opt-in secure
        - Access control
        - Cannot opt someone else in
    - Ensure that the slashing is done correctly
        - Fairly done
        - Implemented as intended by the services

**Watchers- Future implementation** parties reliable for observing "rolled-up" claims (Not supported yet) and step in to invalidate a false claim. Essentially the operators supervisor.

- \textcolor{red}{@audit} We need to check for:
    - Fraudproof period enforcement
        - Allow enough time for watcher to disprove claims
    - Punishment mechanisms
        - Fair, effective, restrictive

In summary, a staker delegates their stake to an operator, who then allocates the assets to secure one or more services. Each service defines tasks, which represent time-bound units of work or commitment, during which the operator's stake is at risk. Operators are responsible for fulfilling their obligations in each task, ensuring that they adhere to the requirements and conditions set by the service.

## Assumptions
**Discretization of tasks** Assuming the service will be in charge of discretizing the tasks.

- Check the task periods, make sure there are no unfair advantages.

**Delegation "trust network" structure** Assuming that stakers are delegating their assets to operators that they trust well. Operators will be able to steal all of the funds.

**Noncompromise of trusted roles** Assuming all trusted roles (multisig, etc.) will be trustworthy.

**Honest Watcher Assumption** Assuming at least one honest watcher to fraudproof all false claims.

## Contract Overview
### `StrategyManager`

`StrategyManager` is the primary contract - this is what users will be sending their tokens to. It then sends funds to the `Strategy` contracts, which may then move the assets outside of Eigenlayer to earn returns.

- Examining the interactions between the StrategyManager and Strategy contracts to ensure secure and accurate transfer of restaked assets. 
- Ensuring that the strategies do not introduce additional risks or vulnerabilities to the system.

Withdraws and undelegations will go through the `StrategyManager` contract.

- Make sure there are delays!
    - funds "at stake" cannot be undelegated or withdrawn
    - We cannot know immediately if funds are at stake, thus delay
    - Users enter the queued withdrawal process
        - Begin the withdrawal, signaling not to be placed "at stake"
            - \textcolor{red}{@audit} Make sure this happens! There should be a flag or something.
        - Push updates to services (or have their operator do it)
        - Complete withdrawal after delay
            - \textcolor{red}{@audit} Is this delay enough? Do we have to push updates to the services?
            - \textcolor{red}{@audit} Is it possible to begin the withdrawal, not push updates, and withdraw the funds before getting slashed?

### `DelegationManager`

`DelegationManager` will handle if stakers register to be operators or if they will delegate their stake to another operator.

- Examine proper tracking of delegated assets, accurate assignment of tasks, and appropriate slashing conditions.
- Funds earned may be sent to a `DelegationTerms`-type contract (or EOA)
    - Helps mediate the relationships between staker & operator
    - \textcolor{red}{@audit} Check if this is a requirement & implemented correctly:
        - If it is not EOA, ensure delegation terms are conducted properly
            - Tracking and management of delegation relationships
            - Proper handling of payment & mediation
        - If it is EOA, is that handled properly?

- `DelegationManager` works closely with `StrategyManager`.
    - Keeps track of all operators (\textcolor{red}{@audit} make sure this is done correctly)
        - Storing the delegation terms for each operator
        - Stores what operator each staker is delegated to
    - Staker becomes operator **irrevocably**
        - Operator is defined as `delegationTerms[operator]` not returning 0 address
            - \textcolor{red}{@audit} Check if this is done, lost funds if not as we cannot change
            - \textcolor{red}{@audit} Can we change another users return?
    - Undelegation needs a delay or clawback mechanism
        - \textcolor{red}{@audit} Again, check to make sure that this is implemented properly
        - \textcolor{red}{@audit} Can we find a way to undelegate while funds are staked?

### `Strategy`

`Strategy` contracts each manage a single ERC20 token. Each users holdings in the strategy should be reflected by how many shares they have.

- `Strategy` is in charge of defining methods of converting from `underlyingToken` and shares(and vice versa).
    - Make sure this works, and that the shares match the `underlyingToken` conversions
- Assets 'may' be depositable & withdrawable in multiple forms
    - \textcolor{red}{@audit} Check that only `StrategyManager` can conduct deposits & withdraws
- 'may' be passive or active with funds

### `Slasher`

`Slasher` is the central point for slashing. Operators opt-in to being slashed by arbitrary contracts by calling `allowToSlash`.

- \textcolor{red}{@audit} Make sure `allowToSlash` cannot be abused. Ensure proper access control
- A contract can revoke its slashing ability after `serveUntil` time
    - \textcolor{red}{@audit} Can this be tampered with? Over/underflow?
    - The time is stored in `contractCanSlashOperatorUntil[operator][contractAddress]`
        - \textcolor{red}{@audit} ensure this works correctly and that the order is correct

- Slashing is a multi-step process
    - First, you freeze the operator with `freezeOperator`
        - any `contractAddress` for which `contractCanSlashOperatorUntil[operator][contractAddress > 0`, can freeze the operator.
            - \textcolor{red}{@audit} check access control & not `>=` or any way to tamper with time
        - When an operator is frozen, and any staker delegated to them, cannot make new deposits or withdrawals, and cannot complete queued withdrawals.
            - \textcolor{red}{@audit} Can we find a way around this?
    - Then, the owner of the `StrategyManager` can call the slash and unfreeze
        -\textcolor{red}{@audit} Wouldn't this be the owner of the protocol? The services cannot slash?

### `EigenPodManager`

`EigenPodManager` handles the Beacon Chain ETH being staked on EigenLayer.

- Creates new `EigenPod` contracts and coordinates virtual deposits and withdrawals of shares in an enshrined `beaconChainETH` strategy to and from the `StrategyManager`.

### `EigenPod`

`EigenPod` is deployable by the stakers. Each staker can only deploy one pod. 

- Allows user to stake ETH on the Beacon Chain
    - Then, restake the deposits into EigenLayer
        - A watcher is in charge of making sure the values are honest
            - \textcolor{red}{@audit} Make sure that the watcher can intervene fairly and only in malicious scenarios

- Calls generally go from `EigenPod` -> `EigenPodManager` -> `StrategyManager` to trigger additional accounting knowledge within EigenLayer.
- \textcolor{red}{@audit} Ensure that all of these access controls are done correctly.
- `EigenPod` is deployed using beacon proxy pattern
    - Allows simultaneous upgrades of all EigenPods.
        - \textcolor{red}{@audit} Can we have a pod not be upgraded?
