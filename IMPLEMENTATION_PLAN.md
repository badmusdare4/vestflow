# Implementation Plan for Issues #609 and #610

## Overview

This document outlines the implementation plan for two contract enhancements in the VestFlow project:

- **Issue #609**: Add StreamReceiver and SplitsReceiver structs with validation
- **Issue #610**: Emit stream_received event after receive_streams call

## Issue #609: StreamReceiver and SplitsReceiver Structs with Validation

### Current State

- `StreamReceiver` struct exists at line 277-281 with basic fields:
  ```rust
  pub struct StreamReceiver {
      pub receiver: Address,
      pub amt_per_sec: i128,
  }
  ```
- `AddressSplitsReceiver` and `NftSplitsReceiver` structs exist at lines 304-320
- No validation logic is currently enforced at the struct level

### Proposed Changes

#### 1. Add validation for StreamReceiver

- Create a validation method or constructor that ensures `amt_per_sec > 0`
- Add typed error for zero/negative rate: `VestFlowError::AmountZero` (already exists as variant #5)

#### 2. Add validation for SplitsReceiver

- Create validation for `AddressSplitsReceiver` to ensure `weight > 0`
- Create validation for `NftSplitsReceiver` to ensure `weight > 0`
- Add typed error for zero weight (may need new variant)

#### 3. Export structs for SDK bindings

- Ensure both `StreamReceiver` and splits receiver structs are properly exported
- They already have `#[contracttype]` decorator which makes them available in SDK

### Testing Requirements

- ✅ Test valid configuration accepted
- ✅ Test zero rate rejected for StreamReceiver
- ✅ Test negative rate rejected for StreamReceiver
- ✅ Test zero weight rejected for AddressSplitsReceiver
- ✅ Test zero weight rejected for NftSplitsReceiver
- ✅ Test structs usable from SDK bindings (integration test)

## Issue #610: Emit stream_received Event

### Current State

- `receive_streams` function exists at lines 3984-4053
- Currently emits event at line 4050:
  ```rust
  env.events()
      .publish((symbol_short!("strm_recv"), funder, token), capped);
  ```
- Event is emitted regardless of whether cycles were processed

### Proposed Changes

#### 1. Event Shape Modification

Current event structure needs enhancement to match spec:

- **Topics**: `[account, token]`
- **Value**: `{ cycles_processed: u32, amount_received: i128 }`

Modifications needed:

- Calculate `cycles_processed` from elapsed time: `elapsed / CYCLE_SECS`
- Add cycles_processed to event value
- Keep amount_received (currently `capped`) in event value

#### 2. Conditional Event Emission

- Only emit event when `cycles_processed > 0`
- This prevents emission when no actual streaming occurred

#### 3. Event Structure

Update line 4050 to:

```rust
let cycles_processed = (elapsed / CYCLE_SECS as i128) as u32;
if cycles_processed > 0 {
    env.events().publish(
        (symbol_short!("strm_recv"), funder, token),
        (cycles_processed, capped),
    );
}
```

### Testing Requirements

- ✅ Test event emitted after successful receive_streams call
- ✅ Test event NOT emitted when 0 cycles are processed (elapsed < CYCLE_SECS)
- ✅ Test event topics include account and token
- ✅ Test event value includes cycles_processed and amount_received
- ✅ Test cycles_processed calculation is correct

## Implementation Order

1. **Issue #610 first** (simpler change, quick win):
   - Modify `receive_streams` function to calculate cycles
   - Update event emission logic
   - Add tests for event behavior

2. **Issue #609 second** (requires more consideration):
   - Add validation helpers for StreamReceiver
   - Add validation helpers for SplitsReceiver structs
   - Update call sites to use validation
   - Add comprehensive tests

## Files to Modify

### contracts/vestflow/src/lib.rs

- Line 277-281: Enhance StreamReceiver with validation
- Line 304-320: Enhance splits receivers with validation
- Line 3984-4053: Modify receive_streams event emission
- Add new tests in test module (starts around line 4706)

## Error Handling

### Issue #609

- Use existing `VestFlowError::AmountZero` for zero/negative rates
- May need to add `VestFlowError::WeightZero` for splits weight validation

### Issue #610

- No new errors needed (event emission is informational)

## Backwards Compatibility

### Issue #609

- Validation changes could break existing code if invalid data was previously accepted
- Need to ensure all existing call sites pass valid data

### Issue #610

- Event structure change is backwards compatible for indexers
- Adding fields to event value is non-breaking
- Conditional emission (no event when cycles=0) is a behavioral change but safe

## Success Criteria

### Issue #609

- [ ] StreamReceiver validation rejects zero/negative rates with typed error
- [ ] SplitsReceiver validation rejects zero weight with typed error
- [ ] All validation tests pass
- [ ] Structs remain usable from SDK bindings
- [ ] No regression in existing functionality

### Issue #610

- [ ] stream_received event emitted with correct shape after receive_streams
- [ ] Event NOT emitted when cycles_processed = 0
- [ ] Event topics include account and token as specified
- [ ] Event value includes cycles_processed (u32) and amount_received (i128)
- [ ] All event tests pass
- [ ] No regression in streaming functionality

## Timeline

- Implementation plan review: ~1 day
- Issue #610 implementation + tests: ~2-3 hours
- Issue #609 implementation + tests: ~4-6 hours
- Total: ~1.5 days

## Notes

- Both issues are contract-level changes requiring Rust/Soroban knowledge
- Testing will use the existing test framework (visible from line 4706 onwards)
- Event emission pattern follows existing conventions in the codebase
- Validation pattern should follow Soroban best practices for typed errors
