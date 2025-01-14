=================================
Quick Start
=================================

---------------
Prerequisites
---------------
Before using Cloak, you need to know Cloak is a framework that includes a
Cloak Service and a Cloak language compiler as
`Introduction <https://oxhainan-cloak-docs.readthedocs-hosted.com/en/latest/started/introduction.html>`__
described, in a normal case, we will provide a test Cloak Service, though
you can deploy Cloak Service for yourself, check `deploy Cloak
Service <https://oxhainan-cloak-docs.readthedocs-hosted.com/en/latest/tee-blockchain-architecture/initialize-cloak-network-on-blockchain.html>`__.

---------------
Installation
---------------
To install compiler, there are two ways, the easiser way is to use docker:

.. code:: 

   docker pull plytools/circleci-compiler:v0.2.0

Or install it to any host that you want, Cloak Compiler is implemented by
Python 3, so you need to prepare an environment that includes an executable
Python 3 and pip3, and its version is at least greater than 3.8.

Clone code:

.. code:: 

   git clone https://github.com/OxHainan/cloak-compiler.git
   cd cloak-compiler

Setup:

.. code:: 

   python3 setup.py develop


--------------------
Cloak by Examples
--------------------
Here we will show you how to compile, deploy and call a Cloak smart contract through `demo.cloak <https://oxhainan-cloak-docs.readthedocs-hosted.com/en/latest/index.html>`__:

.. code-block::

    // SPDX-License-Identifier: Apache-2.0

    pragma cloak ^0.2.0;

    contract Demo {
        final address@all _manager;                  // all
        mapping(address => uint256) pubBalances;     // public
        mapping(address!x => uint256@x) priBalances; // private

        constructor(address manager) public {
            _manager = manager;
            pubBalances[manager] = 1000;
        }

        //  CT-me
        //
        // @dev Deposit token from public to private balances
        // @param value The amount to be deposited.
        //
        function deposit(uint256 value) public returns (bool) {
            require(value <= pubBalances[me]);
            pubBalances[me] = pubBalances[me] - value;
            priBalances[me] = priBalances[me] + value;
            return true;
        }

        //  MPT
        //
        // @dev Transfer token for a specified address
        // @param to The address to transfer to.
        // @param value The amount to be transferred.
        //
        function multiPartyTransfer(address to, uint256 value)
            public
            returns (bool)
        {
            require(value <= priBalances[me]);
            require(to != address(0));

            priBalances[me] = priBalances[me] - value;
            priBalances[to] = priBalances[to] + value;
            return true;
        }
    }

For demonstrating the demo.cloak, We use the following test account as an example.

.. code::

   private key: 0x55b99466a43e0ccb52a11a42a3b4e10bfba630e8427570035f6db7b5c22f689e
   address: 0xDC8FBC8Eb748FfeBf850D6B93a22C3506A465beE

Compile Cloak Contract
**********************

.. code:: 

    python cloak/__main__.py compile -o output demo.cloak

There are three important files in the output directory, including public_contract.sol, private_contract.sol and policy.json.

* public_contract.sol: a solidity contract, it will be deployed to Blockchain.
* private_contract.sol: a solidity contract, it will be deployed to cloak-tee and be executed by eEVM in TEE environment.
* policy.json: privacy policy definition of the Cloak smart contract binding to the private contract.

Deploy Public Contract
***********************
Web3 is a recommended tool for deploying the public contract to the blockchain.  For convenience, cloak-complier provides a command to complete it.

.. code::

    python cloak/__main__.py deploy <compiled output dir> <args...>  --blockchain-backend w3-ganache --blockchain-node-uri http://127.0.0.1:8545 --blockchain-pki-address <PKI Address> --blockchain-service-address <cloak service address>

`<args...>` option is the constructor function arguments. In this example, it is *0xDC8FBC8Eb748FfeBf850D6B93a22C3506A465beE*.

We have writed a `sample <https://github.com/OxHainan/cloak-client/tree/main/samples/demo>`__ that uses cloak-client to show you how to register pk, deploy private contract, bind privacy policy and send MPT, *etc*.
Next, we will step by step go through the Cloak transaction workflow based on the sample.

Register Public Key
***********************
Before executing an MPT, if you are the owner of some state data (*e.g.*, _manager in Demo contract),
you need to register your public key to the PKI contract,
and the public key must be specified by a standard PEM format.
Here is an example that get a PEM-format public key from hex-string private key:

.. code::

    echo 302e0201010420 <PRIVATE KEY> a00706052b8104000a | xxd -r -p | openssl ec -inform d -pubout

Replace <PRIVATE KEY> with `55b99466a43e0ccb52a11a42a3b4e10bfba630e8427570035f6db7b5c22f689e` and execute:

.. code::

   ➜  ~ echo 302e0201010420 55b99466a43e0ccb52a11a42a3b4e10bfba630e8427570035f6db7b5c22f689e a00706052b8104000a | xxd -r -p | openssl ec -inform d -pubout
   read EC key
   writing EC key
   -----BEGIN PUBLIC KEY-----
   MFYwEAYHKoZIzj0CAQYFK4EEAAoDQgAEXFZ6txDg9knTl5E5mFnj7G1j9x91x5d9
   MPCYiA6CoewqASjNGc8orVGol8ajLiz3rnleoSm2OyoPsM/3KXdrMg==
   -----END PUBLIC KEY-----

