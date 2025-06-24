---
marp: true
theme: default
class: lead
paginate: true
backgroundColor: #fff
backgroundImage: url('https://marp.app/assets/hero-background.svg')
---

# How to Avoid Writing Brittle Unit Tests

*Building Robust and Maintainable Test Suites*

---

## What Are Brittle Tests?

- Tests that **break easily** when code changes
- **Over-specified** tests that test implementation details
- Tests with **tight coupling** to internal structures
- Tests that are **hard to maintain** and understand

---

## The Problem with Our Example Test

```python
def test_enable_cpu_watchdog_failure_to_start_service():
    with patch('postupgrade_actions.get_hwsku', return_value='Celestica-E1031-T48S4'), \
         patch('postupgrade_actions.get_sonic_version_info', return_value={'build_version': '20181130.70'}), \
         patch('postupgrade_actions.exec_cmd') as mock_exec_cmd, \
         patch('postupgrade_actions.start_systemd_service', return_value=False) as mock_start_service:

        enable_CPU_watchdog()

        # Too specific about implementation!
        mock_exec_cmd.assert_has_calls([
            call("sudo cp /mocked/path/cpu_wdt /usr/local/bin/"),
            call("sudo chmod a+x /usr/local/bin/cpu_wdt"),
            call("sudo cp /mocked/path/cpu_wdt.service /lib/systemd/system/")
        ], any_order=False)
```

**Issues**: Testing exact shell commands, rigid order enforcement, coupling to implementation details

---

## Problem #1: Testing Implementation Details

**❌ Bad: Testing exact commands and their order**
```python
# The function needs hardware/version checks to run - that's fine!
with patch('postupgrade_actions.get_hwsku', return_value='Celestica-E1031-T48S4'), \
     patch('postupgrade_actions.get_sonic_version_info', return_value={'build_version': '20181130.70'}):

    # But testing exact shell commands is brittle
    mock_exec_cmd.assert_has_calls([
        call("sudo cp /mocked/path/cpu_wdt /usr/local/bin/"),
        call("sudo chmod a+x /usr/local/bin/cpu_wdt"),
        call("sudo cp /mocked/path/cpu_wdt.service /lib/systemd/system/")
    ], any_order=False)
```

---

## Better Approaches to Testing Commands

**✅ Option 1: Test that key operations happened**
```python
mock_exec_cmd.assert_any_call(unittest.mock.ANY)  # Something was executed
assert any('cpu_wdt' in str(call) for call in mock_exec_cmd.call_args_list)
assert mock_exec_cmd.call_count >= 2  # Multiple setup steps
```

---

## More Better Approaches

**✅ Option 2: Test logical groupings**
```python
file_operations = [call for call in mock_exec_cmd.call_args_list
                  if 'cp' in str(call) or 'chmod' in str(call)]
assert len(file_operations) >= 2  # Files were copied and permissions set
```

**✅ Option 3: Test patterns, not exact strings**
```python
commands = [str(call) for call in mock_exec_cmd.call_args_list]
assert any('cpu_wdt' in cmd and '/usr/local/bin' in cmd for cmd in commands)
assert any('chmod' in cmd and 'cpu_wdt' in cmd for cmd in commands)
```

---

## The Real Issues with This Test

1. **Tests shell command details** - What if we change from `cp` to `install`?
2. **Enforces command order** - Does the exact sequence matter?
3. **Hardcodes file paths** - Tied to specific directory structure
4. **Tests "how" not "what"** - Focuses on implementation, not behavior

**We can still test commands, just more flexibly!**

---

## More Flexible Command Testing Strategies

Let's explore better ways to test commands without being brittle...

---

## Strategy 1: Test Intent, Not Implementation

```python
# Option A: Test that the right files are targeted for installation
commands = [str(call) for call in mock_exec_cmd.call_args_list]
binary_installed = any('cpu_wdt' in cmd and 'bin' in cmd for cmd in commands)
service_installed = any('cpu_wdt.service' in cmd for cmd in commands)
assert binary_installed and service_installed

# Option B: Mock higher-level functions (if they exist)
with patch('postupgrade_actions.copy_file') as mock_copy:
    enable_CPU_watchdog()
    # Test that files were copied to correct destinations
    copy_calls = mock_copy.call_args_list
    destinations = [call[0][1] for call in copy_calls]  # Second arg is destination
    assert any('/usr/local/bin' in dest for dest in destinations)
```

---

## Strategy 2: Test Command Categories

```python
# Group commands by purpose
copy_commands = [c for c in mock_exec_cmd.call_args_list if 'cp' in str(c)]
permission_commands = [c for c in mock_exec_cmd.call_args_list if 'chmod' in str(c)]
assert len(copy_commands) >= 1  # Files were copied
assert len(permission_commands) >= 1  # Permissions were set

# Test side effects we can observe
with patch('postupgrade_actions.log_info') as mock_log:
    enable_CPU_watchdog()
    # Check that installation steps were logged
    logged_messages = [call[0][0] for call in mock_log.call_args_list]
    assert any('installing' in msg.lower() for msg in logged_messages)
```

---

## Strategy 3: Test Critical Patterns

```python
# Test the essential parts that must be correct
commands = [str(call) for call in mock_exec_cmd.call_args_list]
assert any('cpu_wdt' in cmd and 'bin' in cmd for cmd in commands)  # Binary installed
assert any('service' in cmd for cmd in commands)  # Service file handled

# Use custom helper functions for readability
def contains_file_operation(calls, filename, operation):
    """Helper to check if a file operation happened"""
    return any(operation in str(call) and filename in str(call) for call in calls)

# Now the test is more readable
calls = mock_exec_cmd.call_args_list
assert contains_file_operation(calls, 'cpu_wdt', 'cp')
assert contains_file_operation(calls, 'cpu_wdt', 'chmod')
```

---

## Problem #2: Rigid Order Dependencies

**❌ Bad: Enforcing exact order when it doesn't matter**
```python
mock_exec_cmd.assert_has_calls([...], any_order=False)
```

**✅ Good: Test logical dependencies only**
```python
# Test that installation was attempted, order usually doesn't matter
assert mock_exec_cmd.call_count == 3
# Only test order if it's critical for correctness
if order_matters:
    assert mock_start_service.called_after(mock_copy_files)
```

---

## Problem #3: Hardcoded Implementation Commands

**❌ Bad: Testing exact shell commands**
```python
# These exact commands could change, breaking the test
mock_exec_cmd.assert_has_calls([
    call("sudo cp /mocked/path/cpu_wdt /usr/local/bin/"),
    call("sudo chmod a+x /usr/local/bin/cpu_wdt"),
    call("sudo cp /mocked/path/cpu_wdt.service /lib/systemd/system/")
])
```

**✅ Good: Test at a higher level of abstraction**
```python
# Mock the higher-level operation instead
with patch('postupgrade_actions.install_watchdog_files') as mock_install:
    enable_CPU_watchdog()
    mock_install.assert_called_once()
```

---

## Better Test Structure

```python
def test_cpu_watchdog_handles_service_startup_failure():
    """Test that CPU watchdog gracefully handles service startup failures."""

    # Arrange: Set up the failure condition
    with patch('module.start_systemd_service', return_value=False) as mock_start:

        # Act: Execute the function
        result = enable_CPU_watchdog()

        # Assert: Verify the behavior, not implementation
        assert result is False
        assert mock_start.called
        # Verify error was logged (outcome, not specific message)
        assert "error" in get_last_log_message().lower()
```

---

## Principle #1: Test Behavior, Not Implementation

**Focus on:**
- What the function **should do**
- The **contract** it fulfills
- **Side effects** that matter to users

**Avoid:**
- Internal method calls
- Exact parameter values
- Implementation details

---

## Principle #2: Test at the Right Level of Abstraction

**Choose the appropriate level to test:**
- **Too low**: Testing exact shell commands, specific parameter values
- **Too high**: Testing only final outcomes without verifying key operations
- **Just right**: Testing the contract and observable behavior

---

## Abstraction Level Examples

```python
# Too low - brittle
mock_exec_cmd.assert_has_calls([
    call("sudo cp /path/cpu_wdt /usr/local/bin/"),
    call("sudo chmod a+x /usr/local/bin/cpu_wdt")
])

# Too high - not enough verification
result = enable_CPU_watchdog()
assert result is True  # But did installation actually happen?
```

---

## Just Right - Balanced Testing

```python
# Just right - flexible but meaningful
assert any('cpu_wdt' in str(call) for call in mock_exec_cmd.call_args_list)
assert mock_exec_cmd.call_count >= 2  # Multiple operations happened
result = enable_CPU_watchdog()
assert result is True  # AND verify the outcome
```

---

## By the Way: Arrange-Act-Assert Pattern

*Not directly about brittleness, but helps with test clarity*

```python
def test_cpu_watchdog_installs_on_compatible_hardware():
    # Arrange: Set up test conditions
    hardware = "Celestica-E1031-T48S4"
    version = {"build_version": "20181130.70"}

    # Act: Execute the behavior being tested
    with patch_compatible_system(hardware, version):
        result = enable_CPU_watchdog()

    # Assert: Verify the outcome
    assert any('cpu_wdt' in str(call) for call in mock_exec_cmd.call_args_list)
    assert result is True
```

**Why it helps**: Clear structure makes it easier to identify what makes tests brittle

---

## Refactored Example Test

```python
class TestCPUWatchdogEnabling:

    @pytest.fixture
    def compatible_system(self):
        with patch('module.get_hwsku', return_value='Celestica-E1031-T48S4'), \
             patch('module.get_sonic_version_info', return_value={'build_version': '20181130.70'}):
            yield

    def test_returns_false_when_service_startup_fails(self, compatible_system):
        """CPU watchdog should return False when systemd service fails to start."""

        with patch('module.start_systemd_service', return_value=False):
            result = enable_CPU_watchdog()

        assert result is False

    def test_logs_error_when_service_startup_fails(self, compatible_system):
        """CPU watchdog should log error when systemd service fails to start."""

        with patch('module.start_systemd_service', return_value=False), \
             patch('module.log_error') as mock_log_error:

            enable_CPU_watchdog()

        mock_log_error.assert_called_once()
        error_message = mock_log_error.call_args[0][0]
        assert "could not be enabled" in error_message.lower()
```

---

## Rules of Thumb for Robust Tests

**Ask yourself these questions:**

1. **Does my test break if I add a harmless `echo "debug info"`?**
   - If yes, you're testing too specifically

2. **Does my test break if I change `cp` to `install`?**
   - If yes, you're coupled to implementation details

3. **Does my test break if I reorder independent operations?**
   - If yes, you're over-specifying sequence

---

## More Rules of Thumb

4. **Would my test still pass if the function did nothing?**
   - If yes, you're not testing enough

5. **Can I understand what this test is checking in 10 seconds?**
   - If no, it's probably too complex or poorly named

**Simple rule**: Test *what* the function accomplishes, not *how* it does it.

---

# Thank You!

*Questions? Discussion?*
