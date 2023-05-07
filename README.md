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
| [7] Function reverts if assetType is not valid                                                                                  |   Low         |   NFT Staking & Lending     |  |
| [8] Expected function call, doesn't get called by EOA                                                                              | Low |        | NFT Staking & Lending |
| [9] Incorrect Loop definition                                              | Low |       | NFT Staking & Lending |
| [10] Consistent calls to getReward can result in 51% attack                                              | Medium |       | NFT Staking & Lending |
| [11] Incorrect Loop definition                                              | Low |       | NFT Staking & Lending |
| [12] NFTs are locked in the contract if no timelockEndTime is associated with them                                             | High |       | NFT Staking & Lending |
|







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


# [6] No check in place referencing the function call exclusive to the Citizen Contract

Bytes2.sol Function getReward

https://github.com/code-423n4/2023-03-neotokyo/blob/dfa5887062e47e2d0c801ef33062d44c09f6f36e/contracts/staking/BYTES2.sol#L114

	
The following comment is mentioned above the function: 
"This function is called by the S1 Citizen contract to emit BYTES to callers based on their state from the staker contract."

	

	function getReward (address _to) external {
		(
			uint256 reward,
			uint256 daoCommision // @audit - @todo Review where this is used.
		) 
		= IStaker(STAKER).claimReward(_to);

		// Mint both reward BYTES and the DAO tax to targeted recipients.
		if (reward > 0) {
			_mint(_to, reward);
		}
		if (daoCommision > 0) {
			_mint(TREASURY, daoCommision);
		}
	}



If the intention is to only calling of this function by the s1Citizen contract, as per the comments above, then you should insert a require statement. Currently, any user or EOA can call this contract. Use below example if it is indeed intentional for the Citizen Contract to call this function.
		
```require(msg.sender == S1_CITIZEN, "Only the S1 Citizen can call this.");```


# [7] Function reverts if assetType is not Valid

Function stake NeoTokyoStaker.sol

https://github.com/code-423n4/2023-03-neotokyo/blob/dfa5887062e47e2d0c801ef33062d44c09f6f36e/contracts/staking/NeoTokyoStaker.sol#L1196