Based on it, in our demo sample, registering pk to blockchain works as following:

.. code::

   async function get_pem_pk(account) {
      const cmd = `echo 302e0201010420 ${account.privateKey.substring(2,)} a00706052b8104000a | xxd -r -p | openssl ec -inform d -pubout`
      const {stdout,} = await p_exec(cmd)
      return stdout.toString()
   }

   async function register_pki(web3, account) {
     const pki_file = compile_dir + "/CloakPKI.sol"
     const [abi, ] = await get_abi_and_bin(pki_file, "CloakPKI")
     var pki = new web3.eth.Contract(abi, pki_address)
     var tx = {
         to: pki_address,
         data: pki.methods.announcePk(await get_pem_pk(account)).encodeABI(),
         gas: 900000,
         gasPrice: 0,
     }
     var signed = await web3.eth.accounts.signTransaction(tx, account.privateKey)
     return web3.eth.sendSignedTransaction(signed.rawTransaction)
   }

Cloak Web3
***********************
Cloak-client wraps a Web3 Provider, so you can create a web3 object and create _manager account like:

.. code::

    const httpsAgent = new Agent({
        rejectUnauthorized: false,
        ca: readFileSync(args[0]+"/networkcert.pem"),
        cert: readFileSync(args[0]+"/user0_cert.pem"),
        key: readFileSync(args[0]+"/user0_privk.pem"),
    });

    var web3 = new Web3()
    web3.setProvider(new CloakProvider("https://127.0.0.1:8000", httpsAgent, web3))
    const acc_1 = web3.eth.accounts.privateKeyToAccount("0x55b99466a43e0ccb52a11a42a3b4e10bfba630e8427570035f6db7b5c22f689e");

`https://127.0.0.1:8000` is cloak-tee service host and port.
Because of encryption, cloak-tee can only accept https request, you need to provide the network.pem of Cloak Network as CA, and a trusted user(cert and pk), 
`args[0]` is the directory of the three files. If you use cloak.py setup your cloak-tee, it will be workerspace/sanbox_common under cloak-tee build directory.

Deploy Private Contract
************************
Deploy private contract is more like standard web3 except the web3 object is wrapped by ``CloakProvider``:

.. code::

    async function get_abi_and_bin(file, name) {
        const cmd = `solc --combined-json abi,bin,bin-runtime,hashes --evm-version homestead --optimize ${file}`
        const {stdout,} = await p_exec(cmd)
        const j = JSON.parse(stdout)["contracts"][`${file}:${name}`]
        return [j["abi"], j["bin"]]
    }

    async function deployContract(web3, account, file, name, params) {
        const [abi, bin] = await get_abi_and_bin(file, name)
        var contract = new web3.eth.Contract(abi)
        return contract.deploy({data: bin, arguments: params}).send({from: account.address})
    }


Bind Privacy Policy
************************

.. code::

    const code_hash = web3.utils.sha3(readFileSync(code_file))
    await web3.cloak.sendPrivacyTransaction({
        account: acc_1,
        params: {
            to: deployed.options.address,
            codeHash: code_hash,
            verifierAddr: public_contract_address,
            data: web3.utils.toHex(readFileSync(policy_file))
        }
    })

Send Deposit Transaction
*************************
The deposit function stores the balance to private mapping from the public contract, the proposer and participant are the same (so-called CT).

.. code::

    // deposit
    var mpt_id = await web3.cloak.sendMultiPartyTransaction({
        account: acc_1,
        params: {
            nonce: web3.utils.toHex(100),
            to: deployed.options.address,
            data: {
                "function": "deposit",
                "inputs": [
                    {"name": "value", "value": "100"},
                ]
            }
        }
    })

Get Transaction Result
**************************

.. code::

    // wait 3 second
    await new Promise(resolve => setTimeout(resolve, 3000));
    web3.cloak.getMultiPartyTransaction({id: mpt_id}).then(console.log).catch(console.log)

After sending a CT/MPT transaction to Cloak Network, it will return an MPT ID, you can use that id to check the transaction status.
We provide a function that loops to get status until MPT finished later.

Multi Party Transfer
**************************
This code shows how to propose an MPT and how to participate that MPT:

.. code::

    // multi party transfer
    const acc_2 = web3.eth.accounts.create();
    await register_pki(ganache_web3, acc_2)

    var mpt_id = await web3.cloak.sendMultiPartyTransaction({
        account: acc_1,
        params: {
            nonce: web3.utils.toHex(100),
            to: deployed.options.address,
            data: {
                "function": "multiPartyTransfer",
                "inputs": [
                    {"name": "value", "value": "10"},
                ]
            }
        }
    })

    await web3.cloak.sendMultiPartyTransaction({
        account: acc_2,
        params: {
            nonce: web3.utils.toHex(100),
            to: mpt_id,
            data: {
                "function": "multiPartyTransfer",
                "inputs": [
                    {"name": "to", "value": acc_2.address},
                ]
            }
        }
    })

For more detail usage of cloak-client, please check: 
`<https://oxhainan-cloak-docs.readthedocs-hosted.com/en/latest/deploy-cloak-smart-contract/deploy.html#cloak-client>`__,
the full sample code: `<https://github.com/OxHainan/cloak-client/tree/main/samples/demo>`__

