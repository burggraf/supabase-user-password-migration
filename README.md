# supabase-user-password-migration
How to migrate user passwords from an external authentication system to Supabase

## Scenario
- You've migrated users from another platform to Supabase (GoTrue)
- These users were authenticated using an email + password combination in the previous platform
- You'd like to authenticate these users in your Supabase application **without**:
  - any additional steps on the part of the user
  - the user even needing to know their account has been migrated (a seamless transition)
  
## Plain-text passwords scenario
If your prior platform did not hash the passwords and stored them as plain-text (which is highly unlikely) then this is a pretty easy task.  In this case, you have access to the users' passwords and you can create the new user accounts in Supabase with the passwords intact.  Simply use the [createUser()](https://supabase.com/docs/reference/javascript/auth-api-createuser) function when creating your users, passing in the user's password.

## Hashed passwords with the same hashing algorithm
It's far more likely that your prior platform hashed the user passwords.  If the hashing algorithm used by your prior platform was exactly the same as the Supabase (GoTrue) algorithm, you could move the passwords over by writing the hashed passwords directly in your `auth.users` table in Supabase.  This, however, is also highly unlikely.

## Hashed passwords with two different algorithms
Most-likely, your prior platform hashed the user passwords using a different algorithm than Supabase (GoTrue) uses.  In this (most common) case, you should migrate your users to Supabase with a random password, but also store the **old password hash** in your Supabase database as a temporary step.  Users who have an **old password hash** in their user records are considered `not yet migrated`.  Once the password has been migrated to the Supabase (GoTrue) format, you will delete the **old password hash** and the user will be considered `migrated`.

### The workflow for a single user
-  The user was migrated from platform A to Supabase, `old_password_hash` as stored as 'XXXX-XXXX-XXXX'
-  After some time, the user tries to sign into your Supabase application
-  Your application sign in screen gathers the email address and password from the user
-  Your middleware function checks to see if the user is `migrated` (meaning, the password has been migrated)
  -  If YES: the user is passed on to the normal Supabase [signIn()](https://supabase.com/docs/reference/javascript/auth-signin) flow
  -  If NO: then your middleware function runs the old password hashing algorithm against the password entered by the user to see if it matches the `old_password_hash`
    - If it DOES NOT MATCH: the sign in is rejected
    - If it MATCHES: 
      - The current password entered by the user is written to Supabase (GoTrue) [See update()](https://supabase.com/docs/reference/javascript/auth-update#update-password-for-authenticated-user)
      - The **old password hash** field is removed from the user, thus indicated that this user's password is now `migrated`

#### Where to store the **old password hash**
You can store the **old password hash** in any of:
- A simple PostgreSQL table (email text primary key, password_hash text)
- The [user's metadata](https://supabase.com/docs/reference/javascript/auth-update#update-a-users-metadata) (note: this is accessible / readable by the user by default and is stored in `auth.users.raw_user_meta_data`)
- User's app metadata (`auth.users.raw_app_meta_data`) (note: this field is not accessible / readable by the user)

## Requirements
In order to create this seamless password migration system, you will need the following:
- user accounts converted to Supabase (GoTrue)
- a stored (temporary) `old_password_hash` field containing the user's password hash from the prior platform
- knowledge of the algorithm used to hash the passwords on the prior platform
- a **validation function** to validate the password against the `old_password_hash`
- a middleware tier (this can be anywhere -- Supabase Edge Functions (Deno), a hosted NodeJS application, or any server-based function tier)

### The Validation Function
This function should be created and tested separately with a known account that contains a password you know.  All this function needs to do is to:
- accept a password (entered by the user) and the `old_password_hash` for the account
- return **TRUE** if the password hash matches the `old_password_hash`
- return **FALSE** if the password hash does not match the `old_password_hash`

You'll need knowledge of the old hashing algorithm in order to make this validation function work.

## Example Implementation
Let's say I've migrated all my users to Supabase, and I've created a table called `old_password_hashes` with a record for each user.  Now, as an un-migrated user I sign in for the first time.  I enter my email address and password in the app's **Sign In** screen.

1. `email` and `password` are sent to the middle tier:
2. the middle tier looks up the user in the `old_password_hashes` table
  - NOT FOUND? then send `email` and `password` to the standard `signIn()` function (see **note** below)
  - FOUND? then send send `password` and `old_password_hash` to the **password validation function**
3. **password validation function**
  - FAILED? reject the sign in
  - SUCCESS? store the new password and sign the user in
    - call `update()` to update the user's password in Supabase
    - delete the corresponding `old_password_hash` record for this user
    - call `signIn()` to sign the user in (see **note** below)
        
**Note:** For this last step, signing the user in, this is a client-side function.  Since all of this processing has been done on the server side, you'll want to return control back to the client side here and let the client side call `signIn()` directly.  So your middleware function should simply return TRUE or FALSE back to your client application.  FALSE means the sign in failed (missing user or bad password).  TRUE means the user's password was either already migrated, or it was just successfuly migrated (it doesn't matter).  If the result is TRUE, we call `signIn()` from the client to complete the sign in process.

