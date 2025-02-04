---
id: verification-api-guide
title: Verification
sidebar_label: Verification
description: "How the proof verification works."
keywords: 
  - docs
  - polygon
  - id
  - verification
  - circuit
image: https://wiki.polygon.technology/img/thumbnail/polygon-id.png
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

After having presented an [Auth Request](./request-api-guide.md) to the user's wallet, it will process the request and generate a proof that is sent back to the Verifier. 
The proof must be verified in order to authenticate the user. 
Let us see how to implement this verification.

:::note
The proof verification always follows the same flow regardless of the request type presented in the previous step by the Verifier.
:::

### Unpack the proof

<Tabs
  defaultValue="golang"
  values={[
    { label: 'Golang', value: 'golang', },
    { label: 'Javascript', value: 'javascript', },
  ]
}>

<TabItem value="golang">

#### GoLang

```go
import (
    "io"
)
tokenBytes, err := io.ReadAll(req.Body)
```

</TabItem>
<TabItem value="javascript">

#### Javascript

```js
const getRawBody = require('raw-body')
const raw = await getRawBody(req);
const tokenStr = raw.toString().trim();
```

`req` is the post request sent by the wallet in response to the Auth Request posed by the Verifier. 
Unpacks the `req` 

</TabItem>
</Tabs>

###  Initiate the verifier

Returns an instance of a Verifier. To set up a verifier, you need to pass different parameters: 

-  `keyDIR` is the path where the public verification keys for iden3 circuits are located (such as `"./keys"`). The verification key folder can be found <a href="https://github.com/iden3/tutorial-examples/tree/main/verifier-integration/keys" target="_blank">here</a>.
- `rpcUrl` is the URL of your RPC node provider such as `"https://eth-mainnet.g.alchemy.com/v2/...."`.
- `contractAddress` is the address of the identity state Smart Contract. On Polygon Mainnet, it is 0xb8a86e138C3fe64CbCba9731216B1a638EEc55c8.

<Tabs
  defaultValue="golang"
  values={[
    { label: 'Golang', value: 'golang', },
    { label: 'Javascript', value: 'javascript', },
  ]
}>

<TabItem value="golang">

#### GoLang

```go
verificationKeyloader := &loaders.FSKeyLoader{Dir: keyDIR}
resolver := state.ETHResolver{
    RPCUrl:   rpcUrl,
    Contract:  contractAddress,
}
verifier := auth.NewVerifier(verificationKeyloader, loaders.DefaultSchemaLoader{IpfsURL: "ipfs.io"}, resolver)    
```

</TabItem>
<TabItem value="javascript">

#### Javascript

```js
const verificationKeyloader = new loaders.FSKeyLoader(keyDIR);
const sLoader = new loaders.UniversalSchemaLoader('ipfs.io');
const ethStateResolver = new resolver.EthStateResolver(rpcUrl, contractAddress);
const verifier = new auth.Verifier(
    verificationKeyloader,
    sLoader, ethStateResolver,
);
```
</TabItem>
</Tabs>

### Execute the verification

This verifies that the proof shared by the user satisfies the criteria set by the Verifier inside the initial request.
`authRequest` is the request previously presented to that specific user. 

<Tabs
  defaultValue="golang"
  values={[
    { label: 'Golang', value: 'golang', },
    { label: 'Javascript', value: 'javascript', },
  ]
}>

<TabItem value="golang">

#### GoLang

```go
authResponse, err := verifier.FullVerify(req.Context(), string(tokenBytes), authRequest.(protocolAuthorizationRequestMessage))
```
</TabItem>
<TabItem value="javascript">

#### Javascript

```js
let authResponse: protocol.AuthorizationResponseMessage;
authResponse = await verifier.fullVerify(tokenStr, authRequest);
```

</TabItem>
</Tabs>

## Verification - Under the Hood

The auth library provides a simple handler to extract all the necessary metadata from the proof and execute all the verifications needed. The verification procedure that is happening behind the scenes involves the following steps: 

### Zero-Knowledge Proof Verification

Starting from the circuit-specific public verification key, the proof, and the public inputs provided by the user, it is possible to verify the proof. In this case, the proof verification involves: 

- Verification of the proof contained based on the <a href="https://docs.iden3.io/protocol/main-circuits/#authentication" target="_blank">`Auth Circuit`</a>
- Verification of the proof contained based on the <a href="https://docs.iden3.io/protocol/main-circuits/#credentialatomicquerysig" target="_blank">`AtomicQuerySig Circuit`</a> or <a href="https://docs.iden3.io/protocol/main-circuits/#credentialatomicquerymtp" target="_blank">`AtomicQueryMTP`</a> based on the query.

### Verification of On-chain Identity States

Starting from the Identifier of the user, the <a href="https://docs.iden3.io/contracts/state" target="_blank">State</a> is fetched from the blockchain and compared to the state provided as input to the proof; this is done to check whether the user is the actual "owner" of the state used to generate the proof or not. It is important to note here that there is no gas cost associated with the verification as the VerifyState method just reads the identity state of the user on-chain without making any operations/smart contract calls. The same verification is performed for the Issuer's Identity State.

In this part, it is also verified that the claim of the query has not been revoked by the Issuer.

### Verification of Circuit Public Inputs

This involves a verification based on the public inputs of the circuits used to generate the proof. These must match the rules requested by the Verifier inside the Auth Request. For example, the query and the claim schema used by the user to generate the proof must match the Auth Request:

  - The message signed by the user is the same must match the one passed to the user inside the auth request.
  - The rules such as the `query` or the claim `schema` used to generate the proof must match the ones included inside the auth request. 
  
This "off-circuit" verification is important because a user can potentially modify the query and present a valid proof. A user born after 2000-12-31 shouldn't pass the check. But if they generate a proof using a query input `"$lt": 20010101`, the Verifier would see it as a valid proof. By doing verification of the public inputs of the circuit, the Verifier is able to detect malicious actors.

> At the end of the workflow:

> - The web client is able to authenticate the user using its identifier `ID` after having established that the user controls that identity and satisfies the query presented in the auth request.
> - The user is able to log into the platform without disclosing any personal information to the client except for its identifier.

