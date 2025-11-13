# WalletConnect Staking Upgrade Plan

Proposed Solution

- `StakeWeight` implements `IVotes.sol`
- Since `StakeWeight` implements a linear decay on staked amount.
- Let's walk through an example
  - `MAX_LOCK_CAP = 209 weeks`
  - Assume at `t0`, we are `block = 1000`
  - At `t0`, bob decides to stake `418 WCT` for `2 weeks`
    - bob's slope =  `418 WCT / 209 weeks` ==> `2 WCT/week`
    - vote weight (bias) for bob at `t0` = `2 WCT/week * 2weeks` = `4 votes`
    - Bob delegates to Alice
      - Get bob's weight at `t0` (`4 votes`)
      - update `delegationCheckpoints[Alice].push({block: 1000, votes: 4})`
    - Assuming Alice has not self delgated and a proposal is posted where votes at snapshoted `t0` (`block : 1000`)
      - Alice's voting power = `4 votes` (due to Bob's delegation)
  - But what happens if proposal is snapshotted at `t3` (3rd week) and Bob has not updated their stake
    - Alice's voting power will still say `4 votes` but it's actually `0` as Bob's decay lock ended at `t2` (2nd week)
    - If Bob had interaction at `t1` (1st week), then we'd have a more realistic number but still not accurate
  - Let's see what happens when another staker delegates to Alice
    - At `t4` (at block `4000`), tom decides to stake `627 WCT` for `1 week`
    - tom's slope = `627 WCT / 209 weeks` ==> `3 WCT/week`
    - vote weight (bias) for tom at `t4` = `3 WCT/week * 1week` ==> `3 votes`
    - Tom delegates to Alice
      - Get toms' weight at `t4` (`3 votes`)
      - Add the delta to `delegationCheckpoints`
      - ```
        delegationCheckpoints[Alice] = [
          {fromBlock: 1000, votes: 4} // updated at t0
          {fromBlock: 4000, votes: 7} // updated at t4
        ] ```
      - This means that the latest voting weight for Alice is `7 votes`, which is **not accurate** because Bob's vote weight has decayed to `0` at `t4` (lock expired at `t2`), but we aren't accounting for that decay. Actual should be: Bob (0) + Tom (3) = 3 votes.
  - Imagine the same situation, now if other stakers had delegated to Alice each with their own decay rate.
    - For Alice's vote weight to be accurate, all of Alice's **delegators** need to have updated their stake to reflect the right value
    - This is unrealistic - we can't rely on every delegator interacting frequently


## Potential Solution

### Option A (Leave as is - Doesn't accurately account for decay rate)
  - Keep track of delegated power for each user
  - Rely on delegatee updating their stake , causing an update on the delegated checkpoint of the delegated user

### Option B (Iterate through all delegatorsOf and compute actual weight):
  - Add reverse mapping: `mapping(address delegatee => address[] delegators) delegatorsOf`
  - Maintain this mapping on every delegation change (add/remove delegators from array)
  - When calculating voting power:
    - Iterate through all delegators for the delegatee
    - Compute each delegator's current weight using `_balanceOfAt(delegator, blockNumber)`
    - Sum them up
    - Same approach for `getPastVotes` - calculate each delegator's weight at snapshot time, then sum
  - **Tradeoff**: Accurate but O(N) gas cost where N = number of delegators. Gets expensive if delegatee has 100s of delegators.

### Option C (Slope Aggregated Per Delegate - Efficient but small drift possible):
  - Given that decay rate is linear, aggregate slopes and biases per delegatee
  - When a delegator delegates, add their slope (decay rate) and bias (initial votes) to the delegatee's totals
  - Store aggregated totals in delegation checkpoints: `{fromBlock, totalSlope, totalBias}`
  - Calculate voting power over time: `bias(at time) = bias - slope × elapsed`
  - This allows efficient O(1) voting power calculation without iterating through delegators
  - **Tradeoff**: More gas-efficient than Option B, but can have small drift when multiple locks expire at different times (as shown in the example below)

