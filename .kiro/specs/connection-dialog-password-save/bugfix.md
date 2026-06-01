# Bugfix Requirements Document

## Introduction

When creating a new SSH connection in the connection dialog, users enter a password in the password field and click "Save and Connect". However, the password is not saved with the connection info, causing the system to prompt for the password again after attempting to connect. This bug affects the user experience by requiring duplicate password entry and defeats the purpose of the "Save and Connect" action.

The bug occurs specifically when:
- Creating a new SSH connection (not editing an existing one)
- Using password authentication method
- Entering a password in the password field
- Clicking "Save and Connect" button

## Bug Analysis

### Current Behavior (Defect)

1.1 WHEN a user creates a new SSH connection with password authentication and enters a password in the password field and clicks "Save and Connect" THEN the system creates a connection with `auth: Password { password: None }` instead of including the entered password

1.2 WHEN the system attempts to connect with the saved connection that has `password: None` THEN the connection fails with "Authentication failed: Password is required but not provided"

1.3 WHEN the connection fails due to missing password THEN the system prompts the user to enter the password again (duplicate entry)

### Expected Behavior (Correct)

2.1 WHEN a user creates a new SSH connection with password authentication and enters a password in the password field and clicks "Save and Connect" THEN the system SHALL capture the password value from the PasswordState component and include it in the connection info

2.2 WHEN the system builds the SSH connection config for a new connection with a non-empty password field THEN the system SHALL encrypt the password using SecureKeyManager and create `SshAuthMethod::Password` with the encrypted password

2.3 WHEN the system attempts to connect with the saved connection that includes the encrypted password THEN the connection SHALL succeed without prompting for password again

### Unchanged Behavior (Regression Prevention)

3.1 WHEN editing an existing connection with a saved password and the password field has not been modified THEN the system SHALL CONTINUE TO preserve the existing encrypted password

3.2 WHEN a user creates a new SSH connection with password authentication but leaves the password field empty THEN the system SHALL CONTINUE TO create a connection with `password: None` (allowing password prompt on connect)

3.3 WHEN a user creates a new SSH connection with private key authentication THEN the system SHALL CONTINUE TO handle private key and passphrase correctly

3.4 WHEN a user creates a new SSH connection with SSH Agent authentication THEN the system SHALL CONTINUE TO handle agent authentication correctly

3.5 WHEN editing an existing connection and modifying the password field THEN the system SHALL CONTINUE TO encrypt and save the new password value
