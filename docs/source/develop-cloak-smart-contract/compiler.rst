=============================
Cloak Compiler
=============================


--------------------
Compile
--------------------

In the development phase, developers first annotate privacy invariants in Solidity smart contract intuitively to get Cloak smart contract.
**Annotation Checker** checks the annotation to make sure the privacy invariants are correct.
The core of the development phase is **Cloak Engine**, in which the **Code Generator**, **Policy Generator**, and **Compilation Core** generate *verifier contract*, *private contract*, and *privacy policy*.


Annotate Privacy Invariants
==============================
Developers could annotate variable owner in the declaration statement to one of the {``all``, ``me``, ``id``, ``tee``}.

* ``all``, means public;

*  ``me``, means the ``msg.sender``;

* ``id``, is declared variable in type address;

* ``tee``, means any registered address of SGX with Cloak runtime.

With Cloak, users could intuitively specify the MPT as a Cloak smart contract, the *.cloak* file shows in the following contract.

.. code-block ::

   contract SupplyChain {

       mapping(address !k => uint @k) balances;
       uint @all mPrice;

       function biddingProcure(
           address[!p] parties,
           uint[@p] bids,
           address tenderer
       ) public
       returns (address winner, uint @winner sPrice) {
           winner = parties[0];
           uint mPrice = reveal(bids[0], all);
           sPrice = reveal(bids[0], winner);
           for (uint i = 1; i < parties.length; i++) {
               if (bids[i] < mPrice) {
                   winner = parties[i];
                   sPrice = mPrice;
                   mPrice = bids[i];
               } else if (bids[i] < sPrice) {
                   sPrice = bids[i];
               }
           }
           balances[tenderer] -= sPrice;
           balances[winner] += sPrice;
       }
   }

* In line 1, the developer could declare the key of balances as a temporary variable ``k``, then specifies the corresponding value is owned by the account with address ``k``, *e.g.*, ``balances[tenderer]`` is only known by the tenderer in line 23.
* In line 2, the developer specifies ``mPrice`` should be public. 
* In lines 6-7, to handle an uncertain number of suppliers, the developer declares owners ``p`` and specifies the owners’ owned data separately in two dynamic arrays. 
* In line 10, the return value ``sPrice`` is owned by the winner.
* In lines 12-13, the developer ``reveal`` private data to another owner, which is forced by Cloak to avoid unconsciously leaking privacy.
* In lines 14-24, it computes the lowest price, the second-lowest price, and the winner. The computation is based on the operation between private data from different parties, *e.g.*, ``bids[i] < sPrice``, ``balances[tenderer] += sPrice``.


Annotation Checker
====================
Taking a Cloak smart contract, Cloak ignores the annotation to checks the Solidity validation first.
Then, Cloak builds an **Abstract Syntax Tree(AST)** for further analysis.
It infers *data owner* and checks the *privacy invariants*. 
It traversals the AST in post-order and updates each parent node’s owner :math:`op = o_l \cup o_r`.
The :math:`o_l` and :math:`o_r` is the owner set of the left and right child node respectively.
Cloak recognizes a function as an MPT if :math:`TEE \in o` or :math:`|o \ \{all\}| ≥ 2`.
The latter means the function will take private data from different parties.
Then, Cloak checks privacy invariants consistency.
For example, Cloak prohibits developers from implicitly assigning their private data to variables owned by others.


Policy Generator
====================
With checked AST, Policy Generator generates a privacy policy :math:`P` for the contract.
:math:`P` simplifies and characterizes the privacy invariants. Typically, :math:`P` includes variables with data type and owners. It also includes ABI, a read-write set of each function.
Specifically, :math:`P` records each function’s characteristics from four aspects, *inputs*, *read*, *mutate* and *return*. The *inputs* includes its parameters with specified data type and owner; *read* records state variables the function read in execution; *mutate* records the contract states it mutated; *return* records the return variables.
Since :math:`P` has recorded the details of state variables in the head, *e.g.*, data type and owner, **Policy Generator** leaves the variable identities in *read*, *mutate* and *return*.

The generated privacy policy is as follows:

.. code-block::

    {
        "policy": {
            "contract":"SupplyChain",
            "states": [{
                "name": "balances",
                "type": "mapping(address=>uint256)",
                "owner": "mapping(address!x=>uint256@x)"
            }],
            "functions": [{
                "name": "settleReceivable",
                "inputs": [{
                    "name": "payee",
                    "type": "uint256",
                    "owner": "all"
                }, {
                    "name": "amount",
                    "type": "uint256",
                    "owner": "tee" 
                }],
                "read": [{
                    "name": "balances"
                    "keys": [
                        "payee", 
                    ]}, 
                ],
                "mutate": [{
                    {
                        "name": "balances",
                        "keys": [
                            "msg.sender"
                        ]
                    },
                }],
                "outputs": [{
                    "name": "",
                    "type": "uint256",
                    "owner": "all"
                }]
            }]
        }
    }

* contract, indicates the name of the confidential smart contract.

* states 

    States records all types of contract data state variables, The meaning of the ``owner`` field is

    * ``owner: "all"`` is defaults value, means that anyone can query the data and store it on Block Chain in plaintext.

    * ``owner: id``, means that the owner of data is ``id``, ``id`` type is ``address``. 
      Only user has verified the identity of the ``id`` (e.g., digital signature) can be allowed to read the data. 
      Therefore, the value of data is private and crypted it before export Cloak (e.g., synchronized data to Blockchain).

    * ``owner: "mapping(address!x=>uint256@x)``, statement of the mapping ``key`` is temporary variable ``x``, 
      and flag the owner of ``value`` is ``x``. the same as ``id``.

    .. note ::

        Temporary variable ``x`` is only valid in the mapping declaration, e.g., in a contract, 
        allow ``mapping(address!x => uint256@x)`` and ``mapping(address!x => mapping(address => uint256@x))`` can be valid 
        at the same time, because the scope of ``x`` is limited to their respective mapping.

* functions

    functions is an array collection, mark the inputs and outputs expressions of a single function, as shown below

    * ``name``, is a name of function

    * ``inputs``, input parameters of the function, each input contains the variable ``name``, ``type``, and ``owner`` of the parameter

    * ``read``, record the name of the contract data state variable required in current function contract code, in order to synchronize data
      with Block Chain.

    * ``mutate``, the contract data state binding relationship of owner of data ``id`` in this function.

    * ``outputs``, output function execution result in EVM.


Code Generator
====================
**Code Generator** generates a private contract :math:`F` and a verifier contract :math:`V`.
While leaving the computation logic in :math:`F`, **Code Generator** generates :math:`V` to verify the result and update the state.
In :math:`V`, **Code Generator** first imports a pre-deployed Cloak TEE registration contract, which holds a list of registered SGXs with Cloak runtime.
Then Cloak transforms each MPT function in *.cloak* into a new function in *V*, which verifies the MPT proof *p* and assigns new state :math:`C(s')` later.



--------------------
Debug
--------------------


