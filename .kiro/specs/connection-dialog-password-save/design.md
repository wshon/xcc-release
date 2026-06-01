# Connection Dialog Password Save Bugfix Design

## Overview

When creating a new SSH connection with password authentication, the password entered by the user is not saved with the connection info. The root cause is that the `PasswordState` component initializes `has_been_modified` to `false` for new connections, and the `build_ssh_config` method incorrectly interprets this as "password unchanged" rather than "new password entered". This causes the system to create a connection with `password: None` even when the user has entered a password.

The fix requires modifying the password handling logic in `build_ssh_config` to distinguish between three scenarios:
1. New connection with password entered (should save the password)
2. Editing existing connection without modifying password field (should preserve existing password)
3. Editing existing connection with modified password field (should save new password)

## Glossary

- **Bug_Condition (C)**: The condition that triggers the bug - when a user creates a new SSH connection with password authentication and enters a password
- **Property (P)**: The desired behavior when C(X) holds - the password should be encrypted and saved with the connection
- **Preservation**: Existing password handling behavior for edit mode and other auth methods that must remain unchanged
- **PasswordState**: The component in `src/ui/components/password_input/password_input.rs` that manages password input state
- **has_been_modified**: A boolean flag in PasswordState that tracks whether the password field has been edited
- **is_edit_mode**: A boolean flag in PasswordState that indicates whether editing an existing password (true) or creating new (false when has_existing_password is true)
- **has_existing_password**: A boolean flag in PasswordState that indicates whether there's a saved password
- **build_ssh_config**: The method in `SshConnectionForm` that constructs the SSH configuration from form inputs
- **connection_info**: An optional field in SshConnectionForm that is None for new connections and Some for editing existing connections

## Bug Details

### Fault Condition

The bug manifests when a user creates a new SSH connection (not editing an existing one), selects password authentication, enters a password in the password field, and clicks "Save and Connect". The `build_ssh_config` method incorrectly treats the password as "unmodified" because `has_been_modified` is false, leading it to create `SshAuthMethod::new_password_plaintext(None)` instead of encrypting and saving the entered password.

**Formal Specification:**
```
FUNCTION isBugCondition(input)
  INPUT: input of type ConnectionFormSubmission
  OUTPUT: boolean
  
  RETURN input.connection_info IS None  // New connection, not editing
         AND input.auth_method == AuthMethod::Password
         AND input.password_field IS NOT empty
         AND input.password_state.has_been_modified == false
         AND input.action == "Save and Connect"
END FUNCTION
```

### Examples

- **New connection with password**: User creates new SSH connection, enters "mypassword123" in password field, clicks "Save and Connect" → System creates connection with `password: None` instead of encrypted password
- **New connection with empty password**: User creates new SSH connection, leaves password field empty, clicks "Save and Connect" → System correctly creates connection with `password: None` (allows prompt on connect)
- **Edit existing connection without changing password**: User edits existing connection with saved password, doesn't modify password field, clicks "Save and Connect" → System correctly preserves existing encrypted password
- **Edit existing connection with new password**: User edits existing connection, clicks edit button on password field, enters new password, clicks "Save and Connect" → System correctly encrypts and saves new password

## Expected Behavior

### Preservation Requirements

**Unchanged Behaviors:**
- Editing an existing connection with a saved password where the password field has not been modified must continue to preserve the existing encrypted password
- Creating a new SSH connection with an empty password field must continue to create a connection with `password: None`
- Creating a new SSH connection with private key authentication must continue to handle private key and passphrase correctly
- Creating a new SSH connection with SSH Agent authentication must continue to handle agent authentication correctly
- Editing an existing connection and modifying the password field must continue to encrypt and save the new password value

**Scope:**
All inputs that do NOT involve creating a new connection with a non-empty password field should be completely unaffected by this fix. This includes:
- Editing existing connections (where `connection_info` is Some)
- Creating connections with empty password fields
- Creating connections with other authentication methods (PrivateKey, Agent)
- All private key passphrase handling logic

## Hypothesized Root Cause

Based on the code analysis, the root cause is in the `build_ssh_config` method in `src/ui/sheets/connection/ssh/ssh_form.rs` (lines 328-505):

1. **Incorrect Logic for New Connections**: The method checks `!self.password_state.read(cx).has_been_modified` to decide whether to preserve the existing password. However, for new connections, `has_been_modified` is initialized to `false` in `PasswordState::new()` (line 83), even when the user enters a password.

