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
| [1] Init function exposed                                                                     | High          |       NFT Lending Platform
| [2] RoundImplementation can be frontrunned                                                             | Medium        | Public Grants System  
| [3] Unsafe usage of ERC20 transferFrom                                                                       | Medium        |    NFT Staking Platform     |  |
| [4] Business Logic can be manipulated                   | High           | Public Grants System      |  |
| [5] Precision Loss in setReadyForPayout                                                       | Medium           | Staking and Lending Platform    |  |
| [6] No check to reference an external call |     Low       | NFT Staking & Lending   |  |
| [7]                                                                                   |            |        |  |
| [8]                                                                               |  |        |  |
| [9]                                               |  |       |  |




# [1] Init function left open for anyone to call and initialize the protocol.

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

## Methods used

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




# [2] Exposed Initializer function in RoundImplementation allows for a malicious user to take control of Voting Stratgey

## Summary
Exposed Initializer function in RoundImplementation allows for a malicious user to take control of Voting Stratgey

"The IVotingStrategy (https://github.com/allo-protocol/contracts/blob/main/contracts/votingStrategy/IVotingStrategy.sol) provides a few methods and modifiers to help you implement a custom Voting Strategy.
A Round contract will call the init() method of a Voting Strategy when the Round is itself initialized. This will set the value for the roundAddress state variable, which serves two purposes: one preventing reinitialization and adding authorization to certain methods that should only be called by the Round contract.
Your implementation of the IVotingStrategy interface should implement a vote(bytes[],address) method. This is where your custom vote-counting logic should live."

## Vulnerability Detail
An attacker (Bob) can call the initialize function in RoundImplementation.

```solidity
  function initialize(
    bytes calldata encodedParameters,
    address _alloSettings
  ) external initializer {
```

Subsequently, Bob, now calling the Initializer in roundImplementation can also the init functions in IVotingStrategy.

Calling the init function in IVotingStrategy

   * @notice Invoked by RoundImplementation on creation to
   * set the round for which the voting contracts is to be used
   
   */
```Solidity
  function init() external {
    require(roundAddress == address(0), "init: roundAddress already set");
    roundAddress = msg.sender;
  }
```

And now, the attacker has the roundAddress under control. Bob can now, as he is the roundAddress, call the below function in QuadraticFundingVotingStratgeyImplementation.

@dev
   * - more voters -> higher the gas
   * - this would be triggered when a voter casts their vote via grant explorer
   * - can be invoked by the round
   * - supports ERC20 and Native token transfer
   
   * @param encodedVotes encoded list of votes
   * @param voterAddress voter address
   */

```solidity
  function vote(bytes[] calldata encodedVotes, address voterAddress) external override payable nonReentrant isRoundContract {

    /// @dev iterate over multiple donations and transfer funds
    for (uint256 i = 0; i < encodedVotes.length; i++) {

      /// @dev decode encoded vote
      (
        address _token,
        uint256 _amount,
        address _grantAddress,
        bytes32 _projectId
      ) = abi.decode(encodedVotes[i], (
        address,
        uint256,
        address,
        bytes32
      ));

      if (_token == address(0)) {
        /// @dev native token transfer to grant address
        // slither-disable-next-line reentrancy-events
        AddressUpgradeable.sendValue(payable(_grantAddress), _amount);
      } else {

        /// @dev erc20 transfer to grant address
        // slither-disable-next-line arbitrary-send-erc20,reentrancy-events,
        SafeERC20Upgradeable.safeTransferFrom(
          IERC20Upgradeable(_token),
          voterAddress,
          _grantAddress,
          _amount
        );
```

## Impact
This will allow Bob to manipulate votes in his favour, and manipulating the voting strategy. 


## Code Snippet
https://github.com/sherlock-audit/2023-03-Gitcoin/blob/main/contracts/contracts/round/RoundImplementation.sol#L196

https://github.com/sherlock-audit/2023-03-Gitcoin/blob/main/contracts/contracts/votingStrategy/IVotingStrategy.sol#L36

## Methods used

VS Code

Manual Review

## Recommendation
This will prevent the above scenario by securing both the Initializer function in RoundImplementation and the init function in IVotingStrategy.

```solidity
function initialize(bytes calldata encodedParameters, address _alloSettings) external Initializer onlyOwner {
```

```solidity
function init() internal {
  require(roundAddress == address(0x0), "roundAddress already set");
  roundAddress = address(this);
```
  


# [3] Unsafe usage of ERC20 transferFrom

## Summary
Unsafe usage of ERC20 transferFrom

## Vulnerability Detail
This function calls the CheckedTransferFrom function of the Erc20CheckedTransfer library contract, passing in 

```solidity
function checkedTransferFrom(IERC20 currency, address from, address to, uint256 amount)
```
However, this function is dealing with an IERC20 standard, which typically needs to return a bool value in Order
to be considered a valid transfer.

## Impact
This means that if the transferFrom function on the IERC20 token that you are using 
does not return a boolean value indicating whether the transfer was successful or not, 
and the transfer fails for any reason (such as insufficient balance or a problem with the token contract), 
then the checkedTransferFrom function will revert, and the useLoan function will also revert. This also means
that any NFTs that were due to be purchased, will not be and any gas spent on the transaction will be lost.

