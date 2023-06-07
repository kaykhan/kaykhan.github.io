---
layout: post
title:  "Brute Force Protection in a NodeJs Auth API using MySQL Part 1"
date:   2023-01-10 12:00:00 +0100
categories: auth api mysql nodejs 
---

In cryptography, a brute-force attack consists of an attacker submitting many passwords or passphrases with the hope of eventually guessing correctly. The attacker systematically checks all possible passwords and passphrases until the correct one is found.

A bad actor may want too gain entry into your api or web application by brute forcing a login request. Brute forcing a login request is when someone
attempts many consecutive requests to try and guess correct combination of username and password. Alternatively a user may have forgotten there 
password, repeated incorrect login attempts should lock the user out and dispatch a forgot password email.

## Goal

Lock out a user's account if there are too many failed login attempts. A question we have to ask is
    - How many failed login attempts are allowed (x) within however long of a timeframe (y) and how long should they be locked out for (z).

* If a user enters the wrong password 10 times in a 5 minute period they should be locked out for 15 minutes.

## Considerations

1. We should create an audit of authentication attempt
2. We should seperatly from the audit when a failed attempt is made and keep track of how many failed attempts are made.
1. A user should be locked out regardless if that user account exists or not to prevent information gathering by a bad actor.
2. A forgot password email should be dispatched if the user exists.

### Part 1 - MySQL Table Setup

For this approach we will need 2 tables. `authentication_attempt` & `authentication_attempt_status`

1. Authentication Attempt

The `authentication_attempt` is an audit table which stores every login attempt. We store values like the `username` and `success` which indicates
if the request was successfull or not.

```
CREATE TABLE `authentication_attempt` (
  `id` int NOT NULL AUTO_INCREMENT,
  `username` varchar(255) DEFAULT NULL,
  `success` tinyint DEFAULT NULL,
  `reason` varchar(255) DEFAULT NULL,
  `ip_address` varchar(255) DEFAULT NULL,
  `created_at` datetime(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  `updated_at` datetime(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  `deleted_at` datetime(6) DEFAULT NULL,
  `authentication_id` varchar(36) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `fk_8a7976a855bfdc513cb39ec9bc21b1b7` (`authentication_id`),
  CONSTRAINT `fk_8a7976a855bfdc513cb39ec9bc21b1b7` FOREIGN KEY (`authentication_id`) REFERENCES `authentication` (`id`) ON DELETE CASCADE
) ENGINE=InnoDB AUTO_INCREMENT=26 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```
2. Authentication Attempt Status

This table is used to keep track of locked account

```
CREATE TABLE `authentication_attempt_status` (
  `id` int NOT NULL AUTO_INCREMENT,
  `username` varchar(255) NOT NULL,
  `failed_attempts` int DEFAULT NULL,
  `last_failed_attempt` timestamp NULL DEFAULT NULL,
  `unlock_at` timestamp NULL DEFAULT NULL,
  `created_at` datetime(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  `updated_at` datetime(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  `deleted_at` datetime(6) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `IDX_7b4c6e6dcc65fbd1bb9e6aefb0` (`username`)
) ENGINE=InnoDB AUTO_INCREMENT=24 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

### Part 2 - Insert login attempt

```
export const AddAuthenticationAttempt = async (authAttempt: IAddAuthenticationAttempt) => {
    const [user, userError] = await LoginGetUser(authAttempt.request);

    const attempt: IAuthAttempt = {
        username: authAttempt.request.username,
        authentication_id: user?.auth_id,
        success: authAttempt.success,
        ip_address: authAttempt.ip_address,
        reason: authAttempt.reason?.errors[0]?.message || null,
    };

    const [authAttemptStatus, attemptStatusError] = await GetAuthAttemptStatus(authAttempt.request.username);

    const authStatus = authAttemptStatus && authAttemptStatus[0] ? authAttemptStatus[0] : null;

    // insert audit entry into authentication_attempts
    const [response, error] = await InsertAuthAttempt(attempt);

    // get list of latest auth attempts
    const [listAuthAttempts, listAuthAttemptErrors] = await GetAuthAttemptsForUser(authAttempt.request.username);

    // if the request was successfull no need to continue
    if (attempt.success) {
        return [null, null];
    }

    // increment auth status entry
    const currentFailedAttempts = isDateOlderThanXMinutes(authStatus?.last_failed_attempt, 5) ? 0 : authStatus?.failed_attempts || 0;
    const updatedFailedAttempts = (listAuthAttempts?.length ?? 0) > currentFailedAttempts ? currentFailedAttempts + 1 : 1;
    const upsertPayloadAuthAttemptStatus: any = {
        username: authAttempt.request.username,
        failed_attempts: updatedFailedAttempts,
        last_failed_attempt: listAuthAttempts ? listAuthAttempts[0]?.created_at : null,
        unlock_at: SetAuthStatusUnlockAt(updatedFailedAttempts, authStatus?.unlock_at),
        locked_email_sent_at: authStatus?.locked_email_sent_at,
    };

    const [upsert, upsertError] = await UpsertAuthAttemptStatus(upsertPayloadAuthAttemptStatus);

    // if unlock_at value is set and no email has been sent before send a locked account email
    if (upsertPayloadAuthAttemptStatus.unlock_at && !authStatus?.locked_email_sent_at) {
        const request: IDispatchLockedAccountEmailRequest = {
            email: user?.email as string,
            name: (user?.display_name as string) || (user?.first_name as string) || (user?.email as string),
            lock_time: 15,
        };

        const [dispatch, dispatchError] = await DispatchLockedAccountEmailRequest(request);
        const [_, __] = await UpdateAuthAttemptStatus({ username: authStatus?.username as string, locked_email_sent_at: new Date() });

        const errorPayload = {
            status: HttpStatusCode.BAD_REQUEST,
            errors: [{ kind: IAuthMethod.LoginUser, message: IAuthErrorKind.AUTH_LOCKED }],
        };
        return [null, errorPayload] as const;
    }
    return [null, error] as const;
};
```