- Let's walk through an example
  - `MAX_LOCK_CAP = 209 weeks`
  - Assume at `t0`, we are `block = 1000`
    - Bob stakes `418 WCT` for `2 weeks`
      - slope: `418 / 209` => `2 WCT/week`
      - bias: `2*2` => `4 votes`
    - Carol stakes `836 WCT` for `4 weeks`
      - slope: `836 / 209` => `4 WCT/week`
      - bias: `4*4` => `16 votes`
    - Dave stakes `209 WCT` for `1 week`
      - slope: `209 / 209` => `1 WCT/week`
      - bias: `1*1` => `1 votes`
  - All 3 of them delegate to Alice at t0, we
    - Alice's total slope: `2 + 4 + 1` = `7`
    - Alice's total bias: `4 + 16 + 1` = `21 votes`
    - At t0 -> Alice's total voting power is `21 votes`
      ```
      delegationCheckpoints[Alice].push({
        fromBlock: blockAt(t0),
        totalSlope: 7,
        totalBias: 21
      })
      ```
  - At `t1` (1 week) (elapsed time = 1 week)
    - `bias (at t1)` = `bias - slope × elapsed`
    - Alice's total bias at t1: `21 - 7 * 1` = `14 votes`
    - Let's manually verify this
      - Bob's bias at t1 = 2 votes
      - Carol's bias at t1 = 12 votes
      - Dave's bias = 0
      - So Alice's bias at t1 = `2 + 12 + 0` = `14 votes`
      - no change in delegationCheckpoints[Alice]
  - At `t2` (2 weeks) (elapsed time = 2 week)
    - `Alice's bias (at t2)` = `21 - 7 * 2` = `7 votes`
    - Let's manually verify this
      - Bob's bias at t2 = 0 votes
      - Carol's bias at t2 = 8 votes
      - Dave's bias = 0 votes
      -  So Alice's bias at t1 = `0 + 8 + 0` = `8 votes`
   -  On comparing
      -  Delegation checkpoint model: 7 votes
      -  True manual model: 8 votes
      -  Drift: 1 vote
   -  Assume at `t2`,  a new staker comes along
      - Ed stakes `418 WCT` for `2 weeks`
      - slope: `418 / 209` => `2 WCT/week`
      - bias: `2*2` => `4 votes`
    - Ed delegates to Alice at t2:
      - Alice's newSlope = `oldSlope + slope_from_Ed` = `7 + 2` = `9`
      - Alice's newBias  = `oldBias  + bias_from_Ed` = `21 + 4` = `25`
        - Update checkpoint
        ```
        delegationCheckpoints[Alice].push({
          fromBlock: blockAt(t2),
          totalSlope: 9,
          totalBias: 25
        })
        ```
        - Alice's actual bias = 25 - (9 * 2) = `7 votes`
      - Let's manually verify this
        - Bob's bias at t2 = 0 votes
        - Carol's bias at t2 = 8 votes
        - Dave's bias = 0 votes
        - Ed's bias = 4 votes
        -  So Alice's bias at t1 = `0 + 8 + 0 + 4` = `12 votes`
      -  On comparing
         -  Delegation checkpoint model: 7 votes
         -  True manual model: 12 votes
         -  Drift: 5 vote


The issue here is that
  - More gas efficient than Option B
  - More accurate than Option A

Note: The drift does build up over time, so that might be an issue
Curious if there is a way we could remove the expired delegations.


### Option D (Slope Aggregated Per Delegate With Future Reduction Slope Tracking ):

Basically same as Option C but introduces a mapping slopeExpiry[delegatee][timestamp] to track when individual delegations will expire in the future.
When a delegation is created, its slope reduction is scheduled at the expiration timestamp -> e.g. slopeExpiry[Alice][t1] = 1 means that 1 slope unit should be removed when t1 arrives.
However, these expirations are not automatically applied -> they're only processed the next time someone interacts (like a new delegation or undelegation).
This can lead to minor drift between "true" off-chain bias and what's on-chain until an update occurs.

