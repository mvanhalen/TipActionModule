# TipActionModule

The `TipActionModule` is a simple Open Action module that allows users to tip the author of a publication. It's based on the "Creating an Open Action" tutorial from the [Lens Docs](https://docs.lens.xyz/docs/creating-a-publication-action) and was built with [Scaffold-Lens](https://github.com/iPaulPro/scaffold-lens)

Additions to the tutorial include:
- ✅ Adds compliance with the Open Action [Module Metadata Standard](https://docs.lens.xyz/docs/module-metadata-standard)
- ✅ Checks token allowance before attempting to send a tip
- ✅ Uses `SafeERC20` to transfer tokens
- ✅ Uses `ReentrancyGuard` to prevent reentrancy attacks

## Using the TipActionModule Contract

To use the live `TipActionModule` you can use the address and metadata below:

| Network | Chain ID | Deployed Contract                                                                                                               | Metadata                                                                     |
|---------|----------|---------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| Mumbai  | 80001    | [0x6111e258a6d00d805DcF1249900895c7aA0cD186](https://mumbai.polygonscan.com/address/0x6111e258a6d00d805DcF1249900895c7aA0cD186) | [link](https://gateway.irys.xyz/WzHPiYtDn5jYb7tO6pi13lNg5ZlcglPdrosoHDAA8co) |

The `TipActionModule` contract can be used as an Open Action Module on Lens Protocol.

The initialize calldata ABI is

```json
[
  { 
    "type": "address", 
    "name": "tipReceiver"
  }
]
```

And the process calldata ABI is

```json
[
    {
      "type": "address",
      "name": "currency"
    },
    {
      "type": "uint256",
      "name": "tipAmount"
    }
  ]
```

Where `currency` is the address of the ERC20 token to use for tips, and `tipAmount` is the amount of tokens to send in wei.

**NOTE:** The token used for tipping must be registered with the Lens Module Registry.

### Using the TipActionModule with the Lens SDK

You can initialize the action and set the tip receiver using the Lens SDK Client.

```typescript
import { type LensClient, type OnchainPostRequest, encodeData } from '@lens-protocol/client';

const openActionContract = "0x6111e258a6d00d805DcF1249900895c7aA0cD186";

const postRequest: OnchainPostRequest = { contentURI };

// The initialization calldata accepts a single address parameter, the tip receiver
const calldata = encodeData(
  [{ name: "tipReceiver", type: "address" }],
  [tipReceiver],
);

postRequest.openActionModules.push({
  unknownOpenAction: {
    address: openActionContract,
    data: calldata,
  },
});

await lensClient.publication.postOnchain(postRequest);
```

To support executing a tip action, you can create an `act` transaction as usual, supplying the currency and amount to tip as the process call data.

```typescript
const tipActionContract = "0x6111e258a6d00d805DcF1249900895c7aA0cD186";

// get the module settings and metadata
const settings = post.openActionModules.find(
  (module) => module.contract.address.toLowerCase() === tipActionContract.toLowerCase(),
);
const metadataRes = await lensClient.modules.fetchMetadata({
  implementation: settings.contract.address,
});

// encode calldata
const processCalldata: ModuleParam[] = JSON.parse(metadataRes.metadata.processCalldataABI);
const calldata = encodeData(
  processCalldata,
  [currency, amount.toString()],
);

// create the act transaction request
const request: ActOnOpenActionRequest = {
  actOn: {
    unknownOpenAction: {
      address: tipActionContract,
      data: calldata,
    },
  },
  for: post.id,
};

const tip = await lensClient.publication.actions.createActOnTypedData(request);
// sign and broadcast transaction
```

The tip receiver address can be obtained from the module settings:
```typescript
if (settings.initializeCalldata) {
  const initData = decodeData(
    JSON.parse(metadataRes.metadata.initializeCalldataABI) as ModuleParam[],
    settings.initializeCalldata,
  );
  // This is the tip receiver address (registered in the initialize calldata)
  const tipReceiver = initData.tipReceiver;
}
```

### Important Notes

1. Clients implementing the tip action should ensure the user has approved the tip amount for the tip currency before attempting to execute the tip action.

   You can check the allowance using the ERC-20 `allowance` function directly, or use the Lens SDK Client:

    ```typescript
    import type { LensClient, ApprovedModuleAllowanceAmountRequest } from '@lens-protocol/client';
    import { BigNumber, ethers } from 'ethers';
   
    const needsApproval = async (
      lensClient: LensClient,
      currency: { address: string, decimals: number },
      tipAmount: BigNumber,
      actionModule: string,
    ): Promise<boolean> => {
      const req: ApprovedModuleAllowanceAmountRequest = {
        currencies: [currency.address],
        unknownOpenActionModules: [actionModule],
      };
      
      const res = await lensClient.modules.approvedAllowanceAmount(req);
      if (res.isFailure()) return true;
      
      const allowances = res.unwrap();
      if (!allowances.length) return true;
    
      const valueInWei = ethers.utils.parseUnits(
        allowances[0].allowance.value, 
        currency.decmals,
      );
      return tipAmount.gt(valueInWei);
    };
    ```
---

## About Scaffold-Lens

Scaffold-Lens is an open-source toolkit, based on Scaffold-ETH 2, made by Paul Burke for building Lens Smart Posts and Open Actions dapps.

Learn more about Scaffold-Lens and read the docs [here](https://github.com/iPaulPro/scaffold-lens).