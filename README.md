# GitHub Actions Push API

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

https://docs.github.com/en/rest/reference/repos#create-or-update-file-contents

## Results

I was able to use the GitHub API to make a file that appears as pushed by the
GitHub Actions service account. The response payload includes these values for
the commiteer and author fields on the commit object:

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
my GitHub user ID, or the workflow ID or something like that. It is consistent
across runs.

This `committer` and `author` objects appear like this even though when I passed
the PAT to cURL using the `-u` option, I specified my GitHub handle as the user
name (and the integration PAT as the password).

## To-Do

### See if the `Authorization` header would work better than the `-u` option

I am using `curl -u ${{github.repository_owner}}:${{github.token}}` in order to
authenticate with GitHub. This requires passing a user name which seems to be
ignored when using the integration PAT, but still, could I do it without one?

When calling the GitHub REST API from Node, I use `token ${token}` for the
`Authorization` header and it works, too. In cURL, the same would go like this:
`curl -H "Authorization: token ${{github.token}}"`.

Will this work the same way as using the `-u` option?

### See if I can create a Git identity with empty name and password and Git push

In order to not have to push files individually, could I use Git instead of the
REST API, authenticate with the GitHub using the integration PAT, make a commit
with no name or email (`git commit --author " <>"`?) and push it using Git?

Will this fail? Will this also use the GitHub Actions service account?

### See if I can use the email returned from the GitHub API for a Git identity

The API based push with the integration token returns this for the committer and
author:

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

I wonder if I could set up a Git identity with my name or handle for the name
and this `41898282+github-actions[bot]@users.noreply.github.com` email and use
it to push to the repository and whether if I do that, the service account of
GitHub Actions will get associated with the commit like when I commit through
the API. This would be useful for committing using Git and making multiple file
changes in the commits instead of using the API which would be clunky for this.
