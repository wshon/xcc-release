# Implementation Plan

- [x] 1. Write bug condition exploration test
  - **Property 1: Fault Condition** - New Connection Password Saved
  - **CRITICAL**: This test MUST FAIL on unfixed code - failure confirms the bug exists
  - **DO NOT attempt to fix the test or the code when it fails**
  - **NOTE**: This test encodes the expected behavior - it will validate the fix when it passes after implementation
  - **GOAL**: Surface counterexamples that demonstrate the bug exists
  - **Scoped PBT Approach**: Scope the property to concrete failing cases - new SSH connections with non-empty passwords
  - Test that for any new connection (connection_info is None) with password authentication and non-empty password field, build_ssh_config creates SshAuthMethod::Password with encrypted password (not None)
  - Test cases: password "test123", long password (50 chars), special characters "p@$w0rd!"
  - Run test on UNFIXED code
  - **EXPECTED OUTCOME**: Test FAILS (this is correct - it proves the bug exists)
  - Document counterexamples found: built config contains `password: None` when password field is non-empty
  - Mark task complete when test is written, run, and failure is documented
  - _Requirements: 1.1, 2.1, 2.2_

- [x] 2. Write preservation property tests (BEFORE implementing fix)
  - **Property 2: Preservation** - Non-Buggy Password Handling
  - **IMPORTANT**: Follow observation-first methodology
  - Observe behavior on UNFIXED code for non-buggy inputs (edit mode, empty passwords, other auth methods)
  - Write property-based tests capturing observed behavior patterns:
    - Edit existing connection without modifying password field → existing password preserved
    - Edit existing connection with modified password field → new password saved
    - New connection with empty password field → password: None
    - Private key authentication → private key handling unchanged
    - Agent authentication → agent handling unchanged
  - Property-based testing generates many test cases for stronger guarantees
  - Run tests on UNFIXED code
  - **EXPECTED OUTCOME**: Tests PASS (this confirms baseline behavior to preserve)
  - Mark task complete when tests are written, run, and passing on unfixed code
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

- [x] 3. Fix for new connection password not being saved

  - [x] 3.1 Implement the fix in build_ssh_config method
    - Modify password handling logic in `src/ui/sheets/connection/ssh/ssh_form.rs` (lines 360-377)
    - Add check for new connection: `self.connection_info.is_none()`
    - For new connections with non-empty password: always encrypt and save password regardless of has_been_modified flag
    - For editing existing connections: preserve existing logic using has_been_modified flag
    - Handle empty password case: create SshAuthMethod with password: None
    - Include error handling for encryption failures with fallback to plaintext
    - _Bug_Condition: isBugCondition(input) where input.connection_info IS None AND input.auth_method == AuthMethod::Password AND input.password_field IS NOT empty AND input.password_state.has_been_modified == false_
    - _Expected_Behavior: For new connections with non-empty password, encrypt password using SecureKeyManager and create SshAuthMethod::Password with encrypted password value_
    - _Preservation: Edit mode password preservation, empty password handling, other auth methods, modified password in edit mode_
    - _Requirements: 1.1, 2.1, 2.2, 2.3, 3.1, 3.2, 3.3, 3.4, 3.5_

  - [x] 3.2 Verify bug condition exploration test now passes
    - **Property 1: Expected Behavior** - New Connection Password Saved
    - **IMPORTANT**: Re-run the SAME test from task 1 - do NOT write a new test
    - The test from task 1 encodes the expected behavior
    - When this test passes, it confirms the expected behavior is satisfied
    - Run bug condition exploration test from step 1
    - **EXPECTED OUTCOME**: Test PASSES (confirms bug is fixed)
    - Verify that new connections with passwords now have encrypted password in built config
    - _Requirements: 2.1, 2.2, 2.3_

  - [x] 3.3 Verify preservation tests still pass
    - **Property 2: Preservation** - Non-Buggy Password Handling
    - **IMPORTANT**: Re-run the SAME tests from task 2 - do NOT write new tests
    - Run preservation property tests from step 2
    - **EXPECTED OUTCOME**: Tests PASS (confirms no regressions)
    - Confirm all tests still pass after fix (no regressions)
    - Verify edit mode, empty passwords, and other auth methods work identically to unfixed code
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

- [x] 4. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.
