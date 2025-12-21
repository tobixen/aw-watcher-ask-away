# Comprehensive Code Review: aw-watcher-ask-away

**Date:** 2025-12-18
**Reviewer:** Claude (AI-assisted code review)
**Test Status:** ✅ All 13 tests passing

## Overview
This is a well-structured ActivityWatch watcher that prompts users to log what they were doing during AFK periods. The codebase is generally clean, well-tested, and follows good Python practices.

---

## 1. CODE QUALITY AND BEST PRACTICES

### Strengths
- **Good separation of concerns**: Core logic, UI, and configuration are properly separated
- **Type hints**: Most functions have type annotations (matches user's global instructions)
- **Proper use of modern Python features**: Pattern matching, f-strings, type unions with `|`
- **Clean imports**: Well-organized and using standard import practices
- **Linting setup**: Uses ruff and black with comprehensive rule sets

### Issues Found

#### 1.1 Missing Return Type Annotations (Per User Instructions)
**Priority:** CRITICAL (per user's `~/.claude/CLAUDE.md` requirements)

**File**: `src/aw_watcher_ask_away/core.py`
- Line 30: `find_afk_bucket(buckets: dict[str, Any])` → should be `-> str`
- Line 40: `find_lid_bucket(buckets: dict[str, Any])` → should be `-> str | None`
- Line 53: `is_afk(event: aw_core.Event) -> bool` ✓ (has annotation)
- Line 66: `get_utc_now()` → should be `-> datetime.datetime`
- Line 70: `get_gaps(events: list[aw_core.Event])` → should be `-> Iterator[aw_core.Event]`
- Line 110: `post_event(self, event: aw_core.Event, message: str)` → should be `-> None`
- Line 114: `get_new_afk_events_to_note(self, seconds: float, durration_thresh: float)` → should be `-> Iterator[aw_core.Event] | None`

**File**: `src/aw_watcher_ask_away/dialog.py`
- Line 19: `open_link(link: str)` → should be `-> None`
- Lines 31, 38, 46, 50, 60, 63, 77, 81, 96, etc.: Many methods missing return types
- Line 364: `ask_string(title: str, prompt: str, history: list[str])` → should be `-> str | None`

**File**: `src/aw_watcher_ask_away/__main__.py`
- Line 24: `prompt(event: aw_core.Event, recent_events: Iterable[aw_core.Event])` → should be `-> str | None`
- Line 34: `get_state_retries(client: ActivityWatchClient, enable_lid_events: bool = True)` → should be `-> AWAskAwayClient`
- Line 50: `main()` → should be `-> None`

**File**: `src/aw_watcher_ask_away/config.py`
- Line 23: `load_config() -> dict` ✓ (has annotation, but could be more specific: `-> dict[str, Any]`)

#### 1.2 Typo in Parameter Name
**Location**: `src/aw_watcher_ask_away/core.py:114`
```python
def get_new_afk_events_to_note(self, seconds: float, durration_thresh: float):
```
**Issue**: `durration_thresh` should be `duration_thresh` (double 'r')
**Impact**: This typo is propagated to line 96 in `__main__.py` as well
**Severity**: Low (functional but poor naming consistency)

#### 1.3 Inconsistent String Quotes
**Location**: `src/aw_watcher_ask_away/__main__.py:31`
```python
[event.data.get(DATA_KEY, '') for event in recent_events]
```
**Issue**: Uses single quotes `''` while most of the codebase uses double quotes `""`
**Recommendation**: Enable black's string normalization or be consistent with quote style
**Severity**: Very low (stylistic)

---

## 2. POTENTIAL BUGS OR EDGE CASES

### 2.1 Critical: Race Condition in Event Detection
**Location**: `src/aw_watcher_ask_away/core.py:143-150`
```python
# Check if currently AFK (from either source)
# Most recent event is LAST after sorting (ascending order)
if all_events:
    most_recent = all_events[-1]  # Last element is most recent
    currently_afk = is_afk(most_recent)
    if currently_afk:
        # Currently AFK, wait to bring up the prompt
        return
```
**Issue**: If the most recent event is old and indicates AFK, but the user has since returned, this will incorrectly skip prompting

**Scenario**:
1. User goes AFK at 10:00 AM
2. User returns at 10:30 AM but no new event has been generated yet
3. Code checks at 10:31 AM and sees the last event is AFK from 10:00
4. Incorrectly waits instead of prompting

**Recommendation**: Check if the most recent event is recent enough (e.g., within the last `frequency` seconds)
```python
if all_events:
    most_recent = all_events[-1]
    event_age = get_utc_now() - (most_recent.timestamp + most_recent.duration)
    if is_afk(most_recent) and event_age < datetime.timedelta(seconds=frequency * 2):
        # Currently AFK, wait to bring up the prompt
        return
```
**Severity**: Medium

### 2.2 Assertion in Production Code
**Location**: `src/aw_watcher_ask_away/core.py:192`
```python
def add_event(self, event: aw_core.Event, message: str):
    assert not self.has_event(event)  # noqa: S101
```
**Issue**: Using `assert` in production code. Assertions can be disabled with `python -O`
**Impact**: If Python is run with optimization, duplicate events could be added
**Recommendation**: Replace with explicit check and raise exception
```python
def add_event(self, event: aw_core.Event, message: str) -> None:
    if self.has_event(event):
        raise AWWatcherAskAwayError(f"Event already exists: {event}")
    event.data[DATA_KEY] = message
    event["id"] = None
    logger.debug(f"Posting event: {event}")
    self.recent_events.append(event)
```
**Severity**: High

### 2.3 Potential IndexError with Empty History
**Location**: `src/aw_watcher_ask_away/dialog.py:295-301`
```python
def previous_entry(self, event=None):  # noqa: ARG002
    self.history_index = max(0, self.history_index - 1)
    self.set_text(self.history[self.history_index])

def next_entry(self, event=None):  # noqa: ARG002
    self.history_index = min(len(self.history) - 1, self.history_index + 1)
    self.set_text(self.history[self.history_index])
```
**Issue**: If `history` is empty, `self.history[self.history_index]` will raise `IndexError`
**Scenario**: First time user with no history presses up/down arrow
**Recommendation**: Add guard check
```python
def previous_entry(self, event=None):
    if not self.history:
        return
    self.history_index = max(0, self.history_index - 1)
    self.set_text(self.history[self.history_index])
```
**Severity**: Critical (causes crash on first use)

### 2.4 Infinite Loop Risk in Retry Logic
**Location**: `src/aw_watcher_ask_away/__main__.py:39-47`
```python
for _ in range(10):
    try:
        return AWAskAwayClient(client, enable_lid_events=enable_lid_events)
    except ConnectionError:
        logger.exception("Cannot connect to client.")
        time.sleep(10)  # 10 * 10 = wait for 100s before giving up.
raise AWWatcherAskAwayError("Could not get a connection to the server.")
```
**Issue**: Only catches `ConnectionError`, but `AWAskAwayClient.__init__` can raise `AWWatcherAskAwayError` (e.g., if AFK bucket not found)
**Impact**: If `find_afk_bucket` raises `AWWatcherAskAwayError`, it will propagate immediately instead of retrying
**Recommendation**: Either:
1. Catch broader exceptions, or
2. Document that only connection issues are retried

**Severity**: Low (design decision, but should be documented)

### 2.5 Missing Validation in Abbreviation Pattern
**Location**: `src/aw_watcher_ask_away/dialog.py:264`
```python
if not re.fullmatch(r"\w+", abbr):
    messagebox.showerror("Invalid abbreviation", "Abbreviations must be alphanumeric and without spaces.")
    return
```
**Issue**: Empty string validation happens after strip, but the regex doesn't enforce non-empty
**Edge Case**: If user enters only whitespace, `abbr.strip()` becomes `""`, which fails the regex but with confusing error message
**Recommendation**: Add explicit empty check
```python
if not abbr or not expansion:
    messagebox.showerror("Invalid input", "Abbreviation and expansion cannot be empty.")
    return
if not re.fullmatch(r"\w+", abbr):
    messagebox.showerror("Invalid abbreviation", "Abbreviations must be alphanumeric and without spaces.")
    return
```
**Severity**: Low

### 2.6 Division by Zero Potential
**Location**: `src/aw_watcher_ask_away/core.py:187`
```python
if overlap / new.duration > overlap_thresh:
```
**Issue**: If `new.duration` is zero (which can happen according to line 219's comment), this will raise `ZeroDivisionError`
**Note**: Events with zero duration are filtered at line 219, but this is in a different method
**Recommendation**: Add defensive check
```python
def has_event(self, new: aw_core.Event, overlap_thresh: float = 0.95) -> bool:
    if new.duration.total_seconds() == 0:
        return False

    for recent in self.recent_events:
        if recent.duration.total_seconds() == 0:
            continue
        # ... rest of logic
```
**Severity**: Medium

---

## 3. PERFORMANCE CONCERNS

### 3.1 Repeated Event Sorting
**Location**: `src/aw_watcher_ask_away/core.py:71, 141, 223`
**Issue**: Events are sorted multiple times with `aw_transform.sort_by_timestamp()`
- Line 141: Sorts all events from both AFK and lid watchers
- Line 223: Sorts non-AFK events again in `get_unseen_afk_events`
- Line 71: Sorts in `get_gaps`

**Impact**: O(n log n) sorting operations repeated unnecessarily
**Recommendation**: Sort once and pass sorted lists, or document that inputs should be pre-sorted
**Severity**: Low (event lists are small, max 10 items)

### 3.2 Deep Copy in squash_overlaps
**Location**: `src/aw_watcher_ask_away/core.py:62-63`
```python
def squash_overlaps(events: list[aw_core.Event]) -> list[aw_core.Event]:
    # Make a deep copy because the period_union function edits the events instead of returning new ones.
    return aw_transform.sort_by_timestamp(aw_transform.period_union(deepcopy(events), []))
```
**Issue**: Deep copying all events for every call
**Impact**: Unnecessary memory allocation and CPU cycles
**Recommendation**: This is acceptable given the comment explains it's needed due to `period_union` mutating inputs. However, consider filing an issue with aw-core to make `period_union` non-mutating
**Severity**: Very low

### 3.3 Deque with Fixed Size Not Enforced
**Location**: `src/aw_watcher_ask_away/core.py:88-90, 160`
```python
recent_events = deque(maxlen=10)
recent_events.extend(aw_transform.sort_by_timestamp(client.get_events(self.bucket_id, limit=10)))
self.state = AWAskAwayState(recent_events)
```
**And**:
```python
self.recent_events = recent_events if isinstance(recent_events, deque) else deque(recent_events, 10)
```
**Issue**: In `AWAskAwayState.__init__`, if a deque is passed, it's used as-is even if it doesn't have `maxlen=10`
**Recommendation**: Always ensure maxlen is set
```python
def __init__(self, recent_events: Iterable[aw_core.Event]):
    if isinstance(recent_events, deque):
        self.recent_events = deque(recent_events, maxlen=10)
    else:
        self.recent_events = deque(recent_events, maxlen=10)
```
**Severity**: Low

---

## 4. SECURITY ISSUES

### 4.1 No Input Sanitization for User Messages
**Location**: `src/aw_watcher_ask_away/core.py:193`
```python
event.data[DATA_KEY] = message
```
**Issue**: User input from dialog is stored directly without sanitization
**Impact**:
- Potential for extremely long strings causing storage issues
- Potential for special characters breaking JSON serialization
- No limit on message length

**Recommendation**: Add length validation and sanitization
```python
def add_event(self, event: aw_core.Event, message: str) -> None:
    if self.has_event(event):
        raise AWWatcherAskAwayError(f"Event already exists: {event}")

    # Limit message length to prevent abuse
    MAX_MESSAGE_LENGTH = 10000
    if len(message) > MAX_MESSAGE_LENGTH:
        logger.warning(f"Message truncated from {len(message)} to {MAX_MESSAGE_LENGTH} characters")
        message = message[:MAX_MESSAGE_LENGTH]

    event.data[DATA_KEY] = message
    # ... rest
```
**Severity**: Low (local application, but good practice)

### 4.2 Hardcoded Web Interface URL
**Location**: `src/aw_watcher_ask_away/dialog.py:310`
```python
def open_web_interface(self, event=None):  # noqa: ARG002
    open_link("http://localhost:5600/#/timeline")
```
**Issue**:
- Hardcoded HTTP (not HTTPS)
- Hardcoded port may not match user's configuration
- Hardcoded localhost won't work with remote servers

**Recommendation**: Make configurable or read from ActivityWatch config
**Severity**: Low (convenience feature only)

### 4.3 No Validation on Loaded JSON Config
**Location**: `src/aw_watcher_ask_away/dialog.py:42`
```python
try:
    self.update(json.load(f))
except json.JSONDecodeError:
    logger.exception("Failed to load abbreviations from config file.")
```
**Issue**: Loads arbitrary JSON from user config without validation
**Impact**: Malformed data could break abbreviations system
**Recommendation**: Add schema validation
```python
try:
    data = json.load(f)
    if not isinstance(data, dict):
        logger.error("Abbreviations config must be a dictionary")
        return
    if not all(isinstance(k, str) and isinstance(v, str) for k, v in data.items()):
        logger.error("All abbreviations must be string key-value pairs")
        return
    self.update(data)
except json.JSONDecodeError:
    logger.exception("Failed to load abbreviations from config file.")
```
**Severity**: Low

---

## 5. DOCUMENTATION QUALITY

### Strengths
- **Good docstrings**: Most classes and complex functions have docstrings
- **Inline comments**: Complex logic is well-explained (e.g., overlap detection rationale)
- **README**: Comprehensive with setup instructions, configuration, and integrations
- **TODO documentation**: Excellent `docs/TODO.md` tracking future work

### Issues Found

#### 5.1 Missing Docstrings
**Locations**:
- `src/aw_watcher_ask_away/core.py:66` - `get_utc_now()` has no docstring
- `src/aw_watcher_ask_away/core.py:110` - `post_event()` has no docstring
- `src/aw_watcher_ask_away/dialog.py:19` - `open_link()` has no docstring
- Many dialog methods lack docstrings

**Recommendation**: Add docstrings to all public methods
**Severity**: Low

#### 5.2 Incorrect/Misleading Parameter Documentation
**Location**: `src/aw_watcher_ask_away/core.py:198-208`
```python
def get_unseen_afk_events(self, events: list[aw_core.Event], recency_thresh: float, durration_thresh: float):
    """Check whether we recently finished a large AFK event.

    Parameters
    ----------
    events : list[aw_core.Event]
        The events to check for AFK events.
    seconds : float    # <-- WRONG parameter name
        Events more than this many seconds ago will be ignored.
    durration_thresh : float
        Events with a durration less than this many seconds will be ignored.
    """
```
**Issue**: Documentation says `seconds` but parameter is `recency_thresh`
**Recommendation**: Update to match actual parameter names
**Severity**: Medium

#### 5.3 Missing Documentation for Lid Watcher Integration
**Location**: Core.py lacks documentation explaining how lid events work
**Issue**: While the code handles lid events, there's no comprehensive docstring explaining:
- What `system-afk` status means
- How lid events differ from regular AFK
- Why events are merged

**Recommendation**: Add module-level or class-level docstring explaining the lid integration
**Severity**: Low

---

## 6. TEST COVERAGE GAPS

### Current Test Status
✓ All 13 tests pass
✓ Good coverage of core logic (double-ask prevention, gaps, overlaps)
✓ Good coverage of lid integration
✓ Good coverage of config loading

### Missing Test Coverage

#### 6.1 No Tests for Dialog/UI Code
**Location**: `src/aw_watcher_ask_away/dialog.py`
**Missing**:
- Abbreviation expansion logic
- History navigation (up/down arrows)
- Keyboard shortcuts
- Abbreviation saving/loading
- Configuration dialog

**Recommendation**: Add unit tests for dialog logic (can be tested without UI):
```python
def test_abbreviation_expansion():
    # Test that abbreviations are expanded correctly
    store = _AbbreviationStore()
    store["mtg"] = "meeting"
    # Test expansion logic
```
**Severity**: Medium

#### 6.2 No Tests for __main__.py
**Location**: `src/aw_watcher_ask_away/__main__.py`
**Missing**:
- `prompt()` function formatting
- `get_state_retries()` retry logic
- Command-line argument parsing
- Config file override behavior

**Recommendation**: Add integration tests
**Severity**: Medium

#### 6.3 No Tests for Error Conditions
**Missing**:
- HTTPError handling in `get_new_afk_events_to_note`
- ConnectionError handling in `get_state_retries`
- Exception handling in main loop (line 102)
- Invalid config file handling

**Recommendation**: Add error scenario tests
```python
def test_get_events_handles_http_error():
    # Mock client to raise HTTPError
    # Verify graceful handling
```
**Severity**: Medium

#### 6.4 No Tests for Edge Cases
**Missing tests for**:
- Empty event lists
- Zero-duration events
- Events with missing data fields
- Overlapping events from both AFK and lid watchers
- Time zone edge cases
- Very long messages

**Severity**: Medium

#### 6.5 Bug in _debug_utils.py
**Location**: `src/aw_watcher_ask_away/_debug_utils.py:32`
```python
if __name__:  # <-- BUG: Should be __name__ == "__main__"
    find_overlapping_events()
```
**Issue**: The condition `if __name__:` is always True (string is truthy)
**Impact**: Debug code likely isn't working as intended
**Severity**: Medium (debug tool)

---

## 7. ADDITIONAL RECOMMENDATIONS

### 7.1 Consider Adding Logging Levels
**Issue**: Most logging is at DEBUG level, but important user-facing events use INFO
**Recommendation**: Review and standardize logging levels:
- ERROR: Failures that prevent functionality
- WARNING: Recoverable issues (lid watcher not found, HTTP errors)
- INFO: Normal operation milestones
- DEBUG: Detailed diagnostic info

### 7.2 Add Type Checking to CI/CD
**Recommendation**: Add mypy to the test suite in `pyproject.toml`
```toml
[tool.hatch.envs.default]
dependencies = ["coverage[toml]>=6.5", "pytest", "mypy"]
```

### 7.3 Consider Adding Pre-commit Hooks
**Recommendation**: Add pre-commit configuration for:
- black formatting
- ruff linting
- type checking
- test execution

### 7.4 Improve Error Messages
**Location**: `src/aw_watcher_ask_away/core.py:33`
```python
raise AWWatcherAskAwayError("Cannot find the afk bucket.")
```
**Recommendation**: Include available buckets in error message for easier debugging
```python
raise AWWatcherAskAwayError(
    f"Cannot find the afk bucket. Available buckets: {list(buckets.keys())}"
)
```

### 7.5 Consider Using Enum for Status Values
**Location**: Status strings like "afk", "not-afk", "system-afk" are hardcoded
**Recommendation**: Define an enum
```python
from enum import Enum

class AFKStatus(str, Enum):
    AFK = "afk"
    NOT_AFK = "not-afk"
    SYSTEM_AFK = "system-afk"
```

### 7.6 Configuration Dialog Doesn't Check enable_lid_events
**Location**: `ConfigDialog` only shows abbreviations tab
**Issue**: The `enable_lid_events` config option can only be edited manually in TOML file
**Recommendation**: Add a settings tab with checkboxes for boolean config options
**Severity**: Low (quality of life)

---

## PRIORITY SUMMARY

### Critical (Fix Immediately)
1. ⚠️ **Empty history IndexError** (Section 2.3) - Will crash on first use
2. ⚠️ **Missing return type annotations** (Section 1.1) - Per user requirements

### High Priority
1. **Assertion in production code** (Section 2.2) - Can be disabled with `-O`
2. **Race condition in AFK detection** (Section 2.1) - May miss prompts
3. **Division by zero in overlap detection** (Section 2.6)
4. **Fix `_debug_utils.py` if statement** (Section 6.5)

### Medium Priority
1. **Typo: durration → duration** (Section 1.2)
2. **Add test coverage for dialog.py** (Section 6.1)
3. **Add test coverage for __main__.py** (Section 6.2)
4. **Add error condition tests** (Section 6.3)
5. **Input validation for messages** (Section 4.1)
6. **Fix docstring parameter names** (Section 5.2)

### Low Priority
1. **Performance optimizations** (Section 3)
2. **Add missing docstrings** (Section 5.1)
3. **Security hardening** (Section 4.2, 4.3)
4. **Code quality improvements** (Section 7)

---

## CONCLUSION

This is a well-crafted project with good architecture and testing. The main concerns are:
1. Missing return type annotations (per user requirements)
2. A few critical bugs (empty history crash, assertion in production)
3. Test coverage gaps for UI and error conditions
4. Some edge cases that need handling

The code demonstrates good understanding of Python best practices, proper use of type hints, and thoughtful handling of complex event overlap scenarios. With the fixes suggested above, this would be production-ready code.

**Overall Grade:** B+ (would be A with critical fixes applied)
