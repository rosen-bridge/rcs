# Add New Network API for a Chain in Rosen Bridge
This document outlines the steps to add a new network API to an existing chain in Rosen Bridge.

## Contents
- [Introduction](#introduction)
- [Watcher Service Integration](#watcher-service-integration)
    - [Scanner](#scanner)
    - [Rosen Extractor](#rosen-extractor)
    - [Observation Extractor](#observation-extractor)
    - [Scanner Sync Check](#scanner-sync-check)
- [Guard Service Integration](#guard-service-integration)
    - [Abstract Chain Network](#abstract-chain-network)
    - [Asset Check](#asset-check)

## Introduction
Adding a new network API is mainly about Watcher and Guard services. This assumes the blockchain is already supported in Rosen Bridge with at least one existing network API. The requirements for each service are different, and this document explains the steps to implement the required modules. Please refer to each module's documentation for details.

> Note: This document is for adding a new network (e.g., a new explorer API) to an existing chain that already has at least one network API. Please refer to [the main document](./main.md) for implementing the first network API. 

> Note: The naming convention is described in each section. `ChainX` represents the new blockchain name and `Api` represents the network API that is used. Examples for Bitcoin blockchain and RPC API:
  - `@rosen-bridge/chainx-api-scanner` -> `@rosen-bridge/bitcoin-rpc-scanner`
  - `ChainXChain` -> `BitcoinChain`
  - `ChainXApiNetwork` -> `BitcoinRpcNetwork`

## Watcher Service Integration
Supporting a new network API in the Watcher service requires specific implementation in four modules.

### Scanner
Steps to implement a General Scanner for the new network API:

1. Add a new package to the [Scanner repository](https://github.com/rosen-bridge/scanner).

    - initialize the package using `kodegen`:
      ```bash
      npx kodegen monorepo add-package
      ```
    - set package name as `@rosen-bridge/chainx-scanner` (e.g., a scanner for Bitcoin based on RPC API will be `@rosen-bridge/bitcoin-scanner`)
    - set package path as `./packages/scanners/chainx-scanner`
    - set description as `A Chain-X blockchain scanner worked with network API` (replace `api` with the corresponding network such as RPC, Explorer, GraphQL, etc.)
    - set package repo url as `https://github.com/rosen-bridge/scanner`
    - enable both features
      - `Prettier and Eslint`
      - `Testing (with coverage support)`

2. Implement a class to interact with the blockchain. It should inherit from the `AbstractNetworkConnector` class (refer to the [`BitcoinRpcNetwork` implementation](https://github.com/rosen-bridge/scanner/blob/dev/packages/scanners/bitcoin-scanner/lib/scanner/BitcoinRpcNetwork.ts) for example).

    - name convention: `ChainXApiNetwork`

3. Implement the scanner class. It should inherit from the `GeneralScanner` class (refer to the [`BitcoinRpcScanner` implementation](https://github.com/rosen-bridge/scanner/blob/dev/packages/scanners/bitcoin-scanner/lib/scanner/BitcoinRpcScanner.ts) for example). Note that to ensure consistent implementation across all scanners, each scanner starts from `initialHeight + 1`.

    - name convention: `ChainXApiScanner`

4. Implement unit tests for all functions of the network class. Note that no real request should be sent in the tests and the connector should be completely mocked.
    - for tests, refer to [`BitcoinRpcNetwork` tests](https://github.com/rosen-bridge/scanner/blob/dev/packages/scanners/bitcoin-scanner/tests/network/BitcoinRpcNetwork.spec.ts)
    - mocking depends on the network connector. For mocking `axios` refer to [`axios.mock.ts`](https://github.com/rosen-bridge/scanner/blob/dev/packages/scanners/bitcoin-scanner/tests/mocked/axiosRpc.mock.ts) in the `bitcoin-scanner` tests. For mocking classes, such as the `ethers.JsonRpcProvider`, refer to [`JsonRpcProvider.mock.ts`](https://github.com/rosen-bridge/scanner/blob/dev/packages/scanners/evm-scanner/tests/mocked/JsonRpcProvider.mock.ts) in the `evm-scanner` tests.

### Rosen Extractor
Since the Rosen Extractor is responsible for extracting bridge request information, all watchers regardless of the network API should reach the same Rosen Data. Therefore, all Rosen Extractors for a chain should be consistent. For example, while the Esplora network returns transaction input addresses, this information is not available through the RPC API. In such scenarios, either the network cannot be supported, or the data extraction approach must be modified across all Rosen Extractors for that chain.

While the Rosen Extractor argument type should be exactly the same as its corresponding scanner, processing the exact type directly is not always necessary. If the transaction can be converted to another transaction type that already has a Rosen Extractor, and the conversion does not significantly reduce processing speed, you can convert to the desired type and use the existing Rosen Extractor instead of implementing from scratch.

Steps to implement a Rosen Extractor from scratch for the new network API:

1. A directory should already exist for ChainX in the `rosen-extractor` package in the [Utils repository](https://github.com/rosen-bridge/utils/tree/dev/packages/rosen-extractor).

2. Implement the Rosen Extractor class. It should inherit from the `AbstractRosenDataExtractor` class (refer to the [`BitcoinRpcRosenExtractor` implementation](https://github.com/rosen-bridge/utils/blob/dev/packages/rosen-extractor/lib/getRosenData/bitcoin/BitcoinRpcRosenExtractor.ts) for example).

    - name convention: `ChainXApiRosenExtractor`

    > **Important Note**: The generic type should be exactly the same as the one used in the corresponding scanner. The exact type should be defined in the package itself and it cannot be imported from the scanner package.

3. Implement unit tests for all scenarios (refer to [`BitcoinRpcRosenExtractor` tests](https://github.com/rosen-bridge/utils/blob/dev/packages/rosen-extractor/tests/getRosenData/bitcoin/BitcoinRpcRosenExtractor.spec.ts) for example)

Steps to implement a wrapper-style Rosen Extractor for the new network API:

1. A directory should already exist for ChainX in the `rosen-extractor` package in the [Utils repository](https://github.com/rosen-bridge/utils/tree/dev/packages/rosen-extractor).

2. Implement the Rosen Extractor class. It should still inherit from the `AbstractRosenDataExtractor` class while initializing another Rosen Extractor in the `constructor`. **Implementation is as simple as converting the transaction to the desired type, calling the `get` function of the other Rosen **Extractor and **returning** the result**** (refer to the [`EvmRosenExtractor` implementation](https://github.com/rosen-bridge/utils/blob/dev/packages/rosen-extractor/lib/getRosenData/evm/EvmRosenExtractor.ts) for example).

    - name convention: `ChainXApiRosenExtractor`

    > **Important Note**: The function should handle exceptions while converting the transaction to the desired type and return `undefined` if it failed to do so.

    > **Important Note**: The generic type should be exactly the same as the one used in the corresponding scanner. The exact type should be defined in the package itself and it cannot be imported from the scanner package.

3. Unlike implementing from scratch, this style only needs tests for three scenarios (refer to [`EvmRosenExtractor` tests](https://github.com/rosen-bridge/utils/blob/dev/packages/rosen-extractor/tests/getRosenData/evm/EvmRosenExtractor.spec.ts) for example):
    - A successful extraction scenario.
    - A failure extraction scenario, where the other Rosen Extractor returns `undefined`.
    - A failure conversion scenario, where it fails to convert the transaction to the desired type.

> Note: Use this approach when your network's transaction format can be easily converted to an existing supported format.

### Observation Extractor
Steps to implement the Observation Extractor for the new network API:

1. A package should already exist for ChainX as `@rosen-bridge/chainx-observation-extractor` in the [Scanner repository](https://github.com/rosen-bridge/scanner).

2. Implement a class to interact with the blockchain. It should inherit from the `AbstractNetworkConnector` class (refer to the [`BitcoinRpcNetwork` implementation](https://github.com/rosen-bridge/scanner/blob/dev/packages/scanners/bitcoin-rpc-scanner/lib/BitcoinRpcNetwork.ts) for example).

    - name convention: `ChainXApiNetwork`

3. Implement the scanner class. It should inherit from the `AbstractObservationExtractor` class (refer to the [`BitcoinRpcObservationExtractor` implementation](https://github.com/rosen-bridge/scanner/blob/dev/packages/observation-extractors/bitcoin-observation-extractor/lib/BitcoinRpcObservationExtractor.ts) for example).

    - name convention: `ChainXApiObservationExtractor`

    > **Important Note**: The generic type should be exactly the same as the one used in the corresponding scanner and Rosen Extractor. The type should be imported from the scanner package.

4. Unit tests are required only if the `processTransactions` function is re-implemented or any logic is added to the package.

### Scanner Sync Check
Steps to implement a Scanner Sync health check parameter for the new network API:

1. A directory should already exist for ChainX in the `scanner-sync-check` package in the [Health Check repository](https://github.com/rosen-bridge/health-check/tree/dev/packages/scanner-sync-check).

2. Implement the Scanner sync check class. It should inherit from the `AbstractScannerSyncHealthCheckParam` class (refer to the [`BitcoinEsploraScannerHealthCheck` implementation](https://github.com/rosen-bridge/health-check/blob/dev/packages/scanner-sync-check/lib/bitcoin/esplora.ts) for example).

    - name convention: `ChainXApiScannerHealthCheck`
    - suggested `getId` function: 
      ```ts
      getId = (): string => {
        return `chainx_api_scanner`; // replace chainx and api
      };
      ```

    - suggested `getTitle` function:
      ```ts
      getTitle = async () => {
        return `ChainX Api Scanner Sync`; // replace ChainX and Api
      };
      ```
      
    - suggested `getLastSavedBlockMessage` function:
      ```ts
      getLastSavedBlockMessage = async () => {
        return `The last block saved by the ChainX Api scanner is ${this.lastBlockHeight}.`; // replace ChainX and Api
      };
      ```


## Guard Service Integration
Adding a new network API for the Guard service typically requires implementing two modules: a network interface class and an asset health check parameter. Some exceptions exist; for example, `EvmRpcNetwork` requires a scanner with a specific extractor configuration.

### Abstract Chain Network
This section covers implementing the network API interface that the Guard service will use to interact with the blockchain.

Steps to implement the network API for the new blockchain:

1. Add a new package to the [Rosen Chains repository](https://github.com/rosen-bridge/rosen-chains).

    - initialize the package using `kodegen`:
      ```bash
      npx kodegen monorepo add-package
      ```
    - set package name as `@rosen-chains/chainx-api` (e.g., a network for Bitcoin based on Esplora explorer will be `@rosen-chains/bitcoin-esplora`)
    - set package path as `./packages/networks/chainx-api`
    - set description as `A package to be used as network api provider for @rosen-chains/chainx package`
    - set package repo url as `https://github.com/rosen-bridge/rosen-chains`
    - enable both features
      - `Prettier and Eslint`
      - `Testing (with coverage support)`

2. Implement a class to interact with the blockchain. It should inherit from the `AbstractChainXNetwork` class which is defined in the `@rosen-chains/chainx` package (refer to the [`BitcoinEsploraNetwork` implementation](https://github.com/rosen-bridge/rosen-chains/blob/dev/packages/networks/bitcoin-esplora/lib/BitcoinEsploraNetwork.ts) for example).

    - name convention: `ChainXApiNetwork`

3. Implement unit tests for all functions of the network class. Note that no real request should be sent in the tests and the connector should be completely mocked.
    - for tests, refer to [`BitcoinEsploraNetwork` tests](https://github.com/rosen-bridge/rosen-chains/blob/dev/packages/networks/bitcoin-esplora/tests/BitcoinEsploraNetwork.spec.ts)
    - mocking depends on the network connector. For mocking `axios` refer to [`axios.mock.ts`](https://github.com/rosen-bridge/rosen-chains/blob/dev/packages/networks/bitcoin-esplora/tests/mocked/axios.mock.ts) in the `bitcoin-esplora` tests. For mocking classes, something like the `ethers.JsonRpcProvider`, refer to [`JsonRpcProvider.mock.ts`](https://github.com/rosen-bridge/scanner/blob/dev/packages/scanners/evm-rpc-scanner/tests/mocked/JsonRpcProvider.mock.ts) in the `evm-rpc-scanner` tests.

### Asset Check
Steps to implement an Asset health check parameter for the new blockchain:

1. A directory should already exist for ChainX in the `asset-check` package in the [Health Check repository](https://github.com/rosen-bridge/health-check/tree/dev/packages/asset-check).

2. Implement the Asset check class. It should inherit from the `AbstractAssetHealthCheckParam` class (refer to the [`CardanoBlockFrostAssetHealthCheckParam` implementation](https://github.com/rosen-bridge/health-check/blob/dev/packages/asset-check/lib/cardano/blockFrost.ts) for example).

    - name convention: `ChainXApiAssetHealthCheckParam`

3. Implement unit tests for the `update` function (refer to [`CardanoBlockFrostAssetHealthCheckParam` tests](https://github.com/rosen-bridge/health-check/blob/dev/packages/asset-check/tests/cardano/blockFrost.spec.ts) for example). Note that no real request should be sent in the tests and the connector should be completely mocked.
