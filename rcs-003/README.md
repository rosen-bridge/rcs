# RCS-003: Bridge Expansion Kit
* Author: [@RaaCT0R](https://github.com/RaaCT0R)
* Created: 27-May-2025
* Updated: -

## Contents
- [Description](#description)
- [Requirements](#requirements)
- [Modules](#modules)
  - [Scanner](#scanner)
  - [Address Codec](#address-codec)
  - [Rosen Extractor (Network-based)](#rosen-extractor-network-based)
  - [Observation Extractor](#observation-extractor)
  - [Rosen Chain](#rosen-chain)
    - [Abstract Chain (Bases)](#abstract-chain-bases)
    - [Rosen Extractor (Universal)](#rosen-extractor-universal)
    - [Abstract Chain](#abstract-chain)
    - [Abstract Chain Network](#abstract-chain-network)
  - [Health Check](#health-check)
    - [Asset Check](#asset-check)
    - [Scanner Sync Check](#scanner-sync-check)
- [Integration](#integration)
  - [Watcher Service](#watcher-service)
  - [Guard Service](#guard-service)
  - [UI](#ui)
  - [Contracts](#contracts)
- [Common Cases](#common-cases)
  - [EVM Chain](#evm-chain)
    - [EVM Address Codec](#evm-address-codec)
    - [EVM Rosen Extractor](#evm-rosen-extractor)
    - [EVM Observation Extractor](#evm-observation-extractor)
    - [EVM Rosen Chain](#evm-rosen-chain)
    - [Watcher and Guard Service](#watcher-and-guard-service)
  - [Bitcoin Fork](#bitcoin-fork)
- [Extending Networks](#extending-networks)

## Description
This document outlines the requirements and steps to integrate a new blockchain into Rosen Bridge.

## Requirements
To support a blockchain in Rosen Bridge, two base requirements should be satisfied:

1. Multi-Signer Addresses

    All assets should be held in the bridge address that requires multiple signers (guards) to transfer from it. Currently, two approaches are used by Rosen; Threshold Signature Scheme (TSS) and Multi-Signature (usually handled on the contract side)

2. Data Writing

    To bridge an asset, in addition to transferring the asset to the bridge address, various data including fee, target chain and target address should be provided. The data is written on the transferring transaction which is also known as the lock transaction.

Based on the chain architecture and the integration type, some requirements may already be present or be omitted:

- Tokens Supporting

    The blockchain may or may not support tokens. If assets are to be transferred from other chains into it (e.g., transferring Erg from Ergo into Chain-X), it should support tokens. However, this may not apply to certain blockchains, such as Bitcoin, which currently only supports BTC bridging.

- dApp Connector

    Users initiate a transfer through a transaction, which typically has a specific structure that may not be easy for them to generate. A convenient solution is to use dApp connectors to interact with users' wallets, enabling automatic generation and signing of transactions.

- Transaction Chaining

    As the bridge scales and the number of users and supported chains increases, multiple requests to the same target chain can arise simultaneously. Chaining these transactions can enhance event processing efficiency; however, it may also lead to deadlocks and delays in payments in some edge cases.

- Event Distinguishability

    It is possible for two identical events to occur at the same time, so the agreed transactions should be distinguishable to avoid double payments or missing an event.

- Transaction Fee Handling

    The time between a bridge request and its payment on the target chain can be lengthy for various reasons. During this period, transaction fees may fluctuate significantly, with some blockchains, like Ethereum, experiencing increases of up to 700% in just one or two hours. The design should minimize the impact of these fluctuations to prevent asset loss and avoid unexpected issues.


## Modules
Once the requirements are checked and the adaption is confirmed, multiple Rosen modules should be updated to support the new blockchain. It is highly recommended to implement them in the following order:

  1. [Scanner](#scanner)
  2. [Rosen Extractor (Network-based)](#rosen-extractor-network-based)
  3. [Observation Extractor](#observation-extractor)
  4. [Rosen Chain](#rosen-chain)
  5. [Health Check](#health-check)
  6. [UI (Lock Transaction)](#ui-lock-transaction)

> Note: This section describes adding a new blockchain with a single network (i.e., a specific API or service used to interact with the blockchain). For adding a new network (e.g., new explorer or API) to an existing chain, refer to the [Extending Networks](#extending-networks) section.

> Note: The naming convention is described in each section. the `ChainX` represents the new blockchain name and `Api` represent the network API that is used. Examples for Bitcoin blockchain and RPC API:
  - `@rosen-bridge/chainx-api-scanner` -> `@rosen-bridge/bitcoin-rpc-scanner`
  - `ChainXChain` -> `BitcoinChain`
  - `ChainXApiNetwork` -> `BitcoinRpcNetwork`

### Scanner
The Scanner module is responsible for fetching all related data from the blockchain. Based on the scanner type and usage, the related data to be scanned might differ. However, because it is almost always in the form of a transaction, we will make that assumption throughout the rest of this document.

Steps to implement a Scanner for the new blockchain:

1. Add a new package to the [Scanner repository](https://github.com/rosen-bridge/scanner).

    - initialize the package using `kodegen`:
      ```bash
      npx kodegen monorepo add-package
      ```
    - set package name as `@rosen-bridge/chainx-api-scanner` (e.g., a scanner for Bitcoin based on RPC API will be `@rosen-bridge/bitcoin-rpc-scanner`)
    - set package path as `./packages/scanners/chainx-api-scanner`
    - set description as `A Chain-X blockchain scanner based on network API` (same as before, replace `api` with the corresponding network such as RPC, Explorer, GraphQL, etc.)
    - set package repo url as `https://github.com/rosen-bridge/scanner`
    - enable both features
      - `Prettier and Eslint`
      - `Testing (with coverage support)`

2. Implement a class to interact with the blockchain. It should inherit from the `AbstractNetworkConnector` class (refer to the [`BitcoinRpcNetwork` implementation](https://github.com/rosen-bridge/scanner/blob/dev/packages/scanners/bitcoin-rpc-scanner/lib/BitcoinRpcNetwork.ts) for example).

    - name convention: `ChainXApiNetwork`

3. Implement the scanner class. It should inherit from the `GeneralScanner` class (refer to the [`BitcoinRpcScanner` implementation](https://github.com/rosen-bridge/scanner/blob/dev/packages/scanners/bitcoin-rpc-scanner/lib/BitcoinRpcScanner.ts) for example).

    - name convention: `ChainXApiScanner`

4. Implement unit tests for all functions of the network class. Note that no real request should be sent in the tests and the connector should be completely mocked.
    - for tests, refer to [`BitcoinRpcNetwork` tests](https://github.com/rosen-bridge/scanner/blob/dev/packages/scanners/bitcoin-rpc-scanner/tests/BitcoinRpcNetwork.spec.ts)
    - mocking depends on the network connector. For mocking `axios` refer to [`axios.mock.ts`](https://github.com/rosen-bridge/scanner/blob/dev/packages/scanners/bitcoin-rpc-scanner/tests/mocked/axios.mock.ts) in the `bitcoin-rpc-scanner` tests. For mocking classes (e.g., `ethers.JsonRpcProvider`) refer to [`JsonRpcProvider.mock.ts`](https://github.com/rosen-bridge/scanner/blob/dev/packages/scanners/evm-rpc-scanner/tests/mocked/JsonRpcProvider.mock.ts) in the `evm-rpc-scanner` tests.

### Address Codec
The Address Codec module is responsible for encoding and verification of addresses in Rosen Bridge.

Steps to integrate the new blockchain into Address Codec, which is in the [Utils repository](https://github.com/rosen-bridge/utils/tree/dev/packages/address-codec) under the path `packages/address-codec/lib`:

1. Add the new chain name to [`const.ts` file](https://github.com/rosen-bridge/utils/blob/dev/packages/address-codec/lib/const.ts).

2. Implement the encoder in [`encoder.ts` file](https://github.com/rosen-bridge/utils/blob/dev/packages/address-codec/lib/encoder.ts). Note that the encoded address should be a hex string **without any leading `0x`**.

2. Implement the decoder in [`decoder.ts` file](https://github.com/rosen-bridge/utils/blob/dev/packages/address-codec/lib/decoder.ts).

3. Implement the validation logic in [`validator.ts` file](https://github.com/rosen-bridge/utils/blob/dev/packages/address-codec/lib/validator.ts).

  [_View file difference in Doge integration_](https://github.com/rosen-bridge/utils/commit/fc41c1386bb00e1a798f4159aa97e365bed6e85b)

4. Implement unit tests for all scenarios (refer to [Doge integration tests](https://github.com/rosen-bridge/utils/commit/9c8886cfb0b337179c37f8afbe887a446f123a52) for example)


### Rosen Extractor (Network-based)
The Rosen Extractor module is responsible for extracting bridge request information from a transaction. The data consists of:
  - _`toChain`_: the target blockchain
  - _`toAddress`_: the destination address on the target blockchain
  - _`bridgeFee`_: request bridge fee
  - _`networkFee`_: request network fee
  - _`fromAddress`_: the source address on the source blockchain (which is transferring the assets to the lock address)
  - _`sourceChainTokenId`_: the token id on the source blockchain
  - _`amount`_: the amount of transfer
  - _`targetChainTokenId`_: the token id on the target blockchain
  - _`sourceTxId`_: the transaction id

Note that the bridge and network fees mentioned above are the fees specified in the transaction, not the actual fees incurred. If the specified fees are lower than the minimum fee (which is configurable and recorded in boxes on the Ergo chain; see the [minimum-fee package](https://github.com/rosen-bridge/utils/tree/dev/packages/boxes/minimum-fee) for more details), the guards will use the minimum fee instead.

Before implementing the Rosen Extractor module, it is crucial to generate example transactions that contain the data explained above. These test transactions enable us to ensure that the implemented extractor can accurately identify and extract the required information. For example, the OP_RETURN output in [this transaction](https://blockstream.info/tx/b1f7938d1c0c43eeed96bee4c0f479a0bfb2e255abeb38cc4f2dab6f51d40b45?expand) is encoding the mentioned data in a transaction using [the `generateOpReturnData` function](https://github.com/rosen-bridge/ui/blob/dd317b22d1ad335a0274a3b590ebb945308f3f28/networks/bitcoin/src/utils.ts#L23). The data also can be decoded using [the `parseRosenData` function](https://github.com/rosen-bridge/utils/blob/c13c77bae45daea03ff9bbe4f7006b0d849b95cc/packages/rosen-extractor/lib/utils.ts#L9).

Steps to implement a Rosen Extractor for the new blockchain:

1. Add a new directory to the `rosen-extractor` package in the [Utils repository](https://github.com/rosen-bridge/utils/tree/dev/packages/rosen-extractor).

    ```
    cd lib/getRosenData
    mkdir chainx
    ```

2. Implement the Rosen Extractor class. It should inherit from the `AbstractRosenDataExtractor` class (refer to the [`BitcoinRpcRosenExtractor` implementation](https://github.com/rosen-bridge/utils/blob/dev/packages/rosen-extractor/lib/getRosenData/bitcoin/BitcoinRpcRosenExtractor.ts) for example).

    - name convention: `ChainXApiRosenExtractor`

    > **Important Note**: The generic type should be exactly the same one that is used in the corresponding scanner. The exact type should be defined in the package itself and it cannot be imported from the scanner package.

3. Implement unit tests for all scenarios (refer to [`BitcoinRpcRosenExtractor` tests](https://github.com/rosen-bridge/utils/blob/dev/packages/rosen-extractor/tests/getRosenData/bitcoin/BitcoinRpcRosenExtractor.spec.ts) for example)

The new chain should also be added to the `SUPPORTED_CHAINS` list in the [`const.ts` file](https://github.com/rosen-bridge/utils/blob/dev/packages/rosen-extractor/lib/getRosenData/const.ts).

### Observation Extractor
The observation extractor is the generic class that utilizes the implemented Rosen extractors (discussed in previous section) to extract bridge requests.

Steps to implement an Observation Extractor for the new blockchain:

1. Add a new package to the [Scanner repository](https://github.com/rosen-bridge/scanner).

    - initialize the package using `kodegen`:
      ```bash
      npx kodegen monorepo add-package
      ```
    - set package name as `@rosen-bridge/chainx-observation-extractor`
    - set package path as `./packages/observation-extractors/chainx-observation-extractor`
    - set description as `Event observation data extractor for Bitcoin chain`
    - set package repo url as `https://github.com/rosen-bridge/scanner`
    - enable both features
      - `Prettier and Eslint`
      - `Testing (with coverage support)`

2. Implement the observation extractor class. It should inherit from the `AbstractObservationExtractor` class (refer to the [`BitcoinRpcObservationExtractor` implementation](https://github.com/rosen-bridge/scanner/blob/dev/packages/observation-extractors/bitcoin-observation-extractor/lib/BitcoinRpcObservationExtractor.ts) for example).

    - name convention: `ChainXApiObservationExtractor`

    > **Important Note**: The generic type should be exactly the same one that is used in the corresponding scanner and Rosen Extractor. The type should be imported from the scanner package.

4. The unit tests are required only if the `processTransactions` function is re-implemented or any logic is added to the package.

### Rosen Chain
[The `@rosen-chains` packages](https://github.com/rosen-bridge/rosen-chains) contain the most functionalities of the new blockchain and mainly is used in the Guard Service. For the sake of simplicity, the implementation is divided into four parts and it is highly recommended to implement them in the following order:

  1. [Abstract Chain (Bases)](#abstract-chain-bases)
  2. [Rosen Extractor (Universal)](#rosen-extractor-universal)
  3. [Abstract Chain](#abstract-chain)
  4. [Abstract Chain Network](#abstract-chain-network)
  
#### Abstract Chain (Bases)
This part is mostly about initializing the package, designing the types and researching the required functions. In this section, only the steps are explained and the detailed document on each function is available in the [Abstract Chain README](https://github.com/rosen-bridge/rosen-chains/blob/dev/packages/abstract-chain/README.md).

There are two types of chains:
- `AbstractChain`
- `AbstractUtxoChain`, which inherits from the `AbstractChain` itself

Based on the ChainX structure, the correct class should be inherited. A generic type, `TxType`, should be defined which represents the type of the transactions fetched from the blockchain. This means that transactions fetched from any API must be convertible into this defined type in order for the API to be supported.

If ChainX is UTXO-based, another generic type, `BoxType`, should also be defined. This type is also independent of the network API and should be designed based on the required functions. At this stage, only a base design is necessary; implementation is not required yet. The design can be further adjusted during the actual implementation, which is covered in [part 3](#abstract-chain).

The main purpose of the research in this part is to answer the question **"How do you intend to implement it?"** for each function of the `AbstractChain`, and **"What other network functions are required in the `AbstractChainXNetwork` class?"** for the network API interface.

After research and design, the package can be initialized:

  - initialize the package using `kodegen`:
    ```bash
    npx kodegen monorepo add-package
    ```
  - set package name as `@rosen-chains/chainx`
  - set package path as `./packages/chains/chainx`
  - suggested description: `this project contains chainX chain for Rosen-bridge`
  - set package repo url as `https://github.com/rosen-bridge/rosen-chains`
  - enable both features
    - `Prettier and Eslint`
    - `Testing (with coverage support)`

Two required classes should be defined:

  - define the network interface as an abstract class at `./lib/network` with all other required network functions. It should inherit from the `AbstractChainNetwork` or `AbstractUtxoChainNetwork` classes.
    - name convention: `AbstractChainXNetwork`
  - define the chain class that inherits from the `AbstractChain` or `AbstractUtxoChain`. All functions should throw an error (the implementation is in [part 3](#abstract-chain)).
    - name convention: `ChainXChain`

    ```ts
    throw Error(`not implemented`);
    ```

An example of this part is the ["Bitcoin: Abstract Network" Merge Request](https://github.com/rosen-bridge/rosen-chains/commit/1afb500f60669b5eeb0f399a755a9f4d7ae9bdd4) (Note that this MR is old and the [current Abstract Chain](https://github.com/rosen-bridge/rosen-chains/blob/dev/packages/abstract-chain/lib/AbstractChain.ts) is slightly different).

#### Rosen Extractor (Universal)
There are two types of Rosen Extractors. Similar to the Scanner, each network requires its own specific Rosen Extractor. For instance, since both Esplora and the RPC API of Bitcoin are supported, two Extractors are needed: `BitcoinEsploraRosenExtractor` and `BitcoinRpcRosenExtractor`. These extractors are network-specific. The second type, Universal, is used in the rosen-chains packages. The main difference between the two types is the structure of the transactions they handle.

Universal Rosen extractor should be compatible with all network-specific extractors defined in previous parts. Consequently, only one implementation of the universal rosen extractor is required per chain. For a general understanding of the Rosen Extractors, refer to [Rosen Extractor (Network-based)](#rosen-extractor-network-based).

Steps to implement the universal Rosen Extractor for the new blockchain:

1. A directory should already exist for ChainX in the `rosen-extractor` package in the [Utils repository](https://github.com/rosen-bridge/utils/tree/dev/packages/rosen-extractor).

2. Implement the Rosen Extractor class. It should inherit from the `AbstractRosenDataExtractor` class (refer to the [`BitcoinRosenExtractor` implementation](https://github.com/rosen-bridge/utils/blob/dev/packages/rosen-extractor/lib/getRosenData/bitcoin/BitcoinRosenExtractor.ts) for example).

    - name convention: `ChainXRosenExtractor`

    > **Important Note**: The generic type should be exactly the same one that is used in the chain (i.e. the generic `TxType` which is defined in previous part). The exact type should be defined in the package itself and it cannot be imported from the rosen-chains package.

3. Implement unit tests for all scenarios (refer to [`BitcoinRosenExtractor` tests](https://github.com/rosen-bridge/utils/blob/dev/packages/rosen-extractor/tests/getRosenData/bitcoin/BitcoinRosenExtractor.spec.ts) for example)


#### Abstract Chain
After implementing the universal Rosen Extractor, the chain class which is defined in [part 1](#abstract-chain-bases) can be implemented. The implementation for this part should be straightforward, although the defined types and interfaces may be altered to satisfy the requirements.

#### Abstract Chain Network
The final part of Rosen Chains implementation is implementing the first network API for the new chain. 

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

2. Implement a class to interact with the blockchain. It should inherit from the `AbstractChainXNetwork` class, which is defined in [part 1](#abstract-chain-bases) and [part 3](#abstract-chain) (refer to the [`BitcionEsploraNetwork` implementation](https://github.com/rosen-bridge/rosen-chains/blob/dev/packages/networks/bitcoin-esplora/lib/BitcoinEsploraNetwork.ts) for example).

    - name convention: `ChainXApiNetwork`

3. Implement unit tests for all functions of the network class. Note that no real request should be sent in the tests and the connector should be completely mocked.
    - for tests, refer to [`BitcoinEsploraNetwork` tests](https://github.com/rosen-bridge/rosen-chains/blob/dev/packages/networks/bitcoin-esplora/tests/BitcoinEsploraNetwork.spec.ts)
    - mocking depends on the network connector. For mocking `axios` refer to [`axios.mock.ts`](https://github.com/rosen-bridge/rosen-chains/blob/dev/packages/networks/bitcoin-esplora/tests/mocked/axios.mock.ts) in the `bitcoin-esplora` tests. For mocking classes, something like the `ethers.JsonRpcProvider`, refer to [`JsonRpcProvider.mock.ts`](https://github.com/rosen-bridge/scanner/blob/dev/packages/scanners/evm-rpc-scanner/tests/mocked/JsonRpcProvider.mock.ts) in the `evm-rpc-scanner` tests.

### Health Check
As the name suggests, these packages check the healthiness of some components in both Watcher and Guard services. For new blockchains, two parameters are required.

#### Asset Check
This health check parameter monitors the balance of specified assets at given addresses, primarily used by the Guard service to ensure sufficient native token to pay bridge transaction fees.

Steps to implement an Asset health check parameter for the new blockchain:

1. Add a new directory to the `asset-check` package in the [Health Check repository](https://github.com/rosen-bridge/health-check/tree/dev/packages/asset-check).

    ```
    cd lib
    mkdir chainx
    ```

2. Implement the Asset check class. It should inherit from the `AbstractAssetHealthCheckParam` class (refer to the [`CardanoBlockFrostAssetHealthCheckParam` implementation](https://github.com/rosen-bridge/health-check/blob/dev/packages/asset-check/lib/cardano/blockFrost.ts) for example).

    - name convention: `ChainXApiAssetHealthCheckParam`

3. Implement unit tests for the `update` function (refer to [`CardanoBlockFrostAssetHealthCheckParam` tests](https://github.com/rosen-bridge/health-check/blob/dev/packages/asset-check/tests/cardano/blockFrost.spec.ts) for example). Note that no real request should be sent in the tests and the connector should be completely mocked.

#### Scanner Sync Check
This parameter checks the last scanned block with the current block of the network and is mainly used in the Watcher service.

Steps to implement a Scanner Sync health check parameter for the new blockchain:

1. Add a new directory to the `scanner-sync-check` package in the [Health Check repository](https://github.com/rosen-bridge/health-check/tree/dev/packages/scanner-sync-check).

    ```
    cd lib
    mkdir chainx
    ```

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


## Integration
After implementing the required modules, they can be integrated into the main services. Unlike module implementation, the order does not matter in the integration phase, and the chain can be deployed once its modules are integrated into the four services:

  - [Watcher Service](#watcher-service)
  - [Guard Service](#guard-service)
  - [Rosen App](#rosen-app)
  - [Contracts](#contracts)

> Note: This section describes adding a new blockchain with a single network. For adding a new network (e.g., new explorer or API) to an existing chain, refer to the [Extending Networks](#extending-networks) section.

### Watcher Service
Integrating a new chain into the Watcher service consists of adding the new scanner, observation extractor and scanner sync health check parameter.

  - The config class and interface should be defined for the new chain in the path `src/config/config.ts`. Three fields are required:
    - _`type`_: the selected network API
    - _`initial.height`_: the starting height of the blockchain for scanning
    - _`api`_: contains required fields for the network api (replace `api` with the actual network API name)

    for example, Bitcoin config with Esplora network looks like this:
    ```ts
    class BitcoinConfig {
      type: string;
      esplora?: {
        url: string;
        timeout: number;
        initialHeight: number;
        interval: number;
      };

      constructor(network: string) {
        ...
      }
    }
    ```

    > Please refer to [Bitcoin implementation](https://github.com/rosen-bridge/watcher/blob/a2ea25316a165902b79e8e8e92a06ef948fc1d4d/src/config/config.ts#L351) for `constructor` implementation.

    The config class should be added to the `ConfigType` interface and should be initialized in the `getConfig` function. Also, the chain name should be added to the `supportedNetworks` list.

    [_View file difference in Bitcoin integration_](https://github.com/rosen-bridge/watcher/commit/a2ea25316a165902b79e8e8e92a06ef948fc1d4d#diff-615dacd7000fa06d7afef0f22b7b6d9267c6219fbfc9fa0315e51a5f5d963755)

    The new chain health check config which consists of the scanner warning and critical difference should be fetched in config (refer to [this commit](https://github.com/rosen-bridge/watcher/commit/b6b01126fa67e5f15856aebbd3bb61b6bacf005d) for example).

  - A default config should be provided in the `default.yaml` file. [View Bitcoin integration](https://github.com/rosen-bridge/watcher/commit/a2ea25316a165902b79e8e8e92a06ef948fc1d4d#diff-a539f2e066889598fd148cd5d815d0bfd6e8c472bf4474a91c441f08c8c3a31b) for example (The health-check parameters update in the Yaml file is available at [this commit](https://github.com/rosen-bridge/watcher/commit/b6b01126fa67e5f15856aebbd3bb61b6bacf005d#diff-a539f2e066889598fd148cd5d815d0bfd6e8c472bf4474a91c441f08c8c3a31b)).

  - The scanner and observation extractor class initializations should be added to the `createScanner` class. A new private function should be defined to initialize these classes and in the `constructor`, the condition to call this private function should be added.

    The specification for these changes is clear in [the `src/utils/scanner.ts` file difference](https://github.com/rosen-bridge/watcher/commit/a2ea25316a165902b79e8e8e92a06ef948fc1d4d#diff-0567662924cbbd9fe10ca0223eff2d2624936dbb0e8b975354a93833abeb51a9) in Bitcoin integration.

  - The job to update the new scanner should be added to the `scannerInit` function in `src/jobs/initScanner.ts`

    [_View file difference in Bitcoin integration_](https://github.com/rosen-bridge/watcher/commit/a2ea25316a165902b79e8e8e92a06ef948fc1d4d#diff-e4e17a37d67ba40d7721e65ca34a0ec2b3d5f271cba5a9162584d61cd81793e8)

  - The scanner sync health check param should be added to the `HealthCheckSingleton`. Similar to the scanner, a new private function should be defined to initialize the health check parameter and register it into the health check (refer to [this commit](https://github.com/rosen-bridge/watcher/commit/b6b01126fa67e5f15856aebbd3bb61b6bacf005d#diff-e20d1c9cffa1cf2496ab9c729a37c3bdb020bb5fbb3870002dd60804671b0b72) for example).


### Guard Service
In order to integrate a new chain into the Guard service, the following changes are required:

  - A config class for the new chain should be defined. Other than four addresses and confirmations which are required for every chain, additional may be required. The required config interface is defined in the [Abstract Chain](#abstract-chain) section. This class should be defined in `src/configs/` path with `GuardsChainXConfigs` as the name convention.

    [_View file difference in Bitcoin integration_](https://github.com/rosen-bridge/guard-service/commit/2a6232b6326a72487993ce42dd12b4029cd96cb1#diff-60abca0e85ab429640f09cd1f5149b2689cf72ce457a6673e4b5961a4da11765)

  - The asset health check warning and critical thresholds for the new chain native token should be fetched in the config file at the `src/configs/Configs.ts` path.

    [_View file difference in Bitcoin integration_](https://github.com/rosen-bridge/guard-service/commit/6ac66824d18be0f3c5722a08f958ab54f8c3202f#diff-f7b5390def5289025c80d06ffa8ff92604a08974b4f5c2a0e42c13943a03652a)

  - A default config should be provided in the `default.yaml` file. [View Bitcoin integration](https://github.com/rosen-bridge/guard-service/commit/2a6232b6326a72487993ce42dd12b4029cd96cb1#diff-a539f2e066889598fd148cd5d815d0bfd6e8c472bf4474a91c441f08c8c3a31b) for example (The health-check parameters update in the Yaml file is available at [this commit](https://github.com/rosen-bridge/guard-service/commit/6ac66824d18be0f3c5722a08f958ab54f8c3202f#diff-a539f2e066889598fd148cd5d815d0bfd6e8c472bf4474a91c441f08c8c3a31b)).

  - The new asset health check param should be initialized and registered in the `getHealthCheck` function in the `src/guard/HealthCheck.ts` path.

    [_View file difference in Bitcoin integration_](https://github.com/rosen-bridge/guard-service/commit/6ac66824d18be0f3c5722a08f958ab54f8c3202f#diff-bf50dc03b444499664d25f80398da45dc0a7d120643229a78dc6246e3f315549)

  - The commitment and event trigger extractor for the new chain should be initialized and registered in the `src/jobs/initScanner.ts` path.

    [_View file difference in Bitcoin integration_](https://github.com/rosen-bridge/guard-service/commit/d7e6d869cbbc20cbfca4334fbc1218648884abda#diff-e4e17a37d67ba40d7721e65ca34a0ec2b3d5f271cba5a9162584d61cd81793e8)

  - A new private function should be defined in the `src/handlers/ChainHandler.ts` path to initialize Rosen Chains packages for the new chain.

    [_View file difference in Bitcoin integration_](https://github.com/rosen-bridge/guard-service/commit/2a6232b6326a72487993ce42dd12b4029cd96cb1#diff-9da44543453c04e80634977b222eb007a618d8f21619bf57a4788d8bc19a3b22)

  - The new chain name should be added to the `SUPPORTED_CHAINS` list and `ChainNativeToken` object in the `src/utils/constants.ts` path.

    [_View file difference in Ethereum integration_](https://github.com/rosen-bridge/guard-service/commit/ac6cc9b4d80aea609381c1b447a1e3a1ff34e3ee#diff-e8d0a358ae2fb090c56f00a058050983cc9691b7d3622ddd443170017123e7d9)

  - If the chain has any secret configs, such as mnemonic, secret key, auth tokens, etc. it should be added to custom environment variables in the `docker/custom-environment-variables.yaml` path.

    [_View file difference in Ethereum integration_](https://github.com/rosen-bridge/guard-service/commit/a4e828f287519d6ebb871d520b61bf2c135da748)

### UI
[Rosen UI Repository](https://github.com/rosen-bridge/ui) is composed of multiple applications and various packages. To integrate a new blockchain into the UI, several critical steps must be undertaken, including the implementation of new chain logics, such as lock transaction generation, and ensuring compatibility with wallets—either by adding the new chain to an existing wallet or by developing a new wallet entirely.

The new modules designed for the integration should seamlessly connect with the Rosen Service. Additionally, most other applications and packages will only require minimal adjustments to accommodate the new integration.

To ensure a smooth and organized integration, the following steps should be prioritized and executed in the specified order. These steps have been structured to minimize type conflicts, ensure logical consistency, and allow incremental progress across packages.

  1. Add base data for the new chain, which requires changes in following packages:
      - [Icons](#icon)
      - [Constants](#constants)
      - [Utils](#utils)
      - [Network Package (Bases)](#network-package-bases)
      - [Rosen App](#rosen-app)
  2. [Asset Calculator](#asset-calculator)
  3. [Rosen Service](#rosen-service)
  4. [Network Package](#network-package)
  5. [Wallet package](#wallet-package)
  6. [Wallet Configuration in Rosen App](#wallet-configuration-in-rosen-app)

  > Note: All changes of step 1 should be implemented in a single merge request.
  
  > Note: Don't forget to rebuild the packages after applying changes of each packages in step 1

#### Icons
A chain icon file, named to match the chain key, should be placed inside the `src/networks` directory of the `@rosen-bridge/icons` package (located at `packages/icons`). The SVG should be monochrome, with no specific `colors` or `sizes` mentioned.

#### Constants
A new key needs to be added to the `NETWORKS` constant object in the `@rosen-ui/constants` package (located at `packages/constants`), named after the new chain key, and an object related to the key needs to be defined to specify the required chain properties.

```
export const NETWORKS = {
  ...
  NEW_NETWORK_KEY: {
    index: 0,
    key: 'NEW_NETWORK_KEY',
    label: 'NEW_NETWORK_LABEL',
    nativeToken: 'NEW_NETWORK_NATIVE_TOKEN_ID',
    id: 'NEW_NETWORK_ID',
  },
  ...
} as const;
```

> Note: This package also affects `@rosen-ui/types` and both packages should be build after applying the changes.

#### Utils
Some new functions should be added to the `@rosen-ui/utils` package (located at `packages/utils`):

- Implement address url logic for the chain inside `src/getAddressUrl.ts`
- Implement token url logic for the chain inside `src/getTokenUrl.ts`
- Implement tx url logic for the chain inside `src/getTxUrl.ts`

#### Network Package (Bases)
This part focuses on initializing a new package inside the `networks` directory to establish the foundation for chain-specific logic. At this stage, functions can simply throw errors as placeholders, with full implementation coming later in the [Network Package](#network-package) section.

Add a new package inside the `networks` directory, including all of 
the chain shared logic, such as:

- Expose a class that implemented the `Network` interface from the `@rosen-network/base`
  - Contains the new chain info (e.g., name, label, logo, lock address, etc.)
  - Implements the `getMaxTransfer`, calculating max transfer for an asset based on user balance and other required parameters (which may vary for each chain)
- A function to generate the lock transaction
- The new chain constants
- The new package should be added to the `build.sh` file

#### Rosen App
These updates are required in the Rosen App (located at `app/rosen`):

- Add the new network package to the `package.json` file
- Add a new directory to `app/_networks`, named after the new network key. Three files should be included: `client.ts`, `index.ts` and `server.ts`. The `client.ts` file should export a new instance of the network defined in `@rosen-network/NEW_NETWORK_KEY`. The `index.ts` file should simply re-export the `client.ts`. The `server.ts` file should export the server actions that need to be used in the network constructor within the `client.ts` file
- Implement a function to fetch the chain current height in `app/_actions/calculateFee.ts`
- Add network API env var to the `.env.example`
- Import the client instance in the `app/_networks/index.ts` file

#### Asset Calculator
The new chain logic should be added to the `@rosen-ui/asset-calculator` (located at `packages/asset-calculator`):

- A chain calculator class should be added in the `lib/calculator/chains` directory, extending `AbstractCalculator`, and then configured in `lib/asset-calculator.ts`
- Add a chain-specific interface that inherits from `CalculatorInterface` to the `lib/interfaces.ts` file

#### Rosen Service
The new chain logic should be added to Rosen Service (located at `app/rosen-service`):

- Configure asset calculator for Rosen service in `config/default.yaml`, providing the list of addresses to watch, network API urls, etc.
- Add chain configs (addresses, backend urls, RWT token id, etc.) in `src/configs.ts` file
- Add a scanner for the chain (in `src/scanner/chains` directory), and add it to the list of scanners in `scanner-service.ts`
- Add an observation extractor for the chain (inside of `src/observation/chains`), and add it to `observationService` object  in `observation-service.ts`
- Add the event trigger extractor for the chain (inside of `src/event-trigger/event-trigger-service.ts`)
- Add relevant constants to `constants.ts`, such as scanning intervals
- The `start` function in `src/calculator/calculator-service.ts` should be updated by adding chain configurations to the `AssetCalculator` constructor parameters. These changes depend on the updates in the `@rosen-ui/asset-calculator` package, which must be applied first before updating the mentioned file
- Add two new keys, `chainXScannerWarnDiff` and `chainXScannerCriticalDiff`, under the `healthCheck` section in the `apps/rosen-service/config/default.yaml` file
  ```
  healthCheck:
    chainXScannerWarnDiff: <value>
    chainXScannerCriticalDiff: <value>
  ```

- Update the `registerAllHealthChecks` function in `apps/rosen-service/src/health-check/health-check-service.ts` by adding a new health check instance for the new chain scanner inside the `checks` array
  ```ts
  const checks = [
    ...
    {
      instance: new NewChainScannerHealthCheck(),
      label: 'chainX',
    },
    ...
  ];
  ```

- Define the required constant keys in the `apps/rosen-service/src/constants.ts` file
- Modify the `getConfig` function in `apps/rosen-service/src/configs.ts` to include the new keys under the `healthCheck` section, ensuring they are properly loaded from the configuration

    [_View file difference in Ethereum integration for some of them_](https://github.com/rosen-bridge/ui/commit/4ae4c3bdce335b5280777b85cce9dada7455e0e7)

#### Network Package
Building upon the foundation established in [part 1](#network-package-bases), this section focuses on fully implementing all required functionality in the network package. The implementation should include:

- Complete implementation of `getMaxTransfer` method:
  - Calculate maximum transferable amount for any asset
  - Consider user balance, network fees, and chain-specific limitations
- Full implementation of lock transaction generation:
  - Build transaction structure according to chain specifications
  - Include proper fee calculation
  - Add required metadata for Rosen protocol
  - Validate all inputs and parameters
  - Handle error cases gracefully
- Additional required functionality such as chain-specific helper methods

#### Wallet package
- Add at least one wallet for the chain (_more details to come in the future_), including:
  - Wallet connection logic
  - Address fetching logic
  - Balance fetching logic
  - Transfer logic
  - Availability check logic
  - Wallet info (e.g., name, label, etc.)
  - Add the list of supported chains
- In case of adding a new wallet, it's package should also be added to the `build.sh` script

#### Wallet Configuration in Rosen App

- Configure each wallet inside the `apps/rosen/app/_wallets` directory
- Each wallet directory must contain the following three files:
  - `index.ts`: This file should export the wallet instance
  - `server.ts`: This file should export server-side functions that will be used by the client-side wallet instance
  - `wallet.ts`: This file should export a new wallet instance configured using the server-side actions
- After creating the wallet files, add the wallet instance to the `apps/rosen/app/_wallets/index.ts` file to make it available throughout the app

### Contracts
Integration into contracts consists of minting the new chain config tokens on Ergo (such as RWT, AWC, etc.) and minting bridge tokens on Ergo and any other chains that are supposed to support them. Note that this is unrelated to the new blockchain's contracts, since the Rosen Bridge structure requires minimal integration on the new chain and usually doesn't need contract development on it.

  > **Important Note**: Every bridge token should be wrapped on Ergo chain.

  > Note: This integration will be done by Rosen Team.

  - Adding the new chain native token ([Bitcoin Example](https://github.com/rosen-bridge/contract/commit/ea0b6d1b264462c3f42ad2a60db560c9537a63c4))

  - Adding bridge config tokens for the new chain ([Bitcoin Example](https://github.com/rosen-bridge/contract/commit/7e72dd4054dedbc6817fdec5f1012ce9a6475b05))


## Common Cases
The previous sections explain how to integrate a new blockchain into Rosen Bridge from scratch, while some blockchains support structures that are already integrated into Rosen Bridge. This facilitates the integration as the most of modules are already implemented and the new chain modules should inherit those instead of implementing from scratch.

### EVM Chain
Integrating a new EVM blockchain is much simpler, as its code is already implemented and only a handful of modules need to be updated or added, with almost no new implementation required.

#### EVM Address Codec
The new chain should be added to all `encoder.ts`, `decoder.ts` and `validator.ts` files without any implementation.

  [_View file difference in Binance integration_](https://github.com/rosen-bridge/utils/commit/07b64dea4c02ba019f0fdad7896e341d0132f13f)

#### EVM Rosen Extractor
The new chain should be added to the `SUPPORTED_CHAINS` list in the [`const.ts` file](https://github.com/rosen-bridge/utils/blob/dev/packages/rosen-extractor/lib/getRosenData/const.ts).

#### EVM Observation Extractor
A package is already exist for EVM as `@rosen-bridge/evm-observation-extractor` in the [Scanner repository](https://github.com/rosen-bridge/scanner). The scanner class should be implemented while inheriting from the `EvmRpcObservationExtractor ` class (refer to the [`BinanceRpcObservationExtractor ` implementation](https://github.com/rosen-bridge/scanner/commit/6efd277e5f4a48b2f791c0a4a8546ed4fb21db6d) for example).

    - name convention: `ChainXApiObservationExtractor`

> Note: Don't forget to export the class in the `index.ts`.

#### EVM Rosen Chain
Steps to implement the network API for the new blockchain:

1. Add a new package to the [Rosen Chains repository](https://github.com/rosen-bridge/rosen-chains).

    - initialize the package using `kodegen`:
      ```bash
      npx kodegen monorepo add-package
      ```
  - set package name as `@rosen-chains/chainx`
  - set package path as `./packages/chains/chainx`
  - suggested description: `this project contains chainX chain for Rosen-bridge`
  - set package repo url as `https://github.com/rosen-bridge/rosen-chains`
  - enable both features
    - `Prettier and Eslint`
    - `Testing (with coverage support)`

2. Implement a class inheriting from the `EvmChain` class, which is defined in the `@rosen-chains/evm` package (refer to the [`BinanceChain` implementation](https://github.com/rosen-bridge/rosen-chains/commit/12c736c0f93d3884d204017e244750e854787c62) for example).

3. Define the chain name, native token id and chain id number in `constants.ts` file (refer to the [`EthereumChain` implementation](https://github.com/rosen-bridge/rosen-chains/blob/dev/packages/chains/ethereum/lib/constants.ts) for example).

#### Watcher and Guard Service
Integrating an EVM chain into Watcher and Guard service has no difference with integrating other chains. Note that other than observation extractor and rosen chains class which are explained above, other packages are under `Evm` alias (e.g., `EvmRpcScanner` should be used for Binance) and the chain name and id should be passed to it.

### Bitcoin Fork
_This section is unavailable for now; since Bitcoin modules are not implemented in an abstract matter. It will be updated after refactoring Bitcoin modules for this purpose._


## Extending Networks
[**Documentation on adding a new network API**](./new-network.md)
