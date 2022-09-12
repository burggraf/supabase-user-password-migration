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
It's far more like that your prior platform hashed the user passwords.  If the hashing algorithm used by your prior platform was exactly the same as the Supabase (GoTrue) algorithm, you could move the passwords over by writing the hashed passwords directly in your `auth.users` table in Supabase.  This, however, is also highly unlikely.

## Hashed passwords with two different algorithms
Most-likely, your prior platform hashed the user passwords using a different algorithm than Supabase (GoTrue) uses.  In this (most common) case, you should migrate your users to Supabase with a random password, but also store the **old password hash** in your Supabase database as a temporary step.  Users who have an **old password hash** in their user records are considered `not yet migrated`.  Once the password has been migrated to the Supabase (GoTrue) format, you will delete the **old password hash** and the user will be considered `migrated`.

### The workflow for a single user
-  The user was migrated from platform A to Supabase, `old_password_hash` as stored as 'XXXX-XXXX-XXXX'
-  After some time, the user tries to log into your Supabase application
-  Your application login screen gathers the email address and password from the user
-  Your middleware function checks to see if the user is `migrated` (meaning, the password has been migrated)
  -  If YES: the user is passed on to the normal Supabase [signIn()](https://supabase.com/docs/reference/javascript/auth-signin) flow
  -  If NO: then your middleware function runs the old password hashing algorithm against the password entered by the user to see if it matches the `old_password_hash`
    - If it DOES NOT MATCH: the login is rejected
    - If it MATCHES: 
      - The current password entered by the user is written to Supabase (GoTrue) [See update()](https://supabase.com/docs/reference/javascript/auth-update#update-password-for-authenticated-user)
      - The **old password hash** field is removed from the user, thus indicated that this user's password is now `migrated`
