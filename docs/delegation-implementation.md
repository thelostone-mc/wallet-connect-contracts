# Delegation Implementation

## New Storage Variables

### `DelegationInfo` struct
```solidity
struct DelegationInfo {
    address delegatee;              // Who receives the votes
    uint256 originalExpiryWeek;     // Original expiry week when delegation was made
}
```

**Why:** We need to track the original expiry week separately from the current lock state. When a user converts from a decay lock to a permanent lock, the current expiry week becomes 0, but we still need the original expiry week to correctly clean up `slopeExpiry` entries.

- **`mapping(address => DelegationInfo) delegates`** - Stores both the delegatee address and the original expiry week together.

- **`mapping(address => Point[]) delegationPoints`** - Checkpoints for each delegatee's voting power over time. Used to calculate historical voting power.

- **`mapping(address => mapping(uint256 => int128)) delegationSlopeExpiry`** - Tracks when slopes decrease due to delegator lock expirations. Needed for accurate piecewise decay calculations.

## New Functions

- **`_checkpointDelegate(...)`** - Called automatically when a user's lock changes (deposit, increase time, convert to permanent).
  - Calculates the old contribution (prevBias, prevSlope)
  - Calculates the new contribution (newBias, newSlope)
  - Removes old contribution from delegatee using **originalExpiryWeek**
  - Adds new contribution to delegatee using **newExpiryWeek**
  - Updates stored `originalExpiryWeek` to match new lock state
  - **Why:** Ensures delegation state stays synchronized with lock changes without requiring re-delegation.

- **`_delegationBiasAndSlopeAt(address delegatee, uint256 timestamp)`** - Calculates a delegatee's voting power at a specific timestamp using piecewise decay.
  - Gets the most recent checkpoint
  - If timestamp is before/at checkpoint, returns checkpoint value
  - Otherwise, iterates week-by-week applying slope changes from `delegationSlopeExpiry`
  - Returns decayed bias and slope
  - **Why:** Needed for `getVotes` and `getPastVotes` to calculate voting power at any point in time.

- **`_updateDelegateeCheckpoint(...)`** - Updates a delegatee's checkpoint with bias/slope changes.
  - Calculates current bias/slope at block.timestamp
  - Creates new checkpoint with updated values
  - Updates `delegationSlopeExpiry` if expiryWeek > 0
  - Emits `DelegateVotesChanged` event
  - **Why:** Centralized function to update delegatee state and manage `slopeExpiry` correctly.

- **`delegates(address account)`** - Returns the delegatee address for an account (required by IVotes interface).

- **`delegate(address delegatee)`** - Public function to delegate votes from the sender to a delegatee (required by IVotes interface).

- **`delegateBySig(...)`** - Delegates votes using a signature, allowing gasless delegation (required by IVotes interface).

- **`getVotes(address account)`** - Returns current voting power of an account (required by IVotes interface).

- **`getPastVotes(address account, uint256 timepoint)`** - Returns voting power of an account at a specific past timestamp (required by IVotes interface).

- **`getPastTotalSupply(uint256 timepoint)`** - Returns total voting power supply at a specific past timestamp (required by IVotes interface).

- **`_delegate(...)`** - Internal function to delegate votes from an account to a delegatee.
  - Stores `originalExpiryWeek` when delegation is created
  - Uses stored `originalExpiryWeek` (not current state) when removing from old delegatee
  - **Why:** Ensures correct cleanup of `slopeExpiry` entries even if lock type changed.

- **`_calculateAccountDelegationParams(address account)`** - Helper function to calculate an account's current delegation parameters (bias, slope, expiryWeek).

## Updated Functions

- **`_checkpoint(...)`** - Now calls `_checkpointDelegate()` after updating user's lock state. **Why:** Automatically syncs delegation when locks change.

## Example Flows

The examples in `spike.md` use a scenario where multiple users (Bob, Carol, Dave, Ed) stake WCT tokens with different lock durations and delegate their voting power to Alice. The examples demonstrate how the delegation system handles storage updates, voting power calculations, and delegation changes over time.

**Scenarios covered:**

1. **Storage Walkthrough: How slopeExpiry is Stored Step-by-Step**
   - Multiple delegators delegating to the same delegatee (Bob, Carol, Dave → Alice)
   - Adding a new delegation after time has passed (Ed → Alice at t2)
   - Shows how `delegationPoints` and `delegationSlopeExpiry` are updated with each operation
   - Demonstrates piecewise decay calculation when adding delegations after time has elapsed

2. **Example: Querying at t5 (3 weeks after checkpoint)**
   - Querying voting power at a future timestamp using `getPastVotes`
   - Shows how piecewise decay iterates week-by-week applying slope changes
   - Demonstrates how expired delegations are handled during calculation

3. **Example: Changing Delegation at t3 (Carol switches from Alice to Ying)**
   - Removing a delegation from one delegatee (Carol from Alice)
   - Adding the same delegation to a new delegatee (Carol to Ying)
   - Shows how `delegationSlopeExpiry` is updated when delegations change
   - Demonstrates cleanup of slope expiry entries when removing delegations

For detailed step-by-step examples, see:

- **[Storage Walkthrough: How slopeExpiry is Stored Step-by-Step](spike.md#storage-walkthrough-how-slopeexpiry-is-stored-step-by-step)** - Shows how storage is updated when delegations are added/removed
- **[Example: Querying at t5 (3 weeks after checkpoint)](spike.md#example-querying-at-t5-3-weeks-after-checkpoint)** - Shows how `getPastVotes` calculates voting power using piecewise decay
- **[Example: Changing Delegation at t3](spike.md#example-changing-delegation-at-t3-carol-switches-from-alice-to-ying)** - Shows how delegation changes are handled

