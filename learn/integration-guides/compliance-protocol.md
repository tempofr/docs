---
title: Compliance Protocol
---

# Compliance Protocol

Complying with Anti-Money Laundering (AML) laws requires financial institutions (FIs) to know not only who their customers are sending money to but who their customers are receiving money from. In some jurisdictions banks are able to trust the AML procedures of other licensed banks. In other jurisdictions each bank must do its own sanction checking of both the sender and the receiver. 
The Compliance Protocol handles all these scenarios.

The customer information that is exchanged between FIs is flexible but the typical fields are:
 - Full Name
 - Date of birth
 - Physical address
 
The Compliance Protocol is an additional step after [federation](https://www.stellar.org/developers/learn/concepts/federation.html). In this step the sending FI contacts the receiving FI to get permission to send the transaction. To do this the receiving FI creates an `AUTH_SERVER` and adds it's location to the [stellar.toml](https://www.stellar.org/developers/learn/concepts/stellar-toml.html) of the FI.

## AUTH_SERVER

The AUTH_SERVER provides one endpoint that is called by a sending FI to get approval to send a payment to one of the receiving FI's customers. It is also used by the sender to follow up if the initial request wasn't able to complete in real time, i.e. either `info_status` or `tx_status` returned pending. The sender will simply call the AUTH_SERVER endpoint after the suggested time has past.

HTTP POST to `https://AUTH_SERVER?data=<json>&sig=<sender sig of data>`


**data** is a block of JSON that contains the following fields:

Name | Description 
-----|------
sender | The stellar address of the customer that is initiating the send.
need_info | If the caller needs the recipient's AML info in order to send the payment.
tx |  The transaction that the sender would like to send in XDR format. This transaction is unsigned.
memo | The full text of the memo the hash of this memo is included in the transaction. The **memo** field follows the [Stellar memo convention]() and should contain at least enough information of the sender to allow the receiving FI to do their sanction check.

**sig** is the signature of the data block made by the sending FI. The receiving institution should check that this signature is valid against the public signature key that is posted in the sending FI's [stellar.toml](https://www.stellar.org/developers/learn/concepts/stellar-toml.html).


#### Reply
Will return a JSON object with the following fields:

Name | Description
----|-----
info_status | If this FI is willing to share AML information or not. {ok, denied, pending}
tx_status | If this FI is willing to accept this transaction. {ok, denied, pending}
dest_info | *(only present if info_status is ok)* JSON of the recipient's AML information. in the Stellar memo convention
pending | *(only present if info_status or tx_status is pending)* Estimated number of seconds till the sender can check back for a change in status. The sender should just resubmit this request after the given number of seconds.

*Reply Example*
```
{
    info_status: "ok",
    tx_status: "pending",
    dest_info: {
        type: "encrypt",
        value: "TGV0IHlvdXIgaG9wZXMsIG5vdCB5b3VyIGh1cnRzLCBzaGFwZSB5b3VyIGZ1dHVyZS4="
    },
    pending: 3600
}
```


----



## Example of Flow
In this example, Aldi `aldi*bankA.com` wants to send to Bogart `bogart*bankB.com`

**1) BankA gets the info needed to interact with BankB**

This is done by looking up BankB's `stellar.toml` file.

BankA  -> fetches `bankB.com/.well-known/stellar.toml`

from this .toml file it pulls out the following info for BankB:
 - FEDERATION_SERVER
 - AUTH_SERVER
 - ENCRYPTION_KEY
 - Needed AML fields? 


**2) BankA gets the routing info for Bogart so it can build the transaction**

This is done by asking BankB's federation server to resolve `bogart*bankB.com`.

BankA -> `https://FEDERATION_SERVER?type=name&q=bogart*bankB.com[&simple_aml=true]`

See [Federation](https://www.stellar.org/developers/learn/concepts/federation.html) for a complete description. The returned fields of interest here are:
 - Stellar AccountID of Bogart's FI
 - Bogart's routing info
 - Need Auth flag that says whether BankB needs to authorize the transaction or not.
 - aml_yes *only returned if simple_aml is true*


**3) BankA makes the Auth Request to BankB**

This request will ask BankB for Bogart's AML info and for permission to send to Bogart.

BankA -> `https://AUTH_SERVER?data=<json>&sig=<bankA sig of data>`

Example data JSON
```
{
    sender: "aldi*bankA.com",
    need_info: "true",
    tx: <base64 encoded xdr of the tx>,
    memo: 
    {
        type: "encrypt"
        value: encrypted(
        {
            route: <Bogart's routing info>
            sender_info:
            {
                stellar: aldi*bankA.com
                name: Aldi Dobbs
                address: <blah>
            }
        })
    }
}
```

**4) BankB handles the Auth request**

 - BankB -> fetches `bankA.com/.well-known/stellar.toml` 
   From this it gets BankA's ENCRYPTION_KEY and SIGNING_KEY
 - BankB verifies the signature on the Auth Request was signed with BankA's SIGNING_KEY
 - BankB does its sanction check on Aldi. This determines the value of `tx_status`. 
 - BankB makes the decision to reveal the AML info of Bogart or not based on the following:
   - Bogart has made their info public
   - Bogart has allowed BankA
   - Bogart has allowed Aldi
   - BankB has allowed BankA 
 - If none of the above criteria are met, BankB should ask Bogart if he wants to reveal this info to BankA and accept this payment. In this case BankB will return `info_status: "pending"` in the Auth request reply to give Bogart time to accept the payment or not.
 - If BankB determines it can share the AML info with BankA, it uses BankA's ENCRYPTION_KEY to encrypt Bogart's info and sends this encrypted dest_info back with the reply.

See [Auth Request](#Auth Request) for potential return values. 

**5) BankA handles the reply from the Auth request**

If the call to the AUTH_SERVER returned `pending`, BankA must resubmit the request again after the estimated number of seconds.

BankA -> `https://AUTH_SERVER?data=<json>&sig=<bankA sig of data>`


**6) BankA does the sanction checks**

Once BankA has been given the `dest_info` from BankB, BankA does the sanction check using this AML info of Bogart. If the sanction check passes, BankA signs and submits the transaction to the Stellar network.


**7) BankB handles the incoming payment.**

 - It checks the transaction hash against a cache it has or redoes the sanction check on the sender.
 - It credits Bogart's account with the amount sent or sends the transaction back.


