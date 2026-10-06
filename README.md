## Findings

### 1. Vote inflation via `resetVote` — `Ktv2.sol:139`

```solidity
blockRwd[startBlock][_to]--;
ocRwdrVote[msg.sender][startBlock] = address(0);
```

The vote is removed from an **arbitrary** `_to` rather than from the address the node actually voted for (`ocRwdrVote[msg.sender][startBlock]`). At the same time the "already voted" flag is cleared, so the node can vote again.

**Scenario** (5 nodes, `consensusReq = 3`; H1 and H2 vote for B):

| Step | M's action | B | A |
|---|---|---|---|
| 1 | `vote(A)` | 2 | 1 |
| 2 | `resetVote(B)` | 1 | 1 |
| 3 | `vote(A)` | 1 | 2 |
| 4 | `resetVote(B)` | 0 | 2 |
| 5 | `vote(A)` | 0 | 3 — consensus |
| 6 | `rwd(A, entire balance)` | | |

A single node moves honest participants' votes to its own address and takes all the ETH (see #2). The same trick can be used simply to wipe out votes for other candidates and block consensus. The `notDeclined(_to)` modifier also checks the wrong address.

**Fix:**

```solidity
function resetVote() external onlyOC epochComplete migrateFees {
    address voted = ocRwdrVote[msg.sender][startBlock];
    require(voted != address(0), "Vote missing");
    blockRwd[startBlock][voted]--;
    ocRwdrVote[msg.sender][startBlock] = address(0);
    resetOCFee();
    emit Voted(startBlock, voted, "rst");
}
```

### 2. Unbounded reward amount — `Ktv2.sol:106`

`rwd(_to, _amt)`: the amount is chosen by the calling node; only consensus on the recipient address is checked. Any node can withdraw the contract's entire balance, including other nodes' unpaid fees (`tlOcFees`). Consensus on "who" is not consensus on "how much".

**Fix:** compute the amount in the contract (e.g. `address(this).balance - tlOcFees` or a fixed share), or make the amount part of what is voted on.

### 3. Node network takeover when the node count is small — `Ktv2.sol:356`, `:261`

The `(totalOC + 1) / 2` threshold is not a majority: with 2 nodes it is 1 vote, with 4 it is 2. One node out of two can use `voteToAdd` to add any number of addresses it controls, and from then on it alone controls voting, removal of honest nodes, and rewards. The same applies to `consensusReq` for `rwd`.

**Fix:** strict majority, `totalOC / 2 + 1`.

### 4. Burning breaks stake accounting — `Ktv2.sol:466`

In `give()`, burned tokens are subtracted from `totalStk` but not from `userStks`. The sum of `userStks` ends up exceeding the actual balance, while `withdraw` requires `amt <= totalStk`. The first users to exit get their full amount; the last ones cannot withdraw anything.

**Fix:** track stakes as shares and burn proportionally, or store a write-down factor and apply it on withdrawal.

### 5. Pool spot price is manipulable — `TokenPrice.sol:13`, `:22`

`slot0` and `getReserves` can be moved with a flash loan within a single transaction. An attacker pushes the token price down, calls `give()` with a minimal `msg.value`, and burns the maximum allowed amount of stake (`maxBrn`) — repeatedly and cheaply.

Additionally:
- it does not account for which token in the pair is `token0` and which is `token1`, so the price direction may be inverted;
- both tokens are assumed to have 18 decimals;
- `sqrtPriceX96 * sqrtPriceX96` (uint160²) may not fit in uint256, and in 0.7.6 overflow is silently truncated — for a `token1/token0` price above ~2⁶⁴ the result is wrong.

**Fix:** a TWAP (`observe` in V3) or an external oracle; `FullMath.mulDiv(sqrtPriceX96, sqrtPriceX96, 1 << 64)` for the square; account for token order and decimals.

### 6. Node add/remove votes are never reset — `Ktv2.sol:356`, `:375`

After an add/remove, `addVotes`, `removeVotes`, `hasVotedAdd`, and `hasVotedRemove` are left as they are:
- votes cast by removed nodes still count;
- if a node is removed, its old votes already meet the threshold when it is re-added — a single new vote is enough;
- a node that has once voted for an address can never vote for it again.

**Fix:** round-based voting (round number in the key), or clear the counters on execution.

### 7. Ownership is permanently lost if steps are done in the wrong order — `Ktv2OwnershipTimelock.sol:46`

`registerOriginalOwner()` requires the caller to be the current `Ktv2` owner. If the owner first transfers ownership to the timelock and only then tries to register, registration is no longer possible, and there is no function to return ownership without a freeze. `Ktv2` is left without control forever.

**Fix:** combine registration and freezing into a single operation, or add a way to return ownership to the registered owner while no freeze is active.

### 8. Stale `registeredOwner` — `Ktv2OwnershipTimelock.sol:58`

`registeredOwner` is not reset when the `Ktv2` owner changes. Scenario: A registers → transfers ownership to B → B transfers ownership to the timelock in order to freeze it. A calls `freezeOwnership` first and, once the freeze expires, receives ownership.

**Fix:** in `freezeOwnership`, record that the transfer came from a specific owner (e.g. two-step transfer via `Ownable2Step`, with the timelock accepting and recording `pendingOwner`), or register and freeze atomically.

### 9. Node fees can be lost — `Ktv2.sol:174`

`withdrawOCFee` zeroes the accrued fee and decreases `tlOcFees` even when the balance is insufficient, but does not perform the transfer. Combined with #2 (draining the entire balance via `rwd`), nodes lose what they earned with no way to claim it later.

**Fix:** `require(address(this).balance >= amt)` instead of silently skipping; reserve `tlOcFees` when computing the reward.

### 10. `give()` denial of service when `burnFactor = 0` — `Ktv2.sol:296`, `:455`

`setBurnFactor` allows 0, after which `(maxBrn * P_FCTR) / (2 * burnFactor)` divides by zero and every `give()` reverts.

**Fix:** `require(amt > 0)` in `setBurnFactor`.
