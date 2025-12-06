# Solis Modbus Performance Analysis

## Current Performance Issue

**Observed**: ~70% CPU usage on dual Intel Xeon E5-2650 v4 @ 2.20GHz (vs ~25% without modbus)

## Root Causes

### 1. **Excessive Event Bus Activity** (PRIMARY ISSUE)

**Location**: `data_retrieval.py:218`

```python
self.hass.bus.async_fire(DOMAIN, {REGISTER: reg, VALUE: corrected_value, CONTROLLER: self.controller.host, SLAVE: self.controller.slave})
```

**Problem**: This fires an HA event for **EVERY SINGLE REGISTER** read!

**Impact**:
- 7 FAST groups polled every 5 seconds
- 2 NORMAL groups polled every 15 seconds
- 18 SLOW groups polled every 30 seconds
- Assuming ~100-150 total registers across all groups
- **Per minute**: ~1,800+ event bus fires (7 groups × ~50 regs × 12 times/min)
- Each event must be processed, dispatched, and potentially trigger entity updates

**Why this is expensive**:
- Event bus processing overhead
- Callback dispatching to all listeners
- Potential entity state updates triggering more events
- Memory allocations for event objects
- Serialization/deserialization overhead

### 2. **Debug Logging Overhead**

**Location**: Multiple places in `data_retrieval.py`

Lines with frequent debug logging:
- Line 195: Per sensor group
- Line 204: When None received
- Line 207: When value count mismatch
- Line 215: **FOR EVERY REGISTER VALUE** - `f"block {start_register}, register {reg} has value {value}"`
- Line 258-266: Spike filtering logs

**Problem**: Even if debug logging is disabled, the f-string formatting still happens!

```python
_LOGGER.debug(f"block {start_register}, register {reg} has value {value}")
```

This creates strings for hundreds of registers per poll cycle, even when not displayed.

### 3. **Polling Frequency**

**Current Defaults**:
- FAST: 5 seconds (7 sensor groups)
- NORMAL: 15 seconds (2 sensor groups)
- SLOW: 30 seconds (18 sensor groups)

**Per Minute**:
- FAST: 12 polls × 7 groups = 84 read operations
- NORMAL: 4 polls × 2 groups = 8 read operations
- SLOW: 2 polls × 18 groups = 36 read operations
- **Total**: ~128 modbus read operations per minute

### 4. **Serial Communication Overhead**

**For RS485 specifically**:
- 9600 baud is relatively slow
- Each modbus transaction has overhead:
  - Request transmission time
  - Inverter processing time
  - Response transmission time
  - Inter-frame delays (modbus_controller.py line 263: 0.05s between operations)

**Calculation** (approximate):
- At 9600 baud, ~960 bytes/second
- Each modbus read: ~10-20 bytes total (request + response)
- Plus mandatory 50ms inter-frame delay
- Per read: ~70-100ms total
- 128 reads/minute = ~8-13 seconds of serial communication per minute

## Performance Optimization Recommendations

### Priority 1: Remove Unnecessary Event Bus Fires (CRITICAL)

**Current code** (`data_retrieval.py:218`):
```python
for i, value in enumerate(values):
    reg = start_register + i
    _LOGGER.debug(f"block {start_register}, register {reg} has value {value}")
    corrected_value = self.spike_filtering(reg, value)
    cache_save(self.hass, reg, corrected_value)
    self.hass.bus.async_fire(DOMAIN, {REGISTER: reg, VALUE: corrected_value, ...})  # ❌ FIRES FOR EVERY REGISTER
```

**Solution 1**: Only fire events when values actually change
```python
for i, value in enumerate(values):
    reg = start_register + i
    corrected_value = self.spike_filtering(reg, value)
    old_value = cache_get(self.hass, reg)
    cache_save(self.hass, reg, corrected_value)

    # Only fire event if value changed
    if old_value != corrected_value:
        self.hass.bus.async_fire(DOMAIN, {REGISTER: reg, VALUE: corrected_value, ...})
```

