# phat-offchain-rollup
Phat Contract Offchain Rollup implementation

## Highlighted TODOs

- Simple scheduler
- Lock: the correct implementation; may require a rewrite
- Optimization: Batch read from rollup anchor

## Phat Contract side

### Misc

- [x] Refactor experimantal code as contracts
    - [x] Switch to OpenBrush's `ink-env` with the advanced unit test kits

### Contract

- [x] SimpleScheduler
    - [x] [Design](https://hackmd.io/vl7oVbUlQmW8a_rcxhk9JQ)
    - [x] Query `poll()`
        - call all the ready targets
        - should check health (trigger exactly one health worker)
    - [x] Tx `register(config, address, calldata)` owner-only (direct call, stateless)
    - [x] Tx `delete(id)` owner-only
    - [x] log the triggered events
- [ ] RollupTransactor
    - [x] Account management: generate secret key & reveal public key
    - [x] Tx `config(rpc, rollup_handler, anchor)` by owner
    - [x] Query `poll()`
        - get `Result<RollupResult, Vec<u8>>` response
        - submit tx to `RollupResult.target_id`
            - use the latest nonce
            - fire and forget
    - [x] enum RollupTarget
        - EVM(chain, address)
        - Pallet(chain)
    - [x] Raw tx submit
    - [ ] Gas efficiency submit
            - for gas efficiency, save the recent submitted tx to local storage (with timeout) to avoid redundant submission in a short period
- [ ] TestOracle
    - [x] Minimum implementation
    - [ ] Real-time fetch price
    - [x] Refactor to strip SDK logic

### SDK

- [ ] Locks
    - [x] Experimental lock tree (tx_read, tx_write)
    - [ ] Correct implementation
- [x] struct RollupTx
    - [x] Condition
    - [x] Updates
    - [x] Actions
- [x] struct RollupResult
    - [x] RollupTx
    - [x] RollupTarget
    - [ ] (opt) signature of RollupTx
- [ ] RollupReadClient
    - [x] Read from EVM
    - [x] Wrap as QueuedAnchorClient
    - [ ] Consider block_hash when reading data
    - [ ] Cross validation
- [ ] (low) Cross-platform Rollup
    - [x] Basic codec abstraction (`platform::Platform`)
    - [ ] State reading abstraction
    - [ ] RollupTx serialization

## Development Notes

- To run Phat Contract unit tests, you need to configure `.env` file under each contract directory. It's suggested to copy `.env_sample` to `.env` and config it based on the comments.
- `abi.decode()` doesn't have any error handling currently. When it failed, the transaction will get revereted silently, which is hard to debug. So it's always a good habit to verify the raw input to `decode()` beforehand.
- In unit tests, you can enable logging output (e.g. from `pink::info!()`) by:
    1. Add `env_logger` as the `[dev-dependency]` in the Cargo.toml file
    2. Call `env_logger::try_init()` at the beginning of each test function
    3. Launch the unit test with additional flags: `RUST_LOG=debug cargo test -- --nocapture`

## Manual Full Test

1. Compile and deploy `SampleOracle` and `EvmTransactor` with their `default()` constructor
    - This can be done with the [Phat Contract UI](https://phat.phala.network)
    - The caller wallet will be the owner of the two contracts.
2. Query `EvmTransactor::wallet()` to get the generated EVM wallet address, and deposit some test ETH to the wallet for gas fee. This account will be used to send rollup transaction from Phat Contract to EVM.
3. Deploy the EVM contracts (`PhatQueuedAnchor` and `TestOracle`) under `evm/contracts`.
    - Deploy with hardhat: `npx hardhat run scripts/deploy-test.ts --network <network>`
    - Please replace `phatEvmTransactor` in `deploy-test.ts` with the wallet you get from step 2
    - The script will output the address of the deployed contracts
4. Config `SampleOracle` and `EvmTransactor`
    - Can be don with [Phat Contract UI](https://phat.phala.network)
    - Config `SampleOracle`: give it the EVM RPC address (e.g. Alchemy Goerli RPC) and the EVM `PhatQueuedAnchor` address
    - Config `EvmTransactor`: give it the EVM RPC address, the Phat Contract `SampleOracle` address (`rollup_handler`), and the EVM `PhatQueuedAnchor` address
    - The [E2E test script](./phat/tests/e2e.test.ts) is an equivalent example
4. Call the EVM contract `TestOracle.request()` to initiate a request
    - [Example](https://goerli.etherscan.io/tx/0x2bd59af64763c8a330698709b34bd8ce70f4a7ce9c5505f855978080a8fa9597)
5. Call Phat Contract `EvmTransactor::poll()` to answer the request
    - It should results in an rollup transaction like [this](https://goerli.etherscan.io/tx/0x888a84c2964b9eac7923f9daa59446e12a9d93414fe63a964004de515bab9f02)

## Deploy with scheduler

1. Follow the "Manual Full Test" section to setup `SampleOracle`, `EvmTransactor`, and the EVM contracts.
2. Deploy `LocalScheduler` with `default()` constructor
3. Call `LocalScheduler::add_job()` to add the automation job to trigger `EvmTransactor::poll()`
    - `name`: an arbitrary name
    - `cron_expr`: a cron-like expression of the automation. `* * * * *` means trigger per minute.
    - `target`: the address of `EvmTransactor`
    - `call`: the encoded call data of `EvmTransactor::poll()`, which is `0x1e44dfc6`
4. Call `poll()` repeatedly. It will trigger the job according to the cron expression.
5. You can check `LocalScheduler::getJobSchedule(n)` to check the scheduled task
    - `n` is the job id, starting from 0.
    - Tasks are scheduled after the first `poll()`. So a newly added task will have `None` output until the first `poll()`.
6. The scheduler prints logs to indicate how it trigger the events. Can read it from the UI.

## Phat Contract E2E test caveats

The [E2E test script](./phat/tests/e2e.test.ts) covers the Phat Contract deployment and rollup test. To run the test, you will need to copy `phat/.env_sample` to `phat/.env` and configure it before running the test.

However, we face a circular dependency problem here: The test requires the `PhatQueuedAnchor` is already deployed on the EVM side. The EVM anchor contract should only accept the transaction from the Phat Conract. So in the manual full test process, we must generate the EvmTransactor wallet (the wallet generated by the Phat Contract interanally) first. Then when constructing the `PhatQueuedAnchor`, we must specify the EvmTransactor wallet in the constructor arguments. However, the probem in Phat Contract E2E test is that, we don't know the wallet before the EvmTransactor is deployed.

In the test we walked around it. First, the EVM contracts should be deployed before running the E2E test. When constructing the `PhatQueueAnchor`, we set a developer owned wallet as the mock EvmTransactor wallet. Then when running the E2E test, instead of using the wallet generated by EvmTransactor, we inject the mock wallet key to the EvmTransactor by `testPollWithKey`. So we break the circular dependency.