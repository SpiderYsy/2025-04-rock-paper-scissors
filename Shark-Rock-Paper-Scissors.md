# Rock Paper Scissors - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. joinGameWithEth can attend the game.bet = 0 when value = 0 ether to win token without risk without check msg.value ==0](#H-01)
    - ### [H-02. Function revealMove() parameter `_move` can be seen on chain ](#H-02)
    - ### [H-03. emit MoveRevealed(_gameId, msg.sender, move, game.currentTurn) will show the move](#H-03)
- ## Medium Risk Findings
    - ### [M-01. GameState.Revealed not be claimed in the revealMove function](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #38

### Dates: Apr 17th, 2025 - Apr 24th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-04-rock-paper-scissors)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 3
- Medium: 1
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. joinGameWithEth can attend the game.bet = 0 when value = 0 ether to win token without risk without check msg.value ==0            



## Summary

joinGameWithEth can attend the game. bet = 0 when value = 0 ether to win token without risk without check msg.value ==0

## Vulnerability Details

`joinGameWithEth ` has no check for  msg.value == 0 , causing attacker can attend Token bet game with 0 value when call `joinGameWithEth`. To win Token without risk.

There is no check for msg.value == 0

&#x20;  &#x20;

```Solidity
function joinGameWithEth(uint256 _gameId) external payable {
        Game storage game = games[_gameId];

        require(game.state == GameState.Created, "Game not open to join");
        require(game.playerA != msg.sender, "Cannot join your own game");
        require(block.timestamp <= game.joinDeadline, "Join deadline passed");
@>      require(msg.value == game.bet, "Bet amount must match creator's bet");
        // what will happen if the msg.value== 0    

        game.playerB = msg.sender;
        emit PlayerJoined(_gameId, msg.sender);
    }
```

## Impact

attacker can attend the game created by `createGameWithToken ` with game.joinGameWithEth{value: 0 ether}(gameId);

Causing the attacker can win Token with no risk.

### Proof of Code

PlayerB get the Token when the game is canceled.

```Solidity
 function testJoinGameWithToken() public {
        console2.log("Player A token: ", token.balanceOf(playerA));
        console2.log("Player B token: ", token.balanceOf(playerB));
        vm.startPrank(playerA);
        token.approve(address(game), 1);
        gameId = game.createGameWithToken(TOTAL_TURNS, TIMEOUT);
        vm.stopPrank();


        vm.startPrank(playerB);

        game.joinGameWithEth{value: 0 ether}(gameId);
        vm.stopPrank();

       //
    
        (address storedPlayerA, address storedPlayerB,,,,,,,,,,,,,, RockPaperScissors.GameState state) =
            game.games(gameId);

        assertEq(storedPlayerA, playerA);
        assertEq(storedPlayerB, playerB);

        console2.log("Player A token: ", token.balanceOf(playerA));
        console2.log("Player B token: ", token.balanceOf(playerB));
        vm.startPrank(playerA); 

        game.cancelGame(gameId);
        vm.stopPrank();
        console2.log("Player A token: ", token.balanceOf(playerA));
        console2.log("Player B token: ", token.balanceOf(playerB));

    }
Log:
  Player A token:  10
  Player B token:  10
  Player A token:  9
  Player B token:  10
  Player A token:  10
  Player B token:  11
```

## Tools Used

manual review

## Recommendations

```diff
function joinGameWithEth(uint256 _gameId) external payable {
        Game storage game = games[_gameId];

        require(game.state == GameState.Created, "Game not open to join");
        require(game.playerA != msg.sender, "Cannot join your own game");
        require(block.timestamp <= game.joinDeadline, "Join deadline passed");
+       require(msg.value != 0);
        require(msg.value == game.bet, "Bet amount must match creator's bet");
        // what will happen if the msg.value== 0    

        game.playerB = msg.sender;
        emit PlayerJoined(_gameId, msg.sender);
    }
```

## <a id='H-02'></a>H-02. Function revealMove() parameter `_move` can be seen on chain             



## Summary

Function revealMove() parameter `_move` can be seen on chain ,causing the first player  to call revealMove will be lose.

## Vulnerability Details

revealMove() use \_move as the parameter ,which can be seen on transaction. So the first player to call this function will lose.

```Solidity
function revealMove(uint256 _gameId, uint8 _move, bytes32 _salt) external {}
```



## Impact

the first player  to call revealMove will lose Token or Ether.

## Tools Used

manual review.

## Recommendations

Use other way to record \_move .

## <a id='H-03'></a>H-03. emit MoveRevealed(_gameId, msg.sender, move, game.currentTurn) will show the move            



## Summary

emit MoveRevealed(\_gameId, msg.sender, move, game.currentTurn) will show the move ,causing the player's move been seen by the other player making losing.

## Vulnerability Details

the event will show the move. the player who call revealMove() first will be seen the move,causing lose ether or Token.

```solidity
emit MoveRevealed(_gameId, msg.sender, move, game.currentTurn);
```

## Impact

the event will show the move. the player who call revealMove() first will be seen the move,causing lose ether or Token.

## Tools Used

manual review

## Recommendations

delete the move in the event.

```diff
-   emit MoveRevealed(_gameId, msg.sender, move, game.currentTurn);
+   emit MoveRevealed(_gameId, msg.sender, game.currentTurn);
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. GameState.Revealed not be claimed in the revealMove function            



## Summary

GameState.Revealed not be claimed in the revealMove function

## Vulnerability Details

```solidity
function revealMove(uint256 _gameId, uint8 _move, bytes32 _salt) external {
        Game storage game = games[_gameId];

        require(msg.sender == game.playerA || msg.sender == game.playerB, "Not a player in this game");
        require(game.state == GameState.Committed, "Game not in reveal phase");
        require(block.timestamp <= game.revealDeadline, "Reveal phase timed out");
        require(_move >= 1 && _move <= 3, "Invalid move");

        Move move = Move(_move);
        bytes32 commit = keccak256(abi.encodePacked(move, _salt));

        if (msg.sender == game.playerA) {
            require(commit == game.commitA, "Hash doesn't match commitment");
            require(game.moveA == Move.None, "Move already revealed");
            game.moveA = move;
        } else {
            require(commit == game.commitB, "Hash doesn't match commitment");
            require(game.moveB == Move.None, "Move already revealed");
            game.moveB = move;
        }

        emit MoveRevealed(_gameId, msg.sender, move, game.currentTurn);

        // If both players have revealed, determine the winner for this turn
        if (game.moveA != Move.None && game.moveB != Move.None) {
            _determineWinner(_gameId);
        }
    }
```

## Impact

GameState.Revealed not be claimed in the revealMove function

## Tools Used

manual review

## Recommendations



```diff
if (game.moveA != Move.None && game.moveB != Move.None) {
+           game.state = GameState.Revealed 
            _determineWinner(_gameId);
        }
```





