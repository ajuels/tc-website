Title: Get Started
Date: 2017-4-6
Category: Tutorial


# Get Started with Town Crier

For smart contracts on blockchain systems, it's currently difficult to get real-world data because authentication for the data fed back to the blockchain is a crucial issue.
A currency exchange contract requires exchange ratio.
An trip insurance contract requires the whether information.
A trade contract requires delivery result.
There are a lot of examples of applications that cannot run without real-world data feed.
The question is who should be trusted to feed such data into smart contracts.

<! Brief idea of Oraclize and comparison with TC required >
The Town Crier system addresses this problem by using trusted hardware such as Intel SGX to provide an authenticated data feed.
When using the data fed by the Town Crier system in smart contracts, people don't have to trust any other party except SGX and the website where the data is from.

In order to get certain data, a smart contract may send a request to the Town Crier server.
Then the Town Crier server will fetch required data from a trusted website and send it back to the contract.
The parsing of a request and the generation of a response with SGX's signature are processed on an SGX enclave inside the Town Crier server.
The fetching of required data is done via TLS connection between the enclave and the website.
SGX guarantees confidentiality and integrity, which means the status of a program running on an SGX enclave cannot be either revealed or modified by an adversary.
TLS provides a secure channel for the communication of two parties, which also guarantees confidentiality and integrity.

For more details of Town Crier system and its security guarantees, please look at our paper [Town Crier: An Authenticated Data Feed for Smart Contracts].

## Understand the ```TownCrier``` Contract

The ```TownCrier``` contract provides uniform interfaces for different application contracts to send requests to the Town Crier server and for the Town Crier server to respond to applications.
It consists of the following three functions.

* ```request(uint8 requestType, address callbackAddr, bytes4 callbackFID, uint256 timestamp, bytes32[] requestData) public payable returns(uint64);```

	For an application contract to call function ```request()```, it needs to send in the following parameters.
    
    - ```requestType```: the type of the request, for the Town Crier server to process it accordingly. You'll see all the request types Town Crier currently supports in the following.
    
    - ```callbackAddr```: the address of the application contract to forward the response from the Town Crier to.
    
    - ```callbackFID```: the specification for the callback function in application contract to forward the response from the Town Crier.
    
    - ```timestamp```: parameter required for a potential feature. Currently Town Crier server could only send response back once discover the request and successfully fetch the data. We expect to make it support fetching data at a certain point of time and then responding, in the future. For now developers don't need to concern about this and just setting it as 0 would be fine.
    
    - ```requestData```: data required to specifying the request. The format will be specified together with request types.

    By calling this function, the request is logged by event ```RequestInfo()```, and then the function will return ```requestId``` which is assigned to this request such that the requester could use it to look up the logs for response or status of the request.
    The Town Crier server will watch events and process the request once find one ```RequestInfo()```. 
    
    Requesters should pay for the gas cost for the TownCrier server to send the response to the blockchain afterwards.
    ```msg.value``` is the amount of wei a requester pays and recorded as ```Request.fee```.
    
* ```deliver(uint64 requestId, bytes32 paramsHash, uing64 error, bytes32 respData) public;```
    
    After fetching data and generating the response for the request with ```requestId```, the Town Crier server sends a transaction calling function ```deliver()```. 
    ```deliver()``` verifies that the function call is made by SGX and that the hash of request parameters is correct, and then it will call the callback function of the application contract to transmit the response. 
   
    The response includes ```error``` and ```respData```.
    The application contract can use ```respData``` if there's no exception, which could be indicated by ```error = 0```.
    The fee paid by requester when requesting will go to the SGX account.
    If ```error = 1```, the request is invalid or cannot be found on the website.
    Otherwise ```error > 1``` indicates that the error exists inside Town Crier server and the requester sould be refunded. 
    
* ```cancel(uint64 requestId) public returns(bool);```
    
    A requester could cancel a request which hasn't been responded to. 
    The fee paid by requester would be partially refunded. 

For more details, you can look into the source code of the contract [TownCrier.sol].

## Application Contract upon Town Crier by Example

For an application contract upon Town Crier, here are the five basic functions:

* ```function() public payable;```

    The fallback function must be payable such that TownCrier contract could refund to this contract when certain condition meets.
    
* ```function Application(TownCrier tc) public;```
    
    The Application contract needs to store the address of TownCrier contract during creation such that it could call ```request()``` and ```cancel()``` of TownCrier contract.

* ```function request(uint8 requestType, bytes32[] requestData) public payable;```
    
    To send a request to the ```TownCrier``` contract, you need to call ```TownCrier.request()```

* ```function response(uint64 requestId, uint64 error, bytes32 respData) public;```



* ```function cancel(uint64 requestId) public;```


 
The following is the complete contract for general request.
```
```
You can use the following scripts to deploy this contract. The parameter fed in is the address of TownCrier contract. ```App``` is the compiled Application contract.




## Tips for Designing Application Contracts
### Flight Insurance
Suppose Alice wanted to provide a flight insurance service by deploying a smart contract such that clients would get paid when their flights insured are delayed or cancelled (or maybe even when they got beaten by UA).
```

```

[Town Crier: An Authenticated Data Feed for Smart Contracts]: https://eprint.iacr.org/2016/168.pdf
[TownCrier.sol]: https://github.com/bl4ck5un/Town-Crier/blob/master/Contracts/TownCrier.sol
[Application.sol]:
[FlightInsurance.sol]:

