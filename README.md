### Welcome to my Smart Contract focussed repo!

I am an experienced Smart Contract Developer and Auditor who has spent the last 10 years working in the Blockchain industry. 

I wanted to offer my knowledge and expertise in this field, as vast as it is, and try to decipher the most basic usages of smart contracts, all the way to the most complex and convoluted. 

I am creating several wiki pages and will open up various discussions for the community to engage with.

I am by no means a "maximalist". By that, I am strictly a Blockchain advocate. I do not believe one coin or chain rules them all.

I am very passionate about Ravencoin, Bitcoin, Litecoin, ZCash, Monero, Ethereum, Avalanche, Fantom and so on...

I appreciate the underlying technologies that programatically form the base of these great ecosystems and I encourage everyone to dive deep into each to understand their capabilities and problems they are trying to solve.

# Security audits by Joe Dakwa

This repository represents my collection of vulnerabilities and bug findings for my portfolio.

| Vulnerability                                                                                                                                             | Severity      | Protocol Type     |          |
| --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- | ------------ | -------- |
| Init function exposed                                                                     | High          |       NFT Lending Platform
| RoundImplementation can be frontrunned                                                             | Medium        | Public Grants System  
|                                                                        | Medium        | Davos        |  |
| [lend() function always return minted tokens equal to zero](Immunefi/README.md#lend-function-always-return-minted-tokens-equal-to-zero)                   | Low           | UniLend      |  |
| [Wrong use of assembly builtin function](Immunefi/README.md#wrong-use-of-assembly-builtin-function)                                                       | Low           | Hyperlane    |  |
| [createCanonicalERC20Wrapper reverts on right erc20 implementation](Immunefi/README.md#createcanonicalerc20wrapper-reverts-on-right-erc20-implementation) | Low           | Superfluid   |  |
| [Unchecked low level call](Immunefi/README.md#unchecked-low-level-call)                                                                                   | Low           | Aurora       |  |
| [Wrong emission of event](Immunefi/README.md#wrong-emission-of-event)                                                                                     | Informational | Revest       |  |
| [Wrong implementation of supportsInterface()](Immunefi/README.md#wrong-implementation-of-supportsinterface)                                               | Informational | Revest       |  |


Go-langer

medium

# Init function left open for anyone to call and initialize the protocol.

## Summary
Init function left open for anyone to call and initialize the protocol.
https://github.com/sherlock-audit/2023-02-kairos/blob/main/kairos-contracts/src/Initializer.sol#L24

## Vulnerability Detail
It gives the attacker the ability to set the initial values for the contract's storage variables 
and manipulate the overall behavior of the protocol, including changing the storage variables, price factors 
in auction and the APR of the tranches, to name a few.
## Impact
The attacker can change the values of ```protocolStorage()``` and ```supplyPositionStorage()```

## Code Snippet

https://github.com/sherlock-audit/2023-02-kairos/blob/main/kairos-contracts/src/Initializer.sol#L24
## Tool used
Vs Code

Manual Review

## Recommendation
Use a modifier to only allow this function to be called by the owner for example and 
introduce an access control modifier on the init function.

```solidity
contract Initializer {
    // ...

    /// @notice initializes the kairos protocol
    function init() external onlyOwner {
        // ...
    }

    // ...
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Only contract owner can call this function");
        _;
    }
}
```

Also this check must also be added to the init function, if you only want the init function to be called once.
```solidity
"require(!initialized, "Contract instance has already been initialized");"
```

Go-langer

high

RoundImplementation can be front-runned and initialized by a malicious user
Summary
A malicious user can frontrun and call the init function in RoundImplementation and take control of the business logic

Vulnerability Detail
An attacker can call the initialize function and therefore assign themselves privaleged roles, such as DEFAULT_ADMIN_ROLE
and ROUND_OPERATOR_ROLE. In turn, can then call all the functions assigned to this Modifier role, including both functions:

function withdraw(address tokenAddress, address payable recipent) external onlyRole(ROUND_OPERATOR_ROLE) {```

function setReadyForPayout() external payable roundHasEnded onlyRole(ROUND_OPERATOR_ROLE) {
to name a few. Subsequently, Bob, now calling the Initializer in roundImplementation can also the init functions in both IPayoutStratgey and IVotingStrategy.

Calling the init function in IPayoutStrategy

@notice Invoked by RoundImplementation on creation to
set the round for which the payout strategy is to be used
function init() external {
    require(roundAddress == address(0x0), "roundAddress already set");
    roundAddress = payable(msg.sender);

    // set the token address
    tokenAddress = RoundImplementation(roundAddress).token();

    isReadyForPayout = false;
  }
And now, the attacker has the roundAddress under control, and can now also call

function setReadyForPayout() external payable isRoundContract roundHasEnded {
    require(isReadyForPayout == false, "isReadyForPayout already set");
    isReadyForPayout = true;
    emit ReadyForPayout();
  }
and can then call function withdrawFunds(address payable withdrawAddress) external payable virtual isRoundOperator {
passing an address of their choice,

Impact
This means that anyone who has control of the roundAddress and admin roles can effectively take control of the entire round, including its funds, participants, and payout strategy. The protocols' business logic.

Note: I have submitted this as a high due to the fact that loss of funds is a realistic scenario. I have submitted a similar issue regarding the IVoting exposure that happens in the same fashion as above, but as a medium.

Code Snippet
https://github.com/sherlock-audit/2023-03-Gitcoin/blob/main/contracts/contracts/round/RoundImplementation.sol#L196

Tool used
Vs Code
Manual Review

Recommendation
I would suggest to protect key functions like below:

function initialize(bytes calldata encodedParameters, address _alloSettings) external Initializer onlyOwner {
function init() internal {
  require(roundAddress == address(0x0), "roundAddress already set");
  roundAddress = address(this);
  
  
Go-langer

high

RoundImplementation can be front-runned and initialized by a malicious user
Summary
A malicious user can frontrun and call the init function in RoundImplementation and take control of the business logic

Vulnerability Detail
An attacker can call the initialize function and therefore assign themselves privaleged roles, such as DEFAULT_ADMIN_ROLE
and ROUND_OPERATOR_ROLE. In turn, can then call all the functions assigned to this Modifier role, including both functions:

function withdraw(address tokenAddress, address payable recipent) external onlyRole(ROUND_OPERATOR_ROLE) {```

function setReadyForPayout() external payable roundHasEnded onlyRole(ROUND_OPERATOR_ROLE) {
to name a few. Subsequently, Bob, now calling the Initializer in roundImplementation can also the init functions in both IPayoutStratgey and IVotingStrategy.

Calling the init function in IPayoutStrategy

@notice Invoked by RoundImplementation on creation to
set the round for which the payout strategy is to be used
function init() external {
    require(roundAddress == address(0x0), "roundAddress already set");
    roundAddress = payable(msg.sender);

    // set the token address
    tokenAddress = RoundImplementation(roundAddress).token();

    isReadyForPayout = false;
  }
And now, the attacker has the roundAddress under control, and can now also call

function setReadyForPayout() external payable isRoundContract roundHasEnded {
    require(isReadyForPayout == false, "isReadyForPayout already set");
    isReadyForPayout = true;
    emit ReadyForPayout();
  }
and can then call function withdrawFunds(address payable withdrawAddress) external payable virtual isRoundOperator {
passing an address of their choice,

Impact
This means that anyone who has control of the roundAddress and admin roles can effectively take control of the entire round, including its funds, participants, and payout strategy. The protocols' business logic.

Note: I have submitted this as a high due to the fact that loss of funds is a realistic scenario. I have submitted a similar issue regarding the IVoting exposure that happens in the same fashion as above, but as a medium.

Code Snippet
https://github.com/sherlock-audit/2023-03-Gitcoin/blob/main/contracts/contracts/round/RoundImplementation.sol#L196

Tool used
Vs Code
Manual Review

Recommendation
I would suggest to protect key functions like below:

function initialize(bytes calldata encodedParameters, address _alloSettings) external Initializer onlyOwner {
function init() internal {
  require(roundAddress == address(0x0), "roundAddress already set");
  roundAddress = address(this);
  
  


## Contacts

I am available for smart contract security consulting. Reach out to me on:

[Twitter](https://mobile.twitter.com/golanger85).

[LinkedIn](https://uk.linkedin.com/in/joe-dakwa-92716065).

Please feel free to connect with me on my Twitter page and LinkedIn. 


I shall keep you posted on any useful articles that I compose, as I shall post them to my wiki page.

https://github.com/joedakwa/Smart-Contracts/wiki