**Expected Impact**: 70-90% reduction in event fires (most register values don't change every poll)

**Solution 2**: Batch events (more advanced)
```python
changed_registers = []
for i, value in enumerate(values):
    reg = start_register + i
    corrected_value = self.spike_filtering(reg, value)
    old_value = cache_get(self.hass, reg)
    cache_save(self.hass, reg, corrected_value)
    if old_value != corrected_value:
        changed_registers.append((reg, corrected_value))

# Fire single event with all changes
if changed_registers:
    self.hass.bus.async_fire(DOMAIN, {
        "registers": changed_registers,
        CONTROLLER: self.controller.host,
        SLAVE: self.controller.slave
    })
```

**Expected Impact**: 95%+ reduction in event fires

### Priority 2: Fix Debug Logging

**Replace**:
```python
_LOGGER.debug(f"block {start_register}, register {reg} has value {value}")
```

**With**:
```python
if _LOGGER.isEnabledFor(logging.DEBUG):
    _LOGGER.debug("block %s, register %s has value %s", start_register, reg, value)
```

**Why**: String formatting only happens if debug is enabled

**Expected Impact**: 5-10% CPU reduction when debug logging is off

### Priority 3: Increase Poll Intervals (User Configuration)

**Current**:
- FAST: 5s → **Recommend: 10s**
- NORMAL: 15s → **Keep**
- SLOW: 30s → **Keep or increase to 60s**

**Why**: Most solar data doesn't change that rapidly. Even PV power can be polled every 10s without losing meaningful information.

**Expected Impact**: 30-50% reduction in modbus transactions

### Priority 4: Add Poll Throttling

Add a minimum time between polls even if HA scheduler triggers faster:

```python
async def get_modbus_updates(self, groups: List[SolisSensorGroup], speed: PollSpeed):
    # Add minimum interval check
    last_poll = self._last_poll.get(speed, 0)
    min_interval = {
        PollSpeed.FAST: 5,
        PollSpeed.NORMAL: 15,
        PollSpeed.SLOW: 30
    }.get(speed, 15)

    if time.time() - last_poll < min_interval:
        return

    self._last_poll[speed] = time.time()
    # ... rest of code
```

### Priority 5: Reduce Serial Overhead (RS485 Specific)

**Option A**: Increase baudrate (if inverter supports it)
- Try 19200 baud instead of 9600
- Cuts transmission time in half

**Option B**: Reduce inter-frame delay (currently 50ms)
```python
# modbus_controller.py line 263
await asyncio.sleep(0.025)  # Try 25ms instead of 50ms
```

**Option C**: Batch register reads more aggressively
- Combine adjacent sensor groups into larger reads
- Trade off: may read some unnecessary registers

## Comparison: HA Native Modbus vs Solis Integration

### HA Native Modbus
- **Typically** polls only on-demand or with user-configured intervals
- No automatic event bus firing for every register
- Simpler code path
- **But**: Requires manual configuration for 100+ entities

### Solis Integration
- **Pros**: Automatic entity discovery, rich features, time sync, etc.
- **Cons**: Aggressive polling, excessive event firing

## Recommended Implementation Plan

### Phase 1: Quick Wins (Immediate, 50% improvement)
1. Add "only fire events on change" logic
2. Fix debug logging string formatting
3. Increase default FAST polling to 10s

### Phase 2: Medium Term (Additional 20% improvement)
4. Add user-configurable poll intervals in HA UI
5. Reduce inter-frame delays
6. Add poll throttling

### Phase 3: Advanced (Additional 10-20% improvement)
7. Batch event firing
8. Optimize register reading strategy
9. Add connection pooling / persistent connections

## Testing Methodology

### Before Changes:
```bash
# Monitor CPU usage
top -b -n 60 -d 1 | grep homeassistant > cpu_before.log

# Count event fires
journalctl -u homeassistant -f | grep "solis_modbus" | grep "async_fire" | wc -l
```

### After Changes:
- Same tests
- Compare CPU usage (target: <40% from current 70%)
- Compare event fire count (target: 90% reduction)

## Expected Results

**After all optimizations**:
- CPU usage: 70% → **30-35%** (50% reduction)
- Event fires: ~1,800/min → **~100/min** (95% reduction)
- Memory usage: Slight reduction from fewer event objects
- Latency: Same or better (fewer events to process)

## Serial vs TCP Performance

**RS485 Serial**:
- Limited by baud rate (9600 = ~960 bytes/sec)
- Higher latency per transaction
- But more stable/reliable connection

**TCP (WiFi Dongles)**:
- Much faster (100+ KB/sec)
- Lower latency
- But potential WiFi instability

**Recommendation**: If using Serial, optimizations are even more critical. Consider increasing to 19200 baud if inverter supports it.
