# Billing utilities
## +active-users
Estimates monthly active users of a GIT remote repo, based on past commits

Usage: 
```
earthly --secret repoUrl=<repoUrl> github.com/earthly/lib/billing+active-users
```

### Private repositories
#### SSH
For SSH URLs, SSH auth is passed-though from the host machine.

#### HTTPS
For HTTPS URLs, pass credentials within the URL:
```
earthly --secret repoUrl=https://<user>@<password>/host/repo.git github.com/earthly/lib/billing+active-users 
```
> Note:
> For GitHub private repos you need to use a [personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) instead of your password 
