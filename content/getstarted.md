Title: Get Started
Date: 2017-4-6
Category: Tutorial


# Get Started with Town Crier

For smart contracts on blockchain systems such as Ethereum, access to real-world data is critical.
A currency exchange contract must be able to learn current exchange rates.
A trip insurance contract must determine whether flights arrive on time.
A contract for the sale of a physical good needs to know whether the good was successfully delivered.
These are just a few of the many examples of applications that can only run with knowledge of real-world data or events.
A critical problem is: <i> Who can be trusted provide data to smart contracts in a trustworthy way? </i>

<! Brief idea of Oraclize and comparison with TC required >
The Town Crier (TC) system addresses this problem by using <i> trusted hardware </i>, namely the Intel SGX instruction set, a new capability in certain Intel CPUs. TC obtains data from target websites specified in queries from application contracts. TC uses SGX to achieve what we call its <i>authenticity property<\i>. <b> Assuming that you trust SGX, data delivered by TC from a website to an application contract is guaranteed to be free from tampering.</b>

This authenticity property means that to trust TC data, you only need to trust Intel's implementation of SGX and the target website. You don't need to trust the operators of TC or anyone else. Even the operators of the TC server cannot tamper with its operation or, for that matter, see the data it's processing. 

Using Town Crier is simple. To obtain data from a target website, an application contract sends a query to the Town Crier Contract, which serves as a front end for TC. This query specifies a target source, i.e., a trusted website, and some query parameters, namely the specifics of the query to the website. For example, if the requesting contract is seeking a stock quote on Oracles 'R Us Ltd., it might specify that it wants the result of sending ticker 'ORUL' to www.besteverstockquotes.com. 

Behind the scenes, when it receives a query from an application contract, the TC server fetches the requested data from the website and relays it back to the requesting contract.
The processing of the query happens inside an SGX-protected environment known as an "enclave." The requested data is fetched via a TLS connection to the target website that terminates inside the enclave. SGX protections prevent even the operator of the server from peeking into the enclave or modifying its behavior, while use of TLS prevents tampering or eavesdropping on communications on the network. 

Town Crier can optionally ingest an <i>encrypted</i> query, allowing it to handle <i> secret query data </i>. For example, a query could include a password used to log into a server or secret trading data. TC's operation in an SGX enclave ensures that the password or trading data is concealed from the TC operator (and everyone else).

Thanks to its use of SGX and various innovations in its end-to-end design, Town Crier offers several properties that other oracles cannot achieve:

<ol>
  <li>Authenicity guarantee: There's no need to trust any particular service provider(s) in order to trust Town Crier data. (You need only believe that SGX is properly implemented.) </li>
  <li>Succinct replies: Town Crier can prune target website replies in a trustworthy way to provide short responses to queries. It does not need to relay verbose website responses. Such succintness is important in Ethereum, for instance, where message length determines transaction costs. </li>
  <li>Confidential queries:</li> Town Crier can handle <i> secret </i> query data in a trustworthy way. This feature makes TC far more powerful and flexible than conventional oracles.
</ol>

For more details on TC, its implementation using SGX, and its security guarantees, please read our paper [Town Crier: An Authenticated Data Feed for Smart Contracts].

TC can provide data in any ecosystem, but its first deployment is on Ethereum.

## Understand the ```TownCrier``` Contract

The ```TownCrier``` contract provides a uniform interface for queries from and replies to an Application Contract, which we also refer to as a "Requester."
This interface consists of the following three functions.

* ```request(uint8 requestType, address callbackAddr, bytes4 callbackFID, uint256 timestamp, bytes32[] requestData) public payable returns(uint64);```

	For an application contract to call function ```request()```, it needs to send in the following parameters.
    
    - ```requestType```: the type of the request, for the Town Crier server to process it accordingly. You'll see all the request types Town Crier currently supports in the following.
    
    - ```callbackAddr```: the address of the Application Contract to which the response from Town Crier is forwarded.
    
    - ```callbackFID```: specification of the callback function in the Application contract to receive the response from TC.
    
    - ```timestamp```: parameter required for a potential feature. Currently TC only responds to requests immediately. Eventually TC will support requests with pre-specified future query times. At present, developers can ignore this parameter and just set it to 0.
    
    - ```requestData```: data specifying request parameters. The format depends on the request type.

    When this function is called, a request is logged by event ```RequestInfo()```. The function returns a ```requestId``` that is uniquely assigned to this request. The Application Contract can use the ```requestId``` to check the response or status of a request in its logs. The Town Crier server watches events and processes a request once logged by ```RequestInfo()```. 
    
    Requesters must prepay the gas cost incurred by the TownCrier server in relaying a response to the Application Contract.
    ```msg.value``` is the amount of wei a requester pays and recorded as ```Request.fee```.
    
* ```deliver(uint64 requestId, bytes32 paramsHash, uing64 error, bytes32 respData) public;```
    
    After fetching data and generating the response for the request with ```requestId```, TC sends a transaction calling function ```deliver()```. 
    ```deliver()``` verifies that the function call is made by SGX and that the hash of request parameters is correct. Then it calls the callback function of the Application Contract to transmit the response. 
   
    The response includes ```error``` and ```respData```. If ```error = 0```, then the Application Contract request has been successfully processed. The application contract can then safely use ```respData```. The fee paid by the Application Contract for the request is then consumed by TC. If ```error = 1```, the Application Contract request is invalid or cannot be found on the website. In this case, similarly, the free is consumed by TC. If ```error > 1```, then an error has occured in the Town Crier server. In this case, the fee is refunded (minus transaction costs). 
    
* ```cancel(uint64 requestId) public returns(bool);```
    
    A requester can cancel a request whose response has not yet been issued. 
    The fee paid by the applciation contract is then refunded (minus processing costs). 

For more details, you can look at the source code of the contract [TownCrier.sol].

## A Reference Application Contract for TC

An Application Contract using Town Crier relies on a set of five basic internal functions:

* ```function() public payable;```

    This fallback function must be payable such that TC can provide a refund under certain conditions.
    
* ```function Application(TownCrier tc) public;```
    
    The Application Contract needs to store the address of the TC Contract during creation so that it can call the ```request()``` and ```cancel()``` functions in the TC contract.

* ```function request(uint8 requestType, bytes32[] requestData) public payable;```
    
    To send a request to the ```TownCrier``` contract, you need to call ```TownCrier.request()```

* ```function response(uint64 requestId, uint64 error, bytes32 respData) public;```



* ```function cancel(uint64 requestId) public;```


 
The following is the complete Application Contract logic required to interface with TC.
```
```
You can use the following scripts to deploy this logic. The input parameter is the address of the TC contract. ```App``` is the compiled Application Contract.




## Tips for Designing Application Contracts
### Flight Insurance
Suppose Alice wants to stand up a flight insurance service in the form of a smart contract AliceIns. A user can buy a policy by sending money to AliceIns. AliceIns offers a payout to the user should her insured flight be delayed or cancelled. (Unfortunately, TC cannot detect whether you've been senselessly beaten and dragged off your flight by United Airlines.)
```

```

[Town Crier: An Authenticated Data Feed for Smart Contracts]: https://eprint.iacr.org/2016/168.pdf
[TownCrier.sol]: https://github.com/bl4ck5un/Town-Crier/blob/master/Contracts/TownCrier.sol
[Application.sol]:
[FlightInsurance.sol]:

