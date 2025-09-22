# Fix: Locale-independent date parsing for reliable operation on international Windows deployments

## Summary

This PR addresses a locale-dependent date parsing issue that can prevent expired firewall rules from being properly removed on Windows servers configured with non-English regional settings. The fix ensures consistent behavior across all supported Windows locales by implementing culture-invariant date parsing.

## Problem Statement

### Issue Description
On Windows servers with non-English locales (Italian, German, French, etc.), the current date parsing implementation in `unban_old_records()` relies on PowerShell's locale-dependent type conversion, which can cause:
- Expired IP bans to remain active indefinitely
- Inconsistent behavior across different server configurations
- Potential parsing errors on certain date formats

### Root Cause Analysis
**Affected Code** (Line 286, original implementation):
```powershell
if ($([int]([datetime]$ReleaseDate- (Get-Date)).TotalSeconds) -lt 0)
```

**Technical Issue**: The casting `[datetime]$ReleaseDate` uses `System.Globalization.CultureInfo.CurrentCulture` for string-to-date conversion. While `firewall_add()` consistently generates ISO-formatted dates (`yyyy-MM-dd HH:mm:ss`), the parsing depends on system locale settings.

## Solution

### Implementation Details

1. **New Helper Function** - Centralized culture-invariant parsing:
```powershell
function ParseDateInvariant([String] $DateString, [String] $Format = "yyyy-MM-dd HH:mm:ss") {
    try {
        return [datetime]::ParseExact($DateString, $Format, [System.Globalization.CultureInfo]::InvariantCulture)
    } catch {
        throw "Failed to parse date '$DateString' with format '$Format': $($_.Exception.Message)"
    }
}
```

2. **Updated `unban_old_records()`** - Simplified and robust:
```powershell
# Before
$ReleaseDateParsed = [datetime]::ParseExact($ReleaseDate, "yyyy-MM-dd HH:mm:ss", [System.Globalization.CultureInfo]::InvariantCulture)

# After  
$ReleaseDateParsed = ParseDateInvariant $ReleaseDate
```

3. **Enhanced `firewall_add()`** - Consistent formatting:
```powershell
# Before
$Expire = (get-date $ExpireDate -format u).replace("Z","")

# After
$Expire = $ExpireDate.ToString("yyyy-MM-dd HH:mm:ss", [System.Globalization.CultureInfo]::InvariantCulture)
```

### Key Improvements
- ✅ **Locale Independence**: Consistent behavior across all Windows regional settings
- ✅ **Centralized Logic**: Single function handles all date parsing operations
- ✅ **Enhanced Error Handling**: Detailed error messages for debugging
- ✅ **Better Logging**: Debug output includes parsing details and time calculations
- ✅ **Maintainability**: Future date handling changes centralized in one location

## Testing

### Test Environment
- **OS**: Windows Server 2019 Standard
- **Locale**: Italian (it-IT)  
- **PowerShell**: 5.1.17763.5830
- **Regional Settings**: Italian date/time formats

### Test Cases Executed

1. **Date Parsing Validation**:
```powershell
Test Dates: 2025-09-22 18:58:08, 2025-12-25 23:59:59, 2025-01-01 00:00:00
✅ All dates parsed correctly with InvariantCulture
✅ Round-trip consistency verified (format → parse → format)
✅ Comparison with locale-dependent parsing confirmed reliability
```

2. **Firewall Rule Lifecycle**:
```powershell
✅ Rule creation with correct ISO date format
✅ Expiration checking with culture-invariant parsing
✅ Automatic cleanup of expired rules
✅ Debug logging provides clear time calculations
```

3. **Error Handling**:
```powershell
✅ Malformed date strings produce clear error messages
✅ Parsing failures don't crash the main loop
✅ Warnings logged for debugging purposes
```

### Performance Impact
- **Minimal**: `ParseDateInvariant()` adds negligible overhead compared to original casting
- **Memory**: No additional memory footprint
- **Compatibility**: Zero breaking changes to existing functionality

## Backward Compatibility

This change maintains full backward compatibility:
- ✅ Existing firewall rules continue to work without modification
- ✅ Same date format stored in rule descriptions
- ✅ Identical behavior on English-locale systems
- ✅ No changes to command-line interface or configuration

## Files Modified

- `wail2ban.ps1`: 
  - Added `ParseDateInvariant()` helper function
  - Updated `unban_old_records()` to use culture-invariant parsing
  - Enhanced `firewall_add()` for consistent date formatting
  - Improved debug logging with detailed time calculations

## Code Quality

This implementation follows PowerShell best practices:
- Uses explicit culture specification for international applications
- Implements proper error handling with meaningful messages
- Maintains clean separation of concerns with helper functions
- Includes comprehensive logging for operational visibility

## Acknowledgment

Thank you for creating and maintaining this excellent tool. wail2ban fills a critical security gap for Windows environments, and your implementation is both elegant and effective. This fix ensures it works reliably across the global Windows server ecosystem.

## Checklist

- [x] Code follows project style guidelines
- [x] Self-review completed
- [x] No breaking changes introduced  
- [x] Tested on target environment (Windows Server 2019 Italian)
- [x] All test cases pass
- [x] Error handling verified
- [x] Documentation updated (inline comments)
- [x] Backward compatibility maintained

---

**Risk Assessment**: Low - Changes are isolated to date parsing logic with comprehensive fallback handling.

**Deployment Notes**: No special deployment considerations required. Change takes effect immediately upon script restart.