2. **Missing Context Differentiation**: The logic doesn't distinguish between:
   - New connection (where `self.connection_info` is None) with password entered
   - Editing existing connection (where `self.connection_info` is Some) without modifying password

3. **PasswordState Initialization**: When creating a new connection, `PasswordState::new()` is called with `has_existing_password: false`, which sets `is_edit_mode: true` and `has_been_modified: false`. The `has_been_modified` flag only becomes true when `on_input_change()` is called, but this may not be triggered reliably for all input scenarios.

4. **Conditional Branch Selection**: The code at lines 360-377 takes the first branch (preserve existing password) when `has_been_modified` is false, even for new connections where there is no existing password to preserve.

## Correctness Properties

Property 1: Fault Condition - New Connection Password Saved

_For any_ connection form submission where the user is creating a new SSH connection (connection_info is None) with password authentication and a non-empty password field, the fixed build_ssh_config function SHALL encrypt the password using SecureKeyManager and create SshAuthMethod::Password with the encrypted password value.

**Validates: Requirements 2.1, 2.2, 2.3**

Property 2: Preservation - Existing Connection Password Preserved

_For any_ connection form submission where the user is editing an existing connection (connection_info is Some) with password authentication and has NOT modified the password field (has_been_modified is false), the fixed build_ssh_config function SHALL produce the same result as the original function, preserving the existing encrypted password from the original connection.

**Validates: Requirements 3.1**

Property 3: Preservation - Empty Password Handling

_For any_ connection form submission where the password field is empty (regardless of new or edit mode), the fixed build_ssh_config function SHALL produce the same result as the original function, creating SshAuthMethod with password: None.

**Validates: Requirements 3.2**

Property 4: Preservation - Other Auth Methods

_For any_ connection form submission where the authentication method is NOT Password (PrivateKey or Agent), the fixed build_ssh_config function SHALL produce the same result as the original function, handling private keys, passphrases, and agent authentication identically.

**Validates: Requirements 3.3, 3.4**

Property 5: Preservation - Modified Password in Edit Mode

_For any_ connection form submission where the user is editing an existing connection and HAS modified the password field (has_been_modified is true), the fixed build_ssh_config function SHALL produce the same result as the original function, encrypting and saving the new password value.

**Validates: Requirements 3.5**

## Fix Implementation

### Changes Required

Assuming our root cause analysis is correct:

**File**: `src/src/ui/sheets/connection/ssh/ssh_form.rs`

**Function**: `build_ssh_config` (lines 328-505)

**Specific Changes**:

1. **Add New Connection Check**: Modify the password handling logic to first check if this is a new connection (`self.connection_info.is_none()`)

2. **Separate New Connection Logic**: For new connections with non-empty password, always encrypt and save the password, regardless of `has_been_modified` flag

3. **Preserve Existing Edit Logic**: Keep the existing logic for editing connections (when `self.connection_info.is_some()`) unchanged

4. **Update Conditional Structure**: Replace the current logic (lines 360-377) with:
   ```rust
   let auth = match self.auth_method {
       AuthMethod::Password => {
           let password = self.password_state.read(cx).get_value(cx);
           
           // Check if this is a new connection
           if self.connection_info.is_none() {
               // New connection: save password if provided
               if password.is_empty() {
                   SshAuthMethod::new_password_plaintext(None)
               } else {
                   match SshAuthMethod::new_password_secure(&password, &self.key_manager) {
                       Ok(auth_method) => auth_method,
                       Err(e) => {
                           tracing::warn!("Failed to encrypt password: {}, falling back to plaintext", e);
                           SshAuthMethod::new_password_plaintext(Some(password.to_string()))
                       }
                   }
               }
           } else {
               // Editing existing connection: use has_been_modified flag
               if !self.password_state.read(cx).has_been_modified {
                   // Password not modified, preserve existing
                   if let Some(ref connection) = self.connection_info {
                       if let ConnectionConfig::SSH(ssh_config) = &connection.config {
                           ssh_config.auth.clone()
                       } else {
                           SshAuthMethod::new_password_plaintext(None)
                       }
                   } else {
                       SshAuthMethod::new_password_plaintext(None)
                   }
               } else if password.is_empty() {
                   SshAuthMethod::new_password_plaintext(None)
               } else {
                   match SshAuthMethod::new_password_secure(&password, &self.key_manager) {
                       Ok(auth_method) => auth_method,
                       Err(e) => {
                           tracing::warn!("Failed to encrypt password: {}, falling back to plaintext", e);
                           SshAuthMethod::new_password_plaintext(Some(password.to_string()))
                       }
                   }
               }
           }
       }
       // ... rest of auth methods unchanged
   };
   ```