## Code Snippet
https://github.com/sherlock-audit/2023-02-kairos/blob/main/kairos-contracts/src/AuctionFacet.sol#L59

https://github.com/sherlock-audit/2023-02-kairos/blob/main/kairos-contracts/src/utils/Erc20CheckedTransfer.sol#L8

## Methods used

VS Code
Manual Review

## Recommendation
You can try to mitigate the above using the below refactored code.

```solidity
function checkedTransferFrom(IERC20 currency, address from, address to, uint256 amount) internal returns (bool) {
    if (currency.transferFrom(from, to, amount)) {
        return true;
    } else {
        return false;
    }
}
```

Its also worth using OZ SafeERC20 library when interacting with ERC20 tokens.




# [4] Business logic can be manipulated

## Summary
A malicious user can frontrun and call the init function in RoundImplementation and take control of the business logic

## Vulnerability Detail
An attacker can call the initialize function and therefore assign themselves privaleged roles, such as DEFAULT_ADMIN_ROLE
and ROUND_OPERATOR_ROLE. In turn, can then call all the functions assigned to this Modifier role, including both functions:
```solidity
function withdraw(address tokenAddress, address payable recipent) external onlyRole(ROUND_OPERATOR_ROLE) {```

function setReadyForPayout() external payable roundHasEnded onlyRole(ROUND_OPERATOR_ROLE) {
```

to name a few. Subsequently, Bob, now calling the Initializer in ```roundImplementation``` can also the init functions in both ```IPayoutStratgey``` and ```IVotingStrategy```. 

Calling the init function in IPayoutStrategy 

* @notice Invoked by RoundImplementation on creation to
   * set the round for which the payout strategy is to be used
   *
```solidity  
function init() external {
    require(roundAddress == address(0x0), "roundAddress already set");
    roundAddress = payable(msg.sender);

    // set the token address
    tokenAddress = RoundImplementation(roundAddress).token();

    isReadyForPayout = false;
  }
```
And now, the attacker has the ```roundAddress``` under control, and can now also call 
```solidity
function setReadyForPayout() external payable isRoundContract roundHasEnded {
    require(isReadyForPayout == false, "isReadyForPayout already set");
    isReadyForPayout = true;
    emit ReadyForPayout();
  }
```
and can then call ```function withdrawFunds(address payable withdrawAddress) external payable virtual isRoundOperator {```
passing an address of their choice, 

## Impact
This means that anyone who has control of the roundAddress and admin roles can effectively take control of the entire round, including its funds, participants, and payout strategy. The protocols' business logic.

Note: I have submitted this as a high due to the fact that loss of funds is a realistic scenario. I have submitted a similar issue regarding the IVoting exposure that happens in the same fashion as above, but as a medium.

## Code Snippet
https://github.com/sherlock-audit/2023-03-Gitcoin/blob/main/contracts/contracts/round/RoundImplementation.sol#L196

## Methods used

Vs Code
Manual Review

## Recommendation
I would suggest to protect key functions like below:

```solidity
function initialize(bytes calldata encodedParameters, address _alloSettings) external Initializer onlyOwner {
```

```solidity
function init() internal {
  require(roundAddress == address(0x0), "roundAddress already set");
  roundAddress = address(this);
```



# [5] Precision loss in function setReadyForPayout

## Summary
There is a danger of precision loss in the calculation to return protocolFeeAmount and roundFeeAmount

## Vulnerability Detail

The division operation

```solidity
matchAmount * alloSettings.protocolFeePercentage() / denominator
matchAmount * roundFeePercentage) / denominator
```

Could result in precision loss if the values are not properly scaled.

For example, if matchAmount is equal to 10^18 and ```alloSettings.protocolFeePercentage()```
and ```roundFeePercentage``` are both equal to 1%, the result of the division operation would be (10^18 * 1%) / 100% = 10^16.
However, since 10^16 is greater than the maximum value that can be represented by a uint256 integer (2^256-1), the result would be truncated, resulting in a loss of precision.

## Impact

If the calculation for protocolFeeAmount or roundFeeAmount results in a loss of precision, it could lead to an incorrect amount being
deducted from the contract and sent to the protocol treasury or the round fee address. Similarly, if the calculation for neededFunds results in a loss of precision, it could lead to an incorrect amount being compared against the balance of the contract, which could result in an incorrect "Not enough funds in contract" error or allow an incorrect payout.



## Recommendation
To mitigate this issue, set the values to fixed point decimals and use safeMath library arithmetic operations.

// calculate fees using SafeMath library

    uint256 protocolFeeAmount = matchAmount.mul(protocolFeePercentage).div(denominator);
    uint256 roundFeeAmount = matchAmount.mul(roundFeePercentage).div(denominator);

## Methods Used

VS Code
Manual Review

## Contacts

I am available for smart contract security consulting. Reach out to me on:

[Twitter](https://mobile.twitter.com/golanger85).

[LinkedIn](https://uk.linkedin.com/in/joe-dakwa-92716065).

Please feel free to connect with me on my Twitter page and LinkedIn. 


I shall keep you posted on any useful articles that I compose, as I shall post them to my wiki page.

https://github.com/joedakwa/Smart-Contracts/wiki
