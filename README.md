# Send Omnichain Messages in Remix

## Introduction

Omnichain messaging is about enabling smart contracts on different blockchains to communicate with each other, to exchange data, and to coordinate actions.

The tutorial will show you everything you need to know about sending Omnichain messages in less than 50 lines of code, giving you a great starting point for how cross-chain messaging allows for the synchronization of states, triggering of actions, and sharing of information across different blockchain ecosystems.

In this tutorial, we’ll send a message from a source blockchain to a destination blockchain, demonstrating how easy it is to use LayerZero.

1. Set up the Optimism Goerli network on MetaMask (optional)
2. Get faucet Goerli and Optimism Goerli (optional)
3. Deploy your contract on the source & destination chain
4. Wire your contracts together using `setTrustedRemote`
5. Estimate how much gas to send using `estimateFees`
6. Send a simple message to your destination chain
7. Examine the resulting transaction on LayerZero Scan

## Prerequisites

You should have a basic understanding of how the Remix IDE works and operates, as well as some familiarity writing smart contracts compatible with the Ethereum Virtual Machine.

## Set up the Optimism Goerli Network on MetaMask (optional)

- **Network Name**: Optimism Goerli
- **New RPC URL**: https://goerli.optimism.io
- **Chain Id:** 420 (not to be confused with the LayerZero ChainId we'll use later)
- **Symbol:** ETH
- **Explorer:** https://goerli-optimism.etherscan.io/

Don't forget to get some **Goerli ETH** & **OP Goerli** here:

- https://community.optimism.io/docs/useful-tools/faucets/#
- https://goerlifaucet.com/

## Create OmniMessage Contract

Now we can create our Omnichain messaging contract on Remix. Open Remix on your browser and create a new file under your contracts folder. This will be the same contract we deploy on both our source and destination chains.

Feel free to read through the contract below, and then copy it into your Remix file.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.17;

// This line imports the NonblockingLzApp contract from LayerZero's solidity-examples Github repo.
import "https://github.com/LayerZero-Labs/solidity-examples/blob/main/contracts/lzApp/NonblockingLzApp.sol";

// This contract is inheritting from the NonblockingLzApp contract.
contract OmniMessage is NonblockingLzApp {

    // A public string variable named "data" is declared. This will be the message sent to destination.
    string public data = "Nothing received yet";

    // A uint16 variable named "destChainId" is declared to hold the LayerZero Chain Id of the destination blockchain.
    uint16 destChainId;

    //This constructor initializes the contract with our source chain's _lzEndpoint.
    constructor(address _lzEndpoint) NonblockingLzApp(_lzEndpoint) {

        // Below is an "if statement" to simplify wiring our contract's together.
        // In this case, we're auto-filling the dest chain Id based on the source endpoint.
        // For example: if our source endpoint is Goerli, then the destination is OP-Goerli.

        // NOTE: This is to simplify our tutorial, and is not standard wiring practice in LayerZero contracts.

        // Wiring 1: If Source == OP Goerli, then Destination Chain = ETH Goerli
        if (_lzEndpoint == 0xae92d5aD7583AD66E49A0c67BAd18F6ba52dDDc1) destChainId = 10121;
        // Wiring 2: If Source == ETH Goerli, then Destination Chain = OP Goerli
        if (_lzEndpoint == 0xbfD2135BFfbb0B5378b56643c2Df8a87552Bfa23) destChainId = 10132;
    }

    // This function is called when data is received. It overrides the equivalent function in the parent contract.
    function _nonblockingLzReceive(uint16, bytes memory, uint64, bytes memory _payload) internal override {

       // The LayerZero _payload (message) is decoded as a string and stored in the "data" variable.
       data = abi.decode(_payload, (string));
    }

    // This function is called to send the data string to the destination.
    // It's payable, so that we can use our native gas token to pay for gas fees.
    function send(string memory _message) public payable {

        // The message is encoded as bytes and stored in the "payload" variable.
        bytes memory payload = abi.encode(_message);

        // The data is sent using the parent contract's _lzSend function.
        _lzSend(destChainId, payload, payable(msg.sender), address(0x0), bytes(""), msg.value);
    }


    // This function allows the contract owner to designate another contract address to trust.
    // It can only be called by the owner due to the "onlyOwner" modifier.
    // NOTE: In standard LayerZero contract's, this is done through SetTrustedRemote.
    function trustAddress(address _otherContract) public onlyOwner {
        trustedRemoteLookup[destChainId] = abi.encodePacked(_otherContract, address(this));
    }


    // This function estimates the fees for a LayerZero operation.
    // It calculates the fees required on the source chain, destination chain, and by the LayerZero protocol itself.

    // @param dstChainId The LayerZero endpoint ID of the destination chain where the transaction is headed.
    // @param adapterParams The LayerZero relayer parameters used in the transaction.
    // Default Relayer Adapter Parameters = 0x00010000000000000000000000000000000000000000000000000000000000030d40
    // @param _message The message you plan to send across chains.

    // @return nativeFee The estimated fee required denominated in the native chain's gas token.
    function estimateFees(uint16 dstChainId, bytes calldata adapterParams, string memory _message) public view returns (uint nativeFee, uint zroFee) {

        //Input the message you plan to send.
        bytes memory payload = abi.encode(_message);

        // Call the estimateFees function on the lzEndpoint contract.
        // This function estimates the fees required on the source chain, the destination chain, and by the LayerZero protocol.
        return lzEndpoint.estimateFees(dstChainId, address(this), payload, false, adapterParams);
    }
}
```

After pasting our contract and compiling the code without errors, open "Deploy & run transactions" from the sidebar.
Click "Environment" on the top-left of our screen, and select Injected Provider.

Next to the "Deploy" button, we'll need to paste the address of the LayerZero endpoint deployed on the same chain as our contract.

That means we'll need to deploy this same contract twice using their respective endpoint addresses: once on `Goerli Ethereum`, another on `Optimism Goerli`.

```solidity
Goerli LZ Endpoint: 0xbfD2135BFfbb0B5378b56643c2Df8a87552Bfa23
```

```solidity
Optimism Goerli LZ Endpoint: 0xae92d5aD7583AD66E49A0c67BAd18F6ba52dDDc1
```

(See our other testnet endpoints [here](https://layerzero.gitbook.io/docs/technical-reference/testnet/testnet-addresses) to try sending messages between other chains!)

Once you've successfully deployed OmniMessage on Goerli and Optimism Goerli, you're good to move on.

## Wire and connect your contracts together

Inside each of your newly deployed contracts, you may notice a wall of functions. Luckily, we only need to worry about one function field in particular: `trustAddress`.

Connecting your contract's together is remarkably easy with LayerZero. To wire your contracts, simply take the address of the destination contract, and use it as an input for `trustAddress`.

TIP: You'll need to wire functions both ways in order to send AND receive messages. That means calling `trustAddress` on both your Goerli and Optimism contract.

Normally this is done in LayerZero by calling the `SetTrustedRemoteAddress` function. We've abstracted this part away in the tutorial to make your life easier! As a challenge, see if you can use `SetTrustedRemoteAddress` and wire your contracts together!

Check and see if your transactions pass on each block explorer. You now should be setup to start sending cross-chain messages!

## Estimate how much gas to send

LayerZero gas requirements can vary based on your source chain, destination chain, and the payload you're attempting to send, which is why we recommend estimating fees before sending your first transaction.

To do this, we'll use the `estimateFees` function.

The purpose of this function is to estimate the fees associated with a particular LayerZero transaction using three inputs:

`dstChainId:` This is the identifier of the destination chain's endpoint where the transaction is intended to go.

`adapterParams:` This is a byte array that contains parameters for how a LayerZero relayer should transmit the transaction. Since LayerZero delivers the destination transaction when a message is sent, it must pay for that destination gas.

`_message:` This is the message you intend to send to your destination chain and contract.

BY default, 200,000 gas is priced into `adapterParams` for simplicity, encoded as a bytes array:

```solidity
// v1 adapterParams, encoded for version 1 style, and 200k gas quote
let adapterParams = ethers.utils.solidityPack(
    ['uint16','uint256'],
    [1, 200000]
)
```

The resulting `adapterParams` should look like this (34 total bytes in length):

```solidity
0x00010000000000000000000000000000000000000000000000000000000000030d40
```

NOTE: For advanced usage and further reading on Relayer Adapter Parameters, see [here](https://layerzero.gitbook.io/docs/evm-guides/advanced/relayer-adapter-parameters).

After inputting your `dstChainId`, `adapterParams`, and intended `_message`, call `estimateFees` to receive a gas fee quote denominated in the native chain's (in Wei).

We'll use this value as a quote for `msg.value` in the next section.

For further reading on `estimateFee` and best pratices for fee estimation, see here.

## Send your first Omnichain message

Finally the moment you've been waiting for: using the `send` function. Simply input a string into the `_message` field that you wish to send to your destination chain.

### Contract A

Remember to pass the `msg.value` we quoted using `estimateFee` in Remix, as we still need to pay gas fees on the source and destination, as well as for the oracle and executor who deliver the messages off-chain. Once you've successfully sent your transaction, call the `data` field from your destination contract to see your first Omnichain message!

### Contract B

Your message may take a few minutes to appear in the destination block explorer, depending on which chains you deploy to.

## Examine the transaction on LayerZero Scan

Finally, let's see what's happening in our transaction. Take your transaction hash and paste it into: https://testnet.layerzeroscan.com/

You should see `Status: Delivered`, confirming your message has been delivered to its destination using LayerZero.

**Congrats, you just sent your first Omnichain message!** 🥳

Whether it's sending a simple message on Ethereum over to Optimism, or a gaming dApp on Polygon interacting with a DAO on Avalanche, LayerZero's messaging lays the groundwork for cross-chain operation.
