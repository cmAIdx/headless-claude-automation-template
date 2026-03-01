# User Authentication

## Overview

Add email/password authentication so users can create accounts and log in. This is the first step toward personalized features.

## Requirements

- Sign-up form with email, password, and password confirmation
- Login form with email and password
- Password must be at least 8 characters with one number and one uppercase letter
- On successful login, redirect to the dashboard
- On failed login, show an inline error (no alerts)
- Store passwords hashed with bcrypt (never plain text)
- Add a `/api/auth/login` POST endpoint and a `/api/auth/register` POST endpoint
- Add a session cookie (httpOnly, secure, sameSite: strict)

## Acceptance Criteria

- A new user can register, log out, and log back in
- Duplicate email registration returns a 409 error
- Invalid credentials return a 401 error with a generic message (no "email not found" vs "wrong password" distinction)
- Passwords are bcrypt-hashed in the database
- Session cookie is set on login and cleared on logout

## Constraints

- Use the existing database connection (see CLAUDE.md for details)
- No third-party auth providers - just email/password for now
- All form validation must happen both client-side and server-side