5. **No Changes to PasswordState**: The PasswordState component behavior should remain unchanged to preserve existing functionality

## Testing Strategy

### Validation Approach

The testing strategy follows a two-phase approach: first, surface counterexamples that demonstrate the bug on unfixed code, then verify the fix works correctly and preserves existing behavior.

### Exploratory Fault Condition Checking

**Goal**: Surface counterexamples that demonstrate the bug BEFORE implementing the fix. Confirm that new connections with passwords fail to save the password value.

**Test Plan**: Write tests that create new SSH connections with password authentication, enter passwords, and verify that the built connection config contains the encrypted password. Run these tests on the UNFIXED code to observe failures and confirm the root cause.

**Test Cases**:
1. **New Connection with Password**: Create new connection, set password "test123", build config → will fail on unfixed code (password will be None)
2. **New Connection with Empty Password**: Create new connection, leave password empty, build config → should pass even on unfixed code (password should be None)
3. **New Connection with Long Password**: Create new connection, set password with 50 characters, build config → will fail on unfixed code (password will be None)
4. **New Connection with Special Characters**: Create new connection, set password with special chars "p@$$w0rd!", build config → will fail on unfixed code (password will be None)

**Expected Counterexamples**:
- Built SSH config contains `SshAuthMethod::Password { password: None }` when password field is non-empty
- Root cause confirmed: `has_been_modified` is false for new connections, causing wrong branch to execute

### Fix Checking

**Goal**: Verify that for all inputs where the bug condition holds (new connections with passwords), the fixed function produces the expected behavior (encrypted password saved).

**Pseudocode:**
```
FOR ALL input WHERE isBugCondition(input) DO
  result := build_ssh_config_fixed(input)
  ASSERT result.auth matches SshAuthMethod::Password { password: Some(encrypted_value) }
  ASSERT encrypted_value can be decrypted to original password
END FOR
```

### Preservation Checking

**Goal**: Verify that for all inputs where the bug condition does NOT hold, the fixed function produces the same result as the original function.

**Pseudocode:**
```
FOR ALL input WHERE NOT isBugCondition(input) DO
  ASSERT build_ssh_config_original(input) = build_ssh_config_fixed(input)
END FOR
```

**Testing Approach**: Property-based testing is recommended for preservation checking because:
- It generates many test cases automatically across the input domain
- It catches edge cases that manual unit tests might miss
- It provides strong guarantees that behavior is unchanged for all non-buggy inputs

**Test Plan**: Observe behavior on UNFIXED code first for edit mode scenarios and other auth methods, then write property-based tests capturing that behavior.

**Test Cases**:
1. **Edit Mode Password Preservation**: Create connection with password, edit without modifying password field → verify existing password preserved on both unfixed and fixed code
2. **Edit Mode Password Modification**: Create connection with password, edit and modify password field → verify new password saved on both unfixed and fixed code
3. **Private Key Auth Preservation**: Create connection with private key auth → verify private key handling identical on both unfixed and fixed code
4. **Agent Auth Preservation**: Create connection with agent auth → verify agent handling identical on both unfixed and fixed code

### Unit Tests

- Test new connection creation with password (various password values)
- Test new connection creation with empty password
- Test editing existing connection without modifying password
- Test editing existing connection with modified password
- Test password encryption and decryption round-trip
- Test fallback to plaintext when encryption fails

### Property-Based Tests

- Generate random passwords (various lengths, character sets) for new connections and verify all are encrypted and saved correctly
- Generate random connection edit scenarios and verify preservation of existing passwords when not modified
- Generate random auth method combinations and verify non-password auth methods are unaffected
- Test that empty password fields always result in `password: None` across many scenarios

### Integration Tests

- Test full flow: create new connection with password → save → retrieve from storage → verify password can be decrypted
- Test full flow: create connection → edit without changing password → save → verify password unchanged
- Test full flow: create connection → edit and change password → save → verify new password saved
- Test connection establishment with saved password works correctly
