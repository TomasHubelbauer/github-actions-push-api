# GitHub Actions Push API

[
  ![creation](https://github.com/TomasHubelbauer/github-actions-push-api/actions/workflows/creation.yml/badge.svg)
](https://github.com/TomasHubelbauer/github-actions-push-api/actions/workflows/creation.yml)
[
  ![modification](https://github.com/TomasHubelbauer/github-actions-push-api/actions/workflows/modification.yml/badge.svg)
](https://github.com/TomasHubelbauer/github-actions-push-api/actions/workflows/modification.yml)

## Preface

I have come up with a way to push changes from a GitHub Actions workflow back to
the repository already here:

https://github.com/tomasHubelbauer/github-actions#write-workflow

It relies on setting up one's own Git identity and then pushing from the GitHub
Actions workflow to the repository using Git. The integration PAT token is used
for authentication, which lets GitHub know the commit was automated and that it
should not run the workflow for it.

It is also possible to hit the GitHub API with the integration PAT. I have repos
where I use it to automatically curate GitHub Issues, for example. An issue made
using the workflow PAT will have `github-actions` for its author and the GitHub
handle. The link of the handle leads to the GitHub Actions landing page, not a
normal GitHub user account page.

One downside of using one's own Git identity and pushing to GitHub using Git and
the integration PAT is that such commits count as one's contributions in the GH
contribution chart on the user profile page. One could set up any other kind of
identity for the commit as GitHub doesn't check the emails associated with the
commits, it just makes them into handles if it is an email known to GitHub. So,
in order to avoid this, a new email could be used for the automated commits so
that they are attributed to the service account / fake email. However, I don't
like this option. I do not like programmatic accounts that look like personal
accounts. GitHub doesn't have support for real service accounts, but it has one
through this `github-actions` special account it assigns the automatically made
issues to.

Side note: the `ghost` account is also a special account, but I don't consider
it to be a service account as it is just an account GitHub special-cased for a
stand-in for deleted users. It even has its own user page. The `github-actions`
one on the other hand has no user page, like was already stated, it leads to the
GitHub Actions landing page, clearly marking it as different from a normal user
account.

Could I make GitHub make a commit appear as thought it was coming from the same
service account it assigns the issues to? I can't use that account for a Git
identity as it has no email, so there is no magical email I can author a commit
with to make it appear that way. I could call the Contents API with the PAT from
the workflow and not specify `committer` and `author` as those seem optional.
Will that work? It says it will use the identity associated with the PAT used,
but if there's none since it is the integration PAT, what will happen?

## Results

I was able to use the GitHub API to make a file that appears as pushed by the
GitHub Actions service account by using the integration PAT and omitting the
`committer` and `author` fields in the request payload. These fields default to
the authenticated user which in the case of the integration PAT is the GitHub
Actions service account. See:

- [`creation.yml`](https://github.com/TomasHubelbauer/github-actions-push-api/actions/workflows/creation.yml)
- [`modification.yml`](https://github.com/TomasHubelbauer/github-actions-push-api/actions/workflows/modification.yml)

## Notes

### API Response

The API response with the `author` and `committer` fields coerced to defaults,
the authenticated user - GitHub Actions service account in the case of the
integration PAT, looks like this:

```json
"author": {
  "name": "github-actions[bot]",
  "email": "41898282+github-actions[bot]@users.noreply.github.com"
},
"committer": {
  "name": "GitHub",
  "email": "noreply@github.com"
}
```

I am not sure what the number `41898282` represents. I am guessing it might be
my GitHub user ID, or the workflow ID or something like that. It is constant
across workflow runs.

This `committer` and `author` objects appear like this even when I use the Basic
auth option and pass in my GitHub handle as the user name:
`curl -u ${{github.repository_owner}}:${{github.token}}`

The user name seems to be completely ignored and just the PAT in the password is
used.

### Authorization

The GitHub API supports Basic authentication where the password is replaced by a
PAT as well as using the `Authorization` header with a PAT. I tried both in the
workflow and both work. The Basic auth option can even have no user name set and
it still works:

`curl -u :${{github.token}}`
(or `curl -u ${{github.repository_owner}}:${{github.token}}`)

https://docs.github.com/en/rest/overview/other-authentication-methods#via-oauth-and-personal-access-tokens

`curl -H "Authorization: token ${{github.token}}"`

https://docs.github.com/en/rest/overview/other-authentication-methods#authenticating-for-saml-sso

## To-Do

### Test out modification instead of creation to have a solution to the `sha`

The GitHub Contents API requires a `sha` field when replacing an existing file.
This is not a SHA of a commit, but the file blob itself. I think the easiest way
to get it is to use the API to inspect the file:

https://docs.github.com/en/rest/reference/repos#get-repository-content

One could also add a checkout step and calculate the checked out file SHA, but
this is going to be more useful when working with single-file commits.

### See if I can create a Git identity with empty name and password and Git push

In order to not have to push files individually, could I use Git instead of the
REST API, authenticate with the GitHub using the integration PAT, make a commit
with no name or email (`git commit --author " <>"`?) and push it using Git?

Will this fail? Will this also use the GitHub Actions service account?

### See if I can use the email returned from the GitHub API for a Git identity

See the [API Response](#api-response).

I wonder if I could set up a Git identity with my name or handle for the name
and this `41898282+github-actions[bot]@users.noreply.github.com` email and use
it to push to the repository and whether if I do that, the service account of
GitHub Actions will get associated with the commit like when I commit through
the API. This would be useful for committing using Git and making multiple file
changes in the commits instead of using the API which would be clunky for this.
