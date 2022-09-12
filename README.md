# supabase-user-password-migration
How to migrate user passwords from an external authentication system to Supabase

## Scenario
- You've migrated users from another platform to Supabase (GoTrue)
- These users were authenticated using an email + password combination in the previous platform
- You'd like to authenticate these users in your Supabase application without:
  - any additional steps on the part of the user
  - without the user even needing to know their account has been migrated (a seamless transition)
  
