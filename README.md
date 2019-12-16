# How to Build a Libra Wallet with 190 Lines of Code

[Libra](https://libra.org) is a global digital currency launched by Facebook. This article shows how to write a simple Libra web wallet with 190 lines of html+javascript code. The following image is the screenshot of the wallet's main page：
![libra wallet screenshot](docs/screenshot.png "libra wallet screenshot")

The web interface of the wallet is built using the classic bootstrap+jquery technology, and the wallet-related logic is implemented by calling [MoveonLibra](https://www.moveonLibra.com)'s OpenAPI. The online access url of the wallet is [https://www.moveonlibra.com/wallet.html](https://www.moveonlibra.com/wallet.html)

## 1. Create Wallet
If you have a wallet, the wallet is stored in the `localStorage` of the browser. Each time the page has finished loading, we try to load it from `localStorage`. If the wallet  doesn't exists, request to create a new wallet：
```javascript
    $(document).ready(function () {
      init_wallet_and_account();
    });
    async function init_wallet_and_account() {
      if (!localStorage.getItem('wallet')) {
        $('#createWalletModal').modal()
      }else{
          ...
      }
    }
```
If program can't find the wallet, a bootstrap modal dialog pops up, prompting the user to enter the name of her wallet：
```html
  <div class="modal fade" id="createWalletModal" tabindex="-1" role="dialog">
      ...
        <div class="modal-body">
          <p>This is your first time to use this libra wallet app. You need to create a wallet of yourself first:</p>
          <input id="wallet_name" type="text" class="form-control" autofocus="true"
            placeholder="Please input the name of wallet">
        </div>
        <div class="modal-footer">
          <button type="button" class="btn btn-primary" onclick="clickCreateWallet()">Create</button>
        </div>
  </div>
```
When the user enters the name of her wallet and clicks the "Create" button, following  method will be execute to create the wallet:
```javascript
async function clickCreateWallet() {
      $('#createWalletModal').modal('hide');
      var name = $("#wallet_name").val()
      $('#wallet_name_tag').html(name)
      $("#wait_tx").css('visibility','visible')
      wallet = await call_api("/v1/wallets", { "name": name }, 'POST')
      localStorage.setItem('wallet', JSON.stringify(wallet))
      account = await call_api("/v1/wallets/" + wallet.wallet_id + "/accounts", {}, 'POST')
      localStorage.setItem('account', JSON.stringify(account))
      $("#wait_tx").css('visibility','hidden')
    }
```
The workflow of function `clickCreateWallet` is as follows:

* Close the modal dialog
* Get the wallet name entered by the user and display it on the page
* Show a waiting icon for wallet creation
* Call MoveOnLibra's API to create the wallet, API address is "/v1/wallets"
* When the wallet is created successfully, store it to the localStorage
* Call MoveOnLibra's API to create an account, API address is "/v1/wallets/<wallet_id>/accounts"
* When the account is created successfully, store it to the localStorage
* Cancel the display of icons in waiting

The wallet is compatible with the [official rust implementation](https://github.com/libra/libra/tree/master/client/libra_wallet) of the wallet.
Each wallet has a `mnemonic` which adheres to the [`BIP39`](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) spec. 


A wallet can have multiple sub-accounts, created by deterministic key derivation, using an algorithm compaible with Libra cli, similar to but different from BIP32.



## 2. Encapsulating MoveOnLibra's Rest API with Promise
Javascript's ajax calls are asynchronous by default, but the wallet's business logic needs to be written synchronously, so we encapsulate Jquery's ajax call and return a Promise so that we can write code in async/await style. For example:
```javascript
    wallet = await call_api("/v1/wallets", { "name": name }, 'POST')
    localStorage.setItem('wallet', JSON.stringify(wallet))
    account = await call_api("/v1/wallets/" + wallet.wallet_id + "/accounts", {}, 'POST')
```
In above code, you must wait for the wallet to be created before you can create an account under the wallet. The implementation of 'call_api' is as follows：
```javascript
    function call_api(url, data, method = "GET") {
      const appkey = "...";
      const host = "https://apitest.moveonLibra.com";
      return new Promise(function (resolve, reject) {
        jQuery.ajax({
          url: host + url,
          headers: { "Authorization": appkey },
          data: data, method: method
        })
          .done(function (data) {resolve(data)})
          .fail(function (jqXHR, textStatus, error) {reject(error)})
      })
    }
```
For the technical overview of async/await, you can refer to this article [javascript-from-callbacks-to-async-await](https://www.freecodecamp.org/news/javascript-from-callbacks-to-async-await-1cc090ddad99/)。

### About API Documentation
The complete documentation for MoveOnLibra API is available [here](https://www.moveonlibra.com/apidoc.html). For example:

* `Create Wallet`, api doc is https://www.moveonlibra.com/apidoc.html#operation/create_wallet
* `Create Account`, api doc is https://www.moveonlibra.com/apidoc.html#operation/create_accounts

### About API Authorization
Access to MoveOnLibra's wallet API requires authorization. You need to [sign up](https://www.moveonlibra.com/users/sign_up) to get your appkey.

A pre-created appkey is used in the demo code, but anyone who gets this appkey can manipulate the wallet belongs to the appkey, so the code is only for demo. 

After get the appkey, put it in the http header with name `Authorization`

| Security scheme type:  | API Key |
|---|---|
| **Header parameter name:** | Authorization |
| **Header parameter value:**  | appkey |


## 3. Display Wallet and Account
If the user already owns the wallet and account, load and display them directly：
```javascript
        wallet = JSON.parse(localStorage.getItem('wallet'))
        account = JSON.parse(localStorage.getItem('account'))
        show_wallet()
        refreshBalance()
```
The `show_wallet` function is as follows：
```javascript
    function show_wallet(){
      $('#wallet_name_tag').html(wallet.name)
      $('#address_tag').html(account.address)
      var qrcode = new QRCode("qrcode", {
        text: account.address, width: 180, height: 180,
      });
    }
```
The above code is used to display the wallet name, the address of the account, and the QR code format image of the address。

The `refreshBalance` function is as follows：
```javascript
    async function refreshBalance(){
      ret = await call_api("/v1/address/balance/"+account.address, {});
      $('#balance').html(ret.balance/1000000)
    }
```
Sync balance data from the Libra blockchain every time the wallet loads.

    Note：All amounts returned by API are `micro_libra`. If you want to display them as `libra`, you need to divide them by 1000000.


## 4. Mint
This wallet is connected to Libra's `testnet` which supports mint coins. When an account is newly established, the libra balance in the account is zero. For the convenience of testing, we can fill the account with some coins through the mint function.

```javascript
    async function clickMint() {
      data = {
        "number_of_micro_libra": 100*1000000,
        "receiver_account_address": account.address
      }
      $("#wait_tx").css('visibility','visible')
      tx = await call_api("/v1/transactions/mint", data, 'POST');
      $("#wait_tx").css('visibility','hidden')
      if(tx.success){
        $('#balance').html(100 + parseFloat($('#balance').html()))
      }else{
        alert("Transaction Error:"+tx.transaction_info.major_status)
      }
    }
```
The above code is relatively simple: call the "/v1/transactions/mint" API to mint coin, the default number is 100 `libra`, after the completion of mint transaction, update the account balance.

## 5. Transfer
The logic of the transfer function is similar to the mint, except that the payee and the number of coins are required. When the user wants to transfer coins, a bootstrap modal dialog pops up to provide the inputs of payee and the number of coins:
```html
  <div class="modal fade" id="transferModal" tabindex="-1" role="dialog">
      ...
        <div class="modal-body">
          <input id="receiver_address" type="text" class="form-control mb-3" autofocus="true"
            placeholder="Receiver Address in hex64 format">
          <input id="transfer_amount" type="number" class="form-control" placeholder="Number of coins">
        </div>
        <div class="modal-footer">
          <button type="button" class="btn btn-primary" onclick="clickTransfer()">Transfer</button>
        </div>
      </div>
    </div>
  </div>
```
When the user clicks the "Transfer" button to confirm the transfer, execute the following js code：
```javascript
    async function clickTransfer(receiver, micro_libra) {
      var transfer_amount = parseInt($("#transfer_amount").val())
      data = {
        "number_of_micro_libra": transfer_amount * 1000000,
        "receiver_account_address": $("#receiver_address").val(),
        "sender_account_address": account.address,
        "wallet_id": wallet.wallet_id,
      }
      $('#transferModal').modal('hide');
      $("#wait_tx").css('visibility','visible')
      tx = await call_api("/v1/transactions/transfer", data, 'POST')
      $("#wait_tx").css('visibility','hidden')
      if(tx.success){
        $('#balance').html(parseFloat($('#balance').html())-transfer_amount)
      }else{
        alert("Transaction Error:"+tx.transaction_info.major_status)
      }
    }
```
The logic of transfer is similar to that of mint, but the parameters is quite different. 

Since the transaction needs to be signed before it's been submited to the blockchain, and the private key to sign the transaction is saved in the wallet, the transfer process requires four parameters, in addition to the regular three parameters：

* `sender_account_address`, payer address
* `receiver_account_address`,  payee address
* `number_of_micro_libra`, transfer amount

An additional parameter is required:

* `wallet_id`, the id of the wallet used to sign the transfer transaction. The payer must be a sub-account of the wallet.


## 6. Account Income and Expenditure
You can query the income and expenditure of an account by checking the events sent and received of the account address. For example：

* See the latest payments, through API [get_account_events_latest_sent](https://www.moveonlibra.com/apidoc.html#operation/get_account_events_latest_sent)
* See the latest incomes, through API [get_account_events_latest_received](https://www.moveonlibra.com/apidoc.html#operation/get_account_events_latest_received)

Since there are already many Libra blockchain browsers that can view the account's details, one shortcut is to link directly to the blockchain browser to view the account's revenue and expenditure details. The code is as follows:
```javascript
<a id="explorer" href="https://explorer.moveonlibra.com/" target="_blank">Show account detail in Libra explorer</a>
    var url = $('#explorer').attr("href") + "accounts/" + account.address
    $('#explorer').attr("href", url);
```

## 7. Conclusion
This article demonstrates how a Libra wallet can be implemented with very little code. 

Source code is [here](https://github.com/MoveOnLibra/libra-wallet-demo-javascript/blob/master/wallet190.html).
This is a 191-line html file that can run on its own. You can download and open it with chrome or firefox. Because IE/Edge/Safari does not support access to `localStorage` in local files, with there browsers, you should try the [online version](https://www.moveonlibra.com/wallet.html).