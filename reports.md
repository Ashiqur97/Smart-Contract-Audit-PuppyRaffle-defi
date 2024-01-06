### [S-#] 
Looping through players array to check for duplicates in `PuppyRaffle::enterRaffle` is a potential denial of service (DOS) attack, incrementing gas costs for future entrants


** Description:** The `PuppyRaffle::enterRaffle` function loops through the `players` array to check for duplicate. 
However, the longer the `PuppyRaffle::players` array is, the more checks a new player will have to make. 
This means the gas costs for players who enter right when the raffle stats will be dramatically lower than those who enter later.
Every aditional address in the `players` array, is an additional check the loop will have to make.

```javascript
// @audit DOS Attack
 @>  

for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }

```

**Impact:** The gas cost for raffle entrants will greatly increase as more players enter the raffle.
Discouraging later users from entering, and causing a rush at the start of a raffle to be one of the first entrants in the queue.
An attacker might make the `PuppyRaffle::entrants` array so big,that no one else enters,guarenteeing themselves the win.

**Proof of Concept:**

If we have 2 sets of 100 players enter, the gas costs will be as such:

-1st 100 players: ~6252048
-2nd 100 players ~18068138

This more than 3x more expensive for the second 100 players.

<details>
    <summary>PoC</summary>
    place the following test into `PuppyRaffleTest.t.sol`.

    ```Javascript
     function test_denialOfService() public {
        // address[] memory players = new address[](1);
        // players[0] = playerOne;
        // puppyRaffle.enterRaffle{value: entranceFee}(players);
        // assertEq(puppyRaffle.players(0), playerOne);

        vm.txGasPrice(1);
        uint256 playersNum = 100;
        address[] memory players = new address[](playersNum);
        for(uint256 i = 0; i < playersNum; i++) {
            players[i] = address(i);
        }
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
        uint256 gasEnd = gasleft();

        uint256 gasUsedFirst = (gasStart - gasEnd)*tx.gasprice;
        console.log("Gas cost of the first 100 players:", gasUsedFirst);

        // now for 2nd 100 players
        address[] memory players2 = new address[](playersNum);
        for(uint256 i=0; i<playersNum; i++) {
            players2[i] = address(i+playersNum);
        }
        uint256 gasStart2 = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players2.length}(players2);
        uint256 gasEnd2 = gasleft();
        uint256 gasUsed2 = (gasStart2 - gasEnd2)*tx.gasprice;
        console.log("Gas cost of the second 100 players:", gasUsed2);

        assert(gasUsedFirst < gasUsed2);
    }
    
    ```
</details>


**Recommended Mitigation:**
