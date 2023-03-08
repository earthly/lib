# Billing utilities
## +active-users
Estimates monthly active users of a GIT remote repo, based on past commits

Usage: 
```
earthly github.com/earthly/lib/billing+active-users --repoUrl
```

### Private repositories
For private repositories use https URLs, and pass credentials within the URL:
```
earthly github.com/earthly/lib/billing+active-users https://<user>@<password>/host/repo.git
```
> Note:
> For GitHub private repos you need to use a [personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) instead of your password 
