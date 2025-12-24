# Rollback Procedure Documentation

## Overview
The rollback system provides safe, reliable reversion to previous application versions with zero downtime.

## Rollback Types

### 1. Automatic Rollback (To Previous Version)
**Trigger**: ACTION=ROLLBACK with empty ROLLBACK_VERSION
**Behavior**:
- Identifies current active version
- Finds previous chronological version
- Validates previous version exists
- Performs rollback
- Removes failed current version

**Safety Checks**:
- Prevents multiple automatic rollbacks in sequence
- Requires manual intervention after auto-rollback
- Validates target version integrity

### 2. Manual Rollback (Specific Version)
**Trigger**: ACTION=ROLLBACK with specific ROLLBACK_VERSION
**Behavior**:
- Validates specified version exists
- Performs rollback to specified version
- Preserves current version (for potential reversion)

**Use Cases**:
- Revert to known stable version
- Test specific historical version
- Emergency recovery to any point