If the ID of the _asset is ever equal to 4, this function will not revert. If its not the intention to do so, then perhaps refactor the below code.

	function stake ( 
		AssetType _assetType,
		uint256 _timelockId,
		uint256,
		uint256,
		uint256
	) external nonReentrant {

		// Validate that the asset being staked is of a valid type.
		if (uint8(_assetType) > 4) {
			revert InvalidAssetType(uint256(_assetType));
		}

		refactored to: 
		
		if (uint8(_assetType) >= 4) {
			revert InvalidAssetType(uint256(_assetType));
		}


With this change, the function will now throw an exception with the InvalidAssetType message if _assetType is equal to or greater than 4, which ensures that only valid asset types are accepted in the contract.


# [8] Function getReward in NTCitizenDeploy calls updateReward, which can only be called by citizenContract

https://github.com/code-423n4/2023-03-neotokyo/blob/dfa5887062e47e2d0c801ef33062d44c09f6f36e/contracts/s1/NTCitizenDeploy.sol#L1621

https://github.com/code-423n4/2023-03-neotokyo/blob/dfa5887062e47e2d0c801ef33062d44c09f6f36e/contracts/s1/BYTESContract.sol#L939

```
 function getReward() external {
        IByteContract byteToken = IByteContract(bytesContract);
        byteToken.updateReward(msg.sender, address(0), 0);
		byteToken.getReward(msg.sender);
	}
```

```
 function updateReward(address _from, address _to, uint256 _tokenId) external {
		require(msg.sender == address(citizenContract));
```

Consider adding in the same functionality here in the getReward Function, that is also comprised in the updateReward function:

			"require(msg.sender == address(citizenContract));"
            
            
# [9] Incorrect loop definition

function getStakerPositions NeoTokyoStaker

https://github.com/code-423n4/2023-03-neotokyo/blob/dfa5887062e47e2d0c801ef33062d44c09f6f36e/contracts/staking/NeoTokyoStaker.sol#L710

```for (uint256 i; i < _stakerS1Position[_staker].length; )```

replace with:

```for (uint256 i = 0; i < _stakerS1Position[_staker].length; i++).```


# [10] User can call getReward multiple times causing 51% attack 

	```solidity
	function getReward (address _to) external {
		(
			uint256 reward,
			uint256 daoCommision 
		) 
		= IStaker(STAKER).claimReward(_to);

		// Mint both reward BYTES and the DAO tax to targeted recipients.
		if (reward > 0) {
			_mint(_to, reward);
		}
		if (daoCommision > 0) {
			_mint(TREASURY, daoCommision);
		}
	}
	```


The platform staking program operates as follows:

"The staker is a competitive system where stakers compete for a fixed emission rate in each of the S1 Citizen, S2 Citizen, and LP token staking pools.
Stakers "may" choose to lock their assets for some period of time, preventing withdrawal, 
in exchange for a multiplying bonus to their share of points in competing for BYTES 2.0 token emissions."



Considering this function is marked as external, it can be called by an EOA. This means that a user can call this function multiple times in a day to claim multiple rewards. This is not the intended behavior. A malicious user called BOB can continously call getReward from various wallets to claim rewards from different wallets. 
The only check in place, is that the rewards must be greater than 0 in order to mint to both the address "_to" and the TREASURY address.

This could mean there can be significant amounts of tokens minted to all addresses, more than what is intended. So you will end up with an attacker who has multiple tokens across various wallets and an over inflated treasury worth of tokens. 


This is evidenced below. As you can see, Bob can only call _mint once, but he can call getReward multiple times,
with various wallets.


 ```solidity
 function _mint(address to, uint256 tokenId) internal virtual {
        require(to != address(0), "ERC721: mint to the zero address");
        require(!_exists(tokenId), "ERC721: token already minted");

        _beforeTokenTransfer(address(0), to, tokenId);

        _balances[to] += 1;
        _owners[tokenId] = to;

        emit Transfer(address(0), to, tokenId);
    }
 ```   


 The fact that the function can be called multiple times by a single user can still result 
 in a disproportionate distribution of rewards, as the proportion of the pool that the user receives is based on the length of time their assets have been staked. 
 So a malicious user could potentially accumulate a large proportion of the rewards by repeatedly calling getreward from different addresses, even if they can't mint more tokens each time. The other scenario to consider is a malicious user reducing their rewards to 0 by claiming BYTES tokens and then calling getReward multiple times additionally.

Really important to address the risk of inflation and the 51% attack, to keep the integrity and value of the Bytes token.


### Refactored code:

```solidity
mapping(address => uint256) private lastRewardTime;

function getReward(address _to) external {
    // Check that the user has an active stake.
    require(IStaker(STAKER).balanceOf(msg.sender) > 0, "No active stake found.");

    // Check that the user has not claimed rewards in the past 24 hours.
    require(block.timestamp >= lastRewardTime[msg.sender] + 24 hours, "Rewards can only be claimed once per day.");

    // Check that the user has rewards to claim.
    require(IStaker(STAKER).earned(msg.sender) > 0, "No rewards to claim.");

    // Record the current time as the last reward time for the user.
    lastRewardTime[msg.sender] = block.timestamp;

    // Claim the user's rewards.
    (uint256 reward, uint256 daoCommision) = IStaker(STAKER).claimReward(msg.sender);

    // Mint both reward BYTES and the DAO tax to targeted recipients.
    _mint(_to, reward);
    if (daoCommision > 0) {
        _mint(TREASURY, daoCommision);
    }
}
```


With this modification, 
the getReward function should now revert if a user with 0 rewards attempts to claim rewards.

# [12] Users cant withdraw S1 or S2 Citizens if no timelockEndTime is associated with it. As the same logic is applied in the withdrawLP function, this is not intended behavior.




Currently, there are no allowances for a S1 or S2 Citizen to withdraw their asset if they have staked it. The "if (block.timestamp < stakedCitizen.timelockEndTime)" statement will revert if the timelockEndTime is 0.
Stakers dont need to add a timelockDuration to their NFT Citizen, according to the documentation, so they should be able to withdraw freely.

```solidity
function _withdrawS1Citizen () private {
		uint256 citizenId;
		assembly {
			citizenId := calldataload(0x24)
		}

		// Validate that the caller has cleared their asset timelock.
		StakedS1Citizen storage stakedCitizen = stakedS1[msg.sender][citizenId];
		if (block.timestamp < stakedCitizen.timelockEndTime) {
			revert TimelockNotCleared(stakedCitizen.timelockEndTime);
		}

}

		// Validate that the caller actually staked this asset.
		if (stakedCitizen.timelockEndTime == 0) {
			revert CannotWithdrawUnownedS1(citizenId);
		}
		```




This updated function below allows for withdrawal of an S1 Citizen without a timelock, 
which is allowed as per the documentation.


## Refactored code:


```solidity
function _withdrawS1Citizen () private {
    uint256 citizenId;
    assembly {
        citizenId := calldataload(0x24)
    }

    // Validate that the caller has cleared their asset timelock.
    StakedS1Citizen storage stakedCitizen = stakedS1[msg.sender][citizenId];
    if (block.timestamp < stakedCitizen.timelockEndTime && stakedCitizen.timelockEndTime != 0) {
        revert TimelockNotCleared(stakedCitizen.timelockEndTime);
    }

    // Validate that the caller actually staked this asset.
    if (stakedCitizen.timelockEndTime == 0 && stakedCitizen.stakedBytes == 0 && stakedCitizen.stakedVaultId == 0) {
        revert CannotWithdrawUnownedS1(citizenId);
    }
    
    // Return any staked BYTES.
    if (stakedCitizen.stakedBytes > 0) {
        _assetTransfer(BYTES, msg.sender, stakedCitizen.stakedBytes);
    }
    
    // Return any non-component Vault if one is present.
    if (stakedCitizen.stakedVaultId != 0) {
        _assetTransferFrom(
            VAULT,
            address(this),
            msg.sender,
            stakedCitizen.stakedVaultId
        );
    }

    // Return the S1 Citizen.
    _assetTransferFrom(S1_CITIZEN, address(this), msg.sender, citizenId);
}
```



## Contacts

I am available for smart contract security consulting. Reach out to me on:

[Twitter](https://mobile.twitter.com/golanger85).

[LinkedIn](https://uk.linkedin.com/in/joe-dakwa-92716065).

Please feel free to connect with me on my Twitter page and LinkedIn. 


I shall keep you posted on any useful articles that I compose, as I shall post them to my wiki page.

https://github.com/joedakwa/Smart-Contracts/wiki