**Example Walkthrough**

Let's walk through an example
  - `MAX_LOCK_CAP = 209 weeks`
  - Assume at `t0`, we are `block = 1000`
    - Bob stakes `418 WCT` for `2 weeks`
      - slope: `418 / 209` => `2 WCT/week`
      - bias: `2*2` => `4 votes`
    - Carol stakes `836 WCT` for `4 weeks`
      - slope: `836 / 209` => `4 WCT/week`
      - bias: `4*4` => `16 votes`
    - Dave stakes `209 WCT` for `1 week`
      - slope: `209 / 209` => `1 WCT/week`
      - bias: `1*1` => `1 votes`
  - All 3 of them delegate to Alice at t0, we
    - Alice's total slope: `2 + 4 + 1` = `7`
    - Alice's total bias: `4 + 16 + 1` = `21 votes`
    - At t0 -> Alice's total voting power is `21 votes`
      ```javascript
      delegationCheckpoints[Alice].push({
        fromBlock: blockAt(t0),
        totalSlope: 7,
        totalBias: 21
      })

      slopeExpiry[Alice][t1] = 1; // Dave expires
      slopeExpiry[Alice][t2] = 2; // Bob expires
      slopeExpiry[Alice][t4] = 4; // Carol expires
      ```
  - At `t1` (1 week) (1 week is over)
    - Assume no one has interacted with their stake, so **no checkpoint is written**
    - However, if we **query** Alice's voting power at t1, we calculate it on-the-fly from the t0 checkpoint:
      - `bias (at t1)` = `checkpoint_bias - (checkpoint_slope × elapsed_time)`
      - `bias (at t1)` = `21 - (7 × 1)` = `14 votes`
    - Alice's queried values at t1:
      - totalSlope(t1): `7` (still from checkpoint, no expirations processed yet)
      - totalBias(t1): `14 votes` (calculated: 21 - 7×1)
    - Let's manually verify this
      - Bob's bias at t1 = 2 votes (was 4, decayed by 2 over 1 week)
      - Carol's bias at t1 = 12 votes (was 16, decayed by 4 over 1 week)
      - Dave's bias = 0 (expired at t1)
      - So Alice's bias at t1 = `2 + 12 + 0` = `14 votes` ✅ matches!
    - **Note**: No checkpoint is written - this is just a read/query calculation
  - At `t2` (2 weeks) (elapsed time = 2 weeks)
    - If we **query** Alice's voting power at t2 (before Ed arrives):
      - `bias (at t2)` = `21 - (7 × 2)` = `7 votes` (calculated from t0 checkpoint)
    - Dave and Bob's locks have expired, but since no one interacted, **no checkpoint was written** - the expired slopes are still in `slopeExpiry` waiting to be processed
  - At `t3` (3 weeks) - **if still no one interacted**:
    - If we **query** Alice's voting power at t3:
      - `bias (at t3)` = `21 - (7 × 3)` = `21 - 21` = `0 votes` ❌ (checkpoint model calculation)
    - Let's manually verify what it **should** be:
      - Bob's bias at t3 = 0 votes (expired at t2)
      - Carol's bias at t3 = 4 votes (was 16, decays by 4 per week: 16 - 4×3 = 4, still has 1 week left)
      - Dave's bias at t3 = 0 votes (expired at t1)
      - So Alice's bias at t3 **should be** = `0 + 4 + 0` = `4 votes` ✅
    - **Drift at t3**: Checkpoint model shows `0 votes`, but actual should be `4 votes` - **4 vote drift!**
    - This shows how drift **accumulates** when checkpoints aren't created to process expirations
  - **Now assume at `t2`, a new staker comes along (THIS IS THE DIFFERENCE)**
    - Ed stakes `418 WCT` for `2 weeks`
      - slope: `418 / 209` => `2 WCT/week`
      - bias: `2*2` => `4 votes`
      - Ed delegates to Alice
    - **Step 1: Contract processes pending expirations first**
      - Checks `slopeExpiry[Alice][t1] = 1` (Dave expired, but wasn't removed yet)
      - Checks `slopeExpiry[Alice][t2] = 2` (Bob expired, but wasn't removed yet)
      - Total slope reduction needed: `1 + 2 = 3`
      - These expired slopes need to be removed from the total
    - **Step 2: Calculate current bias from t0 checkpoint**
      - Time elapsed from t0 to t2: `2 weeks`
      - Current bias at t2: `bias_at_t2 = 21 - (7 × 2) = 7 votes`
      - Note: This uses the old slope (7), which includes Dave and Bob's expired slopes
    - **Step 3: Create new checkpoint with updated values**
      - **New slope**: `oldSlope + Ed's_slope - expired_slopes`
        - `newSlope = 7 + 2 - 3 = 6`
        - Explanation: Start with 7, add Ed's 2, subtract 3 (Dave's 1 + Bob's 2)
      - **New bias**: `current_bias_at_t2 + Ed's_bias`
        - `newBias = 7 + 4 = 11 votes`
        - Explanation: Take the decayed bias (7) and add Ed's fresh bias (4)
    - **Step 4: Write new checkpoint and schedule Ed's expiration**
      - Create new checkpoint at t2:
      ```javascript
      delegationCheckpoints[Alice].push({
        fromBlock: blockAt(t2),
        totalSlope: 6,  // Updated: removed expired slopes
        totalBias: 11   // Updated: decayed bias + Ed's bias
      })
      ```
      - Schedule Ed's future expiration:
        - `slopeExpiry[Alice][t4] += 2` (Ed expires at t4, will reduce slope by 2)
        - Note: Carol also expires at t4, so `slopeExpiry[Alice][t4]` now equals `4 + 2 = 6`
    - **Result at t2**:
      - New checkpoint shows: `bias = 11 votes`, `slope = 6`
      - If queried at t2: `11 votes` (no elapsed time from this checkpoint)
    - Let's manually verify this
      - Bob's bias at t2 = 0 votes
      - Carol's bias at t2 = 8 votes
      - Dave's bias = 0 votes
      - Ed's bias = 4 votes
      -  So Alice's bias at t2 = `0 + 8 + 0 + 4` = `12 votes`
    -  **Why the drift?**
       - **Checkpoint model**: 11 votes
         - Used decayed bias from t0: `21 - (7 × 2) = 7`
         - Added Ed: `7 + 4 = 11`
         - Problem: The decay calculation used slope=7 for the full 2 weeks, but Dave's slope (1) expired at t1, so weeks 1-2 should have used slope=6, not 7
       - **True manual model**: 12 votes
         - Correctly accounts for each delegation's individual decay and expiration
       - **Drift: 1 vote**
       - **Root cause**: The bias decay calculation doesn't account for mid-period slope expirations. It only processes expirations when creating a new checkpoint, but uses the old aggregated slope for the entire decay period.

### Option E (Piecewise Bias Calculation to Eliminate Drift):

Same as Option D, but when calculating bias decay, iterate through time periods and apply slopeExpiry changes at their respective timestamps, similar to how `_totalSupplyAt` works in StakeWeight.

Instead of: `bias_at_t2 = 21 - (7 × 2) = 7` (uses one slope for entire period)
Do piecewise:
  - Week 1: `bias = 21 - (7 × 1) = 14`, then check `delegationSlopeExpiry[t1] = 1`, subtract it: slope becomes `7 - 1 = 6`
  - Week 2: `bias = 14 - (6 × 1) = 8`
  - Add new delegation: `8 + 4 = 12` ✅ (matches manual calculation exactly!)

This eliminates drift by accounting for slope changes during the decay period, not just at checkpoint creation time.

#### Storage Walkthrough: How slopeExpiry is Stored Step-by-Step

**Setup at t0:**
- Bob stakes 418 WCT for 2 weeks → slope=2, bias=4, expires at t2
- Carol stakes 836 WCT for 4 weeks → slope=4, bias=16, expires at t4
- Dave stakes 209 WCT for 1 week → slope=1, bias=1, expires at t1
- All delegate to Alice

**Step 1: Bob delegates to Alice at t0**
```javascript
// Update checkpoint
delegationPoints[Alice].push({
  timestamp: t0,
  blockNumber: blockAt(t0),
  bias: 4,    // Bob's bias
  slope: 2    // Bob's slope
})

// Store Bob's expiration
delegationSlopeExpiry[Alice][t2] = 2  // Bob's slope expires at t2
```

**Current storage state:**
```
delegationPoints[Alice] = [
  { timestamp: t0, blockNumber: blockAt(t0), bias: 4, slope: 2 }
]

delegationSlopeExpiry[Alice] = {
  t2: 2  // Bob expires
}
```

**Step 2: Carol delegates to Alice at t0**
```javascript
// Calculate current bias from previous checkpoint
// No time elapsed yet, so bias_at_t0 = 4 (from Bob)

// Update checkpoint
delegationPoints[Alice].push({
  timestamp: t0,
  blockNumber: blockAt(t0),
  bias: 4 + 16 = 20,  // Bob + Carol
  slope: 2 + 4 = 6    // Bob + Carol
})

// Store Carol's expiration
delegationSlopeExpiry[Alice][t4] = 4  // Carol's slope expires at t4
```

**Current storage state:**
```
delegationPoints[Alice] = [
  { timestamp: t0, blockNumber: blockAt(t0), bias: 20, slope: 6 }
]

delegationSlopeExpiry[Alice] = {
  t2: 2,  // Bob expires
  t4: 4   // Carol expires
}
```

**Step 3: Dave delegates to Alice at t0**
```javascript
// Calculate current bias from previous checkpoint
// No time elapsed yet, so bias_at_t0 = 20 (from Bob + Carol)

// Update checkpoint
delegationPoints[Alice].push({
  timestamp: t0,
  blockNumber: blockAt(t0),
  bias: 20 + 1 = 21,  // Bob + Carol + Dave
  slope: 6 + 1 = 7    // Bob + Carol + Dave
})

// Store Dave's expiration
delegationSlopeExpiry[Alice][t1] = 1  // Dave's slope expires at t1
```

**Final storage state at t0:**
```
delegationPoints[Alice] = [
  { timestamp: t0, blockNumber: blockAt(t0), bias: 21, slope: 7 }
]

delegationSlopeExpiry[Alice] = {
  t1: 1,  // Dave expires (1 week)
  t2: 2,  // Bob expires (2 weeks)
  t4: 4   // Carol expires (4 weeks)
}
```

**At t2 - Ed stakes and delegates `418 WCT` for `2 weeks`:**

**Step 1: Calculate current bias at t2 (piecewise)**

Formula: `bias(tn) = bias(tn-1) - (slope(tn-1) * 1 week)`

1. Week 1 (t0 → t1):
   - Decay: `bias = 21 - (7 × 1) = 14` (slope=7 for 1 week)
   - At t1, Dave’s slope expires: `slope = 7 - 1 = 6`

2. Week 2 (t1 → t2):
   - Decay: `bias = 14 - (6 × 1) = 8` (slope=6 for 1 week)
   - At t2, Bob’s slope expires: `slope = 6 - 2 = 4`

Result at t2: bias=8, slope=4

This is accurate decay: we apply slope changes at their expiration timestamps (t1 for Dave, t2 for Bob) rather than using a constant slope. After calculating the current state at t2, we add Ed’s delegation on top of it. (note: this same iteration would be used when we use getPastVotes)

**Step 2: Add Ed's delegation**
```javascript
// Ed: slope=2, bias=4, expires at t4 (2 weeks from t2)

// New checkpoint
delegationPoints[Alice].push({
  timestamp: t2,
  blockNumber: blockAt(t2),
  bias: 8 + 4 = 12,  // Current bias + Ed's bias
  slope: 4 + 2 = 6   // Current slope + Ed's slope
})

// Store Ed's expiration
// Carol already expires at t4, so we ADD to existing entry
delegationSlopeExpiry[Alice][t4] = 4 + 2 = 6  // Carol + Ed both expire at t4
```

**Storage state after Ed delegates:**
```
delegationPoints[Alice] = [
  { timestamp: t0, blockNumber: blockAt(t0), bias: 21, slope: 7 },
  { timestamp: t2, blockNumber: blockAt(t2), bias: 12, slope: 6 }
]

delegationSlopeExpiry[Alice] = {
  t1: 1,  // Dave expires (already expired at t1, but entry remains)
  t2: 2,  // Bob expires (already expired at t2, but entry remains)
  t4: 6   // Carol + Ed expire (both expire at t4)
}
```

**Important notes about storage:**

1. **`slopeExpiry` entries persist even after expiration:**
   - `t1: 1` and `t2: 2` remain in storage
   - They're checked during piecewise calculation but don't affect future queries once processed

2. **Multiple expirations at the same timestamp are combined:**
   - `t4: 6` = Carol's `4` + Ed's `2`

3. **When querying at a future time, we:**
   - Start from the most recent checkpoint (t2)
   - Iterate week-by-week from t2 to query time
   - Check `delegationSlopeExpiry[Alice][weekCursor]` at each week
   - Apply any found slope changes

#### Example: Querying at t5 (3 weeks after checkpoint)

Let's see how `getPastVotesAt(t5)` works with Option E:

**Scenario:**
- Last checkpoint at t2: `bias = 12`, `slope = 6`
- Query at t5 (3 weeks after t2)
- `delegationSlopeExpiry[Alice]` has entries at: `t1: 1`, `t2: 2`, `t4: 6`

**Step 1: Find most recent checkpoint before t5**
```javascript
checkpoint = delegationPoints[Alice].findLast(timestamp <= t5)
// Result: checkpoint at t2 with {bias: 12, slope: 6}
```

**Step 2: Initialize from t2 checkpoint**
```javascript
delegationPoints[Alice] = [
  { timestamp: t0, blockNumber: blockAt(t0), bias: 21, slope: 7 },
  { timestamp: t2, blockNumber: blockAt(t2), bias: 12, slope: 6 } // Fetch this
]

weekCursor = t2
target = t5
```

**Step 3: Iterate week-by-week from t2 to t5**

**Iteration 1: t2 → t3**
```javascript
weekCursor = t2 + 1 week = t3

// Decay bias
timeElapsed = t3 - t2 = 1 week
bias = 12 - (6 × 1) = 6  => `bias(t2) - (slope(t2) * week)`

// Check delegationSlopeExpiry at t3
delegationSlopeExpiry[Alice][t3] → 0 (no entry)

// No slope change
slope = 6

// Continue (t3 != t5)
```

**Iteration 2: t3 → t4**
```javascript
weekCursor = t3 + 1 week = t4

// Decay bias
timeElapsed = t4 - t3 = 1 week
bias = 6 - (6 × 1) = 0

// Check delegationSlopeExpiry at t4
delegationSlopeExpiry[Alice][t4] → 6 (Carol + Ed expire!)

// Apply slope change
slope = 6 - 6 = 0

// Continue (t4 != t5)
```

**Iteration 3: t4 → t5**
```javascript
weekCursor = t4 + 1 week = t5

// Decay bias (using new slope = 0)
timeElapsed = t5 - t4 = 1 week
bias = 0 - (0 × 1) = 0

// Check delegationSlopeExpiry at t5
delegationSlopeExpiry[Alice][t5] → 0 (no entry)

// No slope change
slope = 0

// Break (t5 == t5) ✅
```

**Complete iteration summary:**

```
Iteration | Week Cursor | Time Elapsed | Bias Calculation           | Slope | Check delegationSlopeExpiry      | Action
----------|-------------|--------------|----------------------------|-------|-------------------------|------------------
Start     | t2          | -            | bias = 12                  | 6     | -                       | From checkpoint
1         | t3          | 1 week       | bias = 12 - (6 × 1) = 6    | 6     | delegationSlopeExpiry[t3] = 0     | No change, continue
2         | t4          | 1 week       | bias = 6 - (6 × 1) = 0     | 0     | delegationSlopeExpiry[t4] = 6     | Apply: slope = 6 - 6 = 0, continue
3         | t5          | 1 week       | bias = 0 - (0 × 1) = 0     | 0     | delegationSlopeExpiry[t5] = 0     | No change, break
Result    | t5          | -            | bias = 0 votes              | 0     | -                       | ✅ All expired
```

**Key points:**
- **Total iterations: 3** (one per week from t2 to t5)
- We check **every week boundary** including t3, t4, and t5
- We check even if no expiration exists (returns 0)
- We apply slope changes when found (like at t4 where Carol + Ed expire)
- We continue until `weekCursor == targetTime`

**Manual verification:**
- Bob at t5: 0 votes (expired at t2)
- Carol at t5: 0 votes (expired at t4)
- Dave at t5: 0 votes (expired at t1)
- Ed at t5: 0 votes (expired at t4)
- Total: 0 + 0 + 0 + 0 = 0 votes ✅ **Matches Option E exactly!**

### Example: Changing Delegation at t3 (Carol switches from Alice to Ying)

**Scenario:**
- Current state at t2: Alice has checkpoint with `slope = 6`, `bias = 11`
  - Bob: expired at t2 (already removed)
  - Carol: slope=4, bias=8 at t2, expires at t4
  - Dave: expired at t1 (already removed)
  - Ed: slope=2, bias=4 at t2, expires at t4
- At t3, Carol changes her delegation from Alice to a new delegatee Ying
- `delegationSlopeExpiry[Alice][t4] = 6` (Carol's 4 + Ed's 2)

**Step 1: Calculate Carol's current state at t3**

Carol staked at t0 with:
- Initial: slope=4, bias=16, expires at t4
- At t3 (3 weeks elapsed from t0):
  - `bias_at_t3 = 16 - (4 × 3) = 4 votes`
  - `slope_at_t3 = 4` (unchanged)
  - Time remaining: 1 week (expires at t4)

**Step 2: Remove Carol from Alice's delegation**

**2a. Find Alice's most recent checkpoint (t2)**
```javascript
checkpoint = delegationPoints[Alice].findLast(timestamp <= t3)
// Result: checkpoint at t2 with {bias: 11, slope: 6}
```

**2b. Calculate Alice's current bias at t3 (decay from t2 checkpoint)**
```javascript
timeElapsed = t3 - t2 = 1 week
bias_at_t3 = 11 - (6 × 1) = 5 votes
```

**2c. Remove Carol's contribution**
- Carol's bias at t3: 4 votes
- Carol's slope: 4
- New bias: `5 - 4 = 1 vote` (only Ed remains)
- New slope: `6 - 4 = 2` (only Ed's slope remains)

**2d. Update Alice's delegationSlopeExpiry**
- Remove Carol's expiration from `delegationSlopeExpiry[Alice][t4]`
- Old: `delegationSlopeExpiry[Alice][t4] = 6` (Carol's 4 + Ed's 2)
- New: `delegationSlopeExpiry[Alice][t4] = 2` (only Ed expires at t4)

**2e. Create new checkpoint for Alice at t3**
```javascript
delegationPoints[Alice].push({
  timestamp: t3,
  blockNumber: blockAt(t3),
  bias: 1,    // Only Ed's remaining bias
  slope: 2    // Only Ed's slope
})
```

**Step 3: Add Carol to Ying's delegation**

**3a. Check if Ying has any existing delegations**
- Assume Ying is new (no existing delegations)
- If Ying had existing delegations, we'd need to:
  1. Find Ying's most recent checkpoint
  2. Decay bias from that checkpoint to t3
  3. Add Carol's bias and slope

**3b. Create initial checkpoint for Ying at t3**
```javascript
delegationPoints[Ying].push({
  timestamp: t3,
  blockNumber: blockAt(t3),
  bias: 4,    // Carol's bias at t3
  slope: 4   // Carol's slope
})
```

**3c. Schedule Carol's expiration for Ying**
```javascript
delegationSlopeExpiry[Ying][t4] = 4  // Carol expires at t4 (1 week from t3)
```

**Complete storage state after t3:**

**Alice's storage:**
```javascript
delegationPoints[Alice] = [
  { timestamp: t0, blockNumber: blockAt(t0), bias: 21, slope: 7 },  // Original
  { timestamp: t2, blockNumber: blockAt(t2), bias: 11, slope: 6 }, // After Ed joined
  { timestamp: t3, blockNumber: blockAt(t3), bias: 1, slope: 2 }   // After Carol left
]

delegationSlopeExpiry[Alice] = {
  t1: 1,  // Dave expired (already processed, but entry remains)
  t2: 2,  // Bob expired (already processed, but entry remains)
  t4: 2   // Ed expires (updated: was 6, removed Carol's 4)
}
```

**Ying's storage:**
```javascript
delegationPoints[Ying] = [
  { timestamp: t3, blockNumber: blockAt(t3), bias: 4, slope: 4 }  // Carol's delegation
]

delegationSlopeExpiry[Ying] = {
  t4: 4  // Carol expires at t4
}
```

**Manual verification at t3:**

**Alice's voting power:**
- Ed's bias at t3: `4 - (2 × 1) = 2 votes` (was 4 at t2, decayed 1 week)
- Alice total: `2 votes` ✅
- Checkpoint shows: `1 vote` (calculated from t2: `11 - (6 × 1) = 5`, then `5 - 4 = 1`)
- **Note**: There's a 1 vote difference because we removed Carol's full bias (4) but Ed's bias should be 2, not 1. This is because we used Option D's simple decay. With Option E, we'd get exactly 2.

**Ying's voting power:**
- Carol's bias at t3: `4 votes` ✅
- Checkpoint shows: `4 votes` ✅

**Key points:**
1. **When removing a delegation:**
   - Find most recent checkpoint
   - Decay bias from checkpoint to current time
   - Subtract delegator's current bias and slope
   - Update `delegationSlopeExpiry` to remove delegator's expiration
   - Create new checkpoint with updated values

2. **When adding a delegation:**
   - Find most recent checkpoint (or create first one)
   - Decay bias from checkpoint to current time (if existing delegations)
   - Add delegator's current bias and slope
   - Update `delegationSlopeExpiry` to add delegator's expiration
   - Create new checkpoint with updated values

3. **delegationSlopeExpiry updates:**
   - When removing: Subtract delegator's slope from the expiration timestamp (e.g., `delegationSlopeExpiry[delegatee][expiry] -= delegatorSlope`)
   - When adding: Add delegator's slope to the expiration timestamp (e.g., `delegationSlopeExpiry[delegatee][expiry] += delegatorSlope`)
   - Multiple delegators expiring at same time are combined in one entry
   - Note: Values are stored as positive numbers representing the amount of slope to subtract when the expiration occurs

4. **Checkpoint creation:**
   - Every delegation change creates a new checkpoint
   - Checkpoints are immutable snapshots
   - Old checkpoints remain for historical queries


