# Octokit Handbook

Octokit is a set of official GitHub libraries written in different languages in order simplify the usage of the GitHub platform.

This handbook aims to help maintainers to create and maintain Octokit libraries as well as community-maintained equivalents that are not an official Octokit library (yet).

<!-- toc -->

- [Authentication](#authentication)
  - [Personal Access Tokens](#personal-access-tokens)
  - [User access tokens (OAuth)](#user-access-tokens-oauth)
  - [Installation access tokens (GitHub Apps)](#installation-access-tokens-github-apps)
  - [OAuth App authentication (`client_id`/`client_secret`)](#oauth-app-authentication-client_idclient_secret)
  - [GitHub App authentication (JSON Web Token)](#github-app-authentication-json-web-token)
- [Rest API](#rest-api)
  - [To consider](#to-consider)
  - [Octokit Implementations](#octokit-implementations)
  - [Gotchas](#gotchas)
    - [Inputs without namespace](#inputs-without-namespace)
    - [Some “list” endpoint response that paginate have a different response structure](#some-list-endpoint-response-that-paginate-have-a-different-response-structure)
    - [Some “list” endpoint respond with `204` or `409` if the repository is empty](#some-list-endpoint-respond-with-204-or-409-if-the-repository-is-empty)
    - [Same route, different results](#same-route-different-results)
    - [Supporting GitHub Enterprise](#supporting-github-enterprise)
    - [Preview headers](#preview-headers)
    - [`/repositories/:id` and `/users/:id` are currently not documented / not part of the OpenAPI specification](#repositoriesid-and-usersid-are-currently-not-documented--not-part-of-the-openapi-specification)
    - [URL parameters should be encoded](#url-parameters-should-be-encoded)
    - [Rate & Abuse limits](#rate--abuse-limits)
    - [200 Response to HEAD requests](#200-response-to-head-requests)
- [GraphQL](#graphql)
  - [Implementations:](#implementations)
  - [Gotchas](#gotchas-1)
    - [Pagination](#pagination)
    - [Preview headers](#preview-headers-1)
    - [Differences between api.github.com and GHE](#differences-between-apigithubcom-and-ghe)
- [OAuth Apps](#oauth-apps)
  - [Implementations](#implementations)
  - [GitHub’s API endpoints for OAuth App clients](#githubs-api-endpoints-for-oauth-app-clients)
  - [Security considerations](#security-considerations)
  - [Resources](#resources)
- [Webhooks](#webhooks)
  - [Implementations](#implementations-1)
  - [Gotchas](#gotchas-2)
- [GitHub Apps](#github-apps)
  - [Implementations](#implementations-2)
  - [Gotchas](#gotchas-3)
    - [Replication lag after creating installation token](#replication-lag-after-creating-installation-token)
    - [Scopes are not supported](#scopes-are-not-supported)
    - [Permissions & repositories for OAuth tokens cannot be limited](#permissions--repositories-for-oauth-tokens-cannot-be-limited)
    - [Differences between api.github.com and GHE](#differences-between-apigithubcom-and-ghe-1)
- [Actions](#actions)
- [Shared resources](#shared-resources)
  - [OpenAPI](#openapi)
  - [Webhooks](#webhooks-1)
  - [Fixtures](#fixtures)
  - [GraphQL Schema](#graphql-schema)
  - [GitHub App Permissions](#github-app-permissions)
- [License](#license)

<!-- tocstop -->

# Authentication

There are different means of authentication.

See https://github.com/octokit/auth.js for a reference implementation for the most common strategies.

## Personal Access Tokens

Personal access tokens are passed in the `Authorization` header. Example: `Authorization: token <your token here>`.

Personal access tokens cannot be created programmatically. They work the same as user access tokens created by OAuth apps. They have a set of scopes and can optionally be set to expire (the current default is 30 days from token creation).

Personal access tokens are meant for testing. You can create one at https://github.com/settings/tokens/new.

## User access tokens (OAuth)

User access tokens can be created by GitHub Apps (they are called [user-to-server tokens](https://docs.github.com/en/developers/apps/identifying-and-authorizing-users-for-github-apps)) or by OAuth Apps (they are just called [access tokens](https://docs.github.com/en/developers/apps/authorizing-oauth-apps)). There are several differences betweet the user access tokens created by GitHub Apps vs OAuth Apps.

1. **Access**

   OAuth Apps: user access tokens have access to all of the user's repositories. Access to organizations [can be restricted](https://docs.github.com/en/github/setting-up-and-managing-organizations-and-teams/restricting-access-to-your-organizations-data). Access to organizations with restrictions enabled can be granted if the user is an owner, or [requested](https://docs.github.com/en/github/setting-up-and-managing-organizations-and-teams/approving-oauth-apps-for-your-organization) if the user is a member.

   GitHub Apps: user access tokens inherit the user permissions of the app (see below) and can only access repositories and organizations that the app is installed on. Uninstalling an app or suspending an installation immediately revokes access to the respective repository/organization resources.

1. **Scopes vs Permissions.**

   OAuth Apps have the concept of [scopes](https://docs.github.com/en/developers/apps/scopes-for-oauth-apps#available-scopes). The app itself does not have a global set of scopes, each new user token can be granted any combination of scopes.

   GitHub Apps have the concept of [permissions](https://docs.github.com/en/rest/reference/permissions-required-for-github-apps). The app registration includes permissions for repositories, organizations, and users. User permissions (e.g. `starring`, `blocking`) are inherited directly from the app's registration settings. The permissions for repositories (e.g. `issues`, `pull_requests`) and organizations (e.g. `administration`, `team_discussions`) are inherited from each installation. Installation permissions are not automatically updated, each time a GitHub App requests additional repository/organization permissions, they have to be approved for each installation.

1. **Web flow**

   The OAuth web flow for [OAuth Apps](https://docs.github.com/en/developers/apps/authorizing-oauth-apps#web-application-flow) and [GitHub Apps](https://docs.github.com/en/developers/apps/identifying-and-authorizing-users-for-github-apps#web-application-flow) are nearly identical. OAuth apps accept a `?scope` query parameter to request any set of scopes.

1. **Device flow**

   The device flow works the same for OAuth Apps and GitHub Apps.

1. **Expiring tokens**

   [Expiring user access tokens](https://docs.github.com/en/developers/apps/refreshing-user-to-server-access-tokens) are only supported by GitHub Apps and are opt-in by the app owners.

## Installation access tokens (GitHub Apps)

Installation access tokens can be created by a GitHub App in order to access repository and organization resources for an installation. Installation access tokens expire after 1 hour. Using an installation access token, a GitHub App can interact on repositories or organizations as itself (e.g. create comments).

**Gotchas**

- Installation tokens cannot be used for authenticated git operations the same way that OAuth/Personal Access Tokens can be. E.g. this git URL will not work `https://${GITHUB_TOKEN}@github.com/${GITHUB_REPOSITORY}.git`. As a workaround, this scheme works with installation tokens / GitHub Action tokens:
  `https://x-access-token:{GITHUB_TOKEN}@github.com/${GITHUB_REPOSITORY}.git`
- The [OAuth device flow](https://docs.github.com/en/developers/apps/authorizing-oauth-apps#device-flow) does not require the `client_secret` so it can be safely used on clients without the need of maintaining a custom server. However, the flow is not possible for browsers due to CORS restrictions.

## OAuth App authentication (`client_id`/`client_secret`)

OAuth apps can authenticate using its `client_id` and `client_secret` in order to avoid the low rate limiting for unauthenticated requests. `client_id` and `client_secret` have to be passed as Basic authorization in the `Authorization` header.

**Gotchas**

- OAuth App authentication does not work for GraphQL queries
- The API endpoint to exchange a `code` for an OAuth access token is [`POST http(s)://[hostname]/login/oauth/access_token`](https://docs.github.com/en/developers/apps/authorizing-oauth-apps#2-users-are-redirected-back-to-your-site-by-github). It does not use the standard `api.github.com` host or `/v3/api` path for GitHub enterprise and is not documented as part of the REST API. The `Accept` & `Content-Type` headers must be set to `application/json`. The server does not respond with an `4xx` error code, even if the request failed. You have to check for the presence of the `"error"` response key instead.

## GitHub App authentication (JSON Web Token)

A JWT is passed in the `Authorization` header. Example: `Authorization: Bearer <your JWT here>`. In order to generate a JWT, a GitHub Apps private key is required. [See Authenticating as a GitHub App](https://developer.github.com/apps/building-github-apps/authenticating-with-github-apps/#authenticating-as-a-github-app).

Only a few REST API endpoints require JWT authentication

1. All routes starting with `/app`
2. `GET /orgs/:org/installation`
3. `GET /repos/:owner/:repo/installation`
4. `GET /users/:username/installation`

For all other routes and for GraphQL queries, an installation access token or user access token needs to be created.

An installation access token can be retrieved using the [`POST /app/installations/:installation_id/access_tokens`](https://docs.github.com/en/rest/reference/apps#create-an-installation-access-token-for-an-app) endpoint. Usually the `:installation_id` is retrieved from a webhook event payload or from [`GET /app/installations`](https://developer.github.com/v3/apps/#list-installations). Installation tokens can only access repositories that the app was installed on, and expire after 1h.

GitHub Apps can also create OAuth tokens for users, every GitHub App also has `client_id` and `client_secret` just like OAuth Apps. However, these cannot be used like [OAuth App authentication](#oauth-app-authentication), GitHub Apps can only use JWT in order to authenticate as itself.

**Gotchas**

- JWT authentication does not work for GraphQL queries
- When using an Installation Access Token directly after creation, a request might error with `401`. Not because the token is invalid, but because the token did not yet propagate across all read-only databases. See https://github.com/octokit/auth-app.js/issues/65#issuecomment-629384898 for more details
- When receiving one of the following errors

  > 'Expiration time' claim ('exp') must be a numeric value representing the future time at which the assertion expires

  or

  > 'Issued at' claim ('iat') must be an Integer representing the time that the assertion was issued

  The most likely problem is that the system time is out of sync with GitHub’s APIs time.

  In order to mitigate the problem, read out the `date` header from the error response, use it to calculate the approximate difference between the system time and GitHub’s API time, then calculate the JWT again and retry the request.

# Rest API

- **complete**: create methods for all endpoints, or at least document all if the Octokit library is using a generic request library. See OpenAPI in [shared resources](#shared-resources)
- **generated**: in order to implement and document the complete set of REST API route endpoints, and to (automatically) keep it up-to-date with new endpoints, the codebase should be generated. See OpenAPI in [shared resources](#shared-resources)

## To consider

- Set media type format per request, but without overriding current accept header, as it might include previews (GHE version 3.2 and lower)
- Implement `Sunset` / `Deprecation` headers (_to be documented_)

## Octokit Implementations

- https://github.com/octokit/octokit.net
- https://github.com/octokit/octokit.rb
- https://github.com/octokit/rest.js

Community implementations

- https://github.com/google/go-github

## Gotchas

### Inputs without namespace

- https://docs.github.com/en/rest/reference/markdown#render-a-markdown-document-in-raw-mode
- https://docs.github.com/en/rest/reference/repos#upload-a-release-asset

### Some “list” endpoint response that paginate have a different response structure

They have a `total_count` key in the response (search also has `incomplete_results`, `/installation/repositories` also has `repository_selection`), as well as a key with the list of the items which name varies from endpoint to endpoint

- https://developer.github.com/v3/search/ (all endpoints, key `items`)
- https://developer.github.com/v3/checks/runs/#list-check-runs-for-a-specific-ref (key: `check_runs`)
- https://developer.github.com/v3/checks/runs/#list-check-runs-in-a-check-suite (key: `check_runs`)
- https://developer.github.com/v3/checks/suites/#list-check-suites-for-a-specific-ref (key: `check_suites`)
- https://developer.github.com/v3/apps/installations/#list-repositories (key: `repositories`)
- https://developer.github.com/v3/apps/installations/#list-repositories-accessible-to-the-user-for-an-installation (key: `repositories`)
- https://developer.github.com/v3/actions/artifacts/#list-workflow-run-artifacts
  `GET /repos/:owner/:repo/actions/runs/:run_id/artifacts`, key: `artifacts`
- https://developer.github.com/v3/actions/secrets/#list-secrets-for-a-repository
  `GET /repos/:owner/:repo/actions/secrets`, key: `secrets`
- https://developer.github.com/v3/actions/workflows/#list-repository-workflows
  `GET /repos/:owner/:repo/actions/workflows`, key: `workflows`
- https://developer.github.com/v3/actions/workflow_jobs/#list-jobs-for-a-workflow-run
  `GET /repos/:owner/:repo/actions/runs/:run_id/jobs`, key: `jobs`
- https://developer.github.com/v3/actions/workflow_runs/#list-workflow-runs
  `GET /repos/:owner/:repo/actions/workflows/:workflow_id/runs`, key: `workflow_runs`
- https://developer.github.com/v3/actions/workflow_runs/#list-repository-workflow-runs
  `GET /repos/:owner/:repo/actions/runs`, key: `workflow_runs`

An Octokit library should normalize these responses so that paginated results are always returned following the same structure.

**IMPORTANT**: **If the list response has only one page, no `Link` header is provided**. Checking for the `Link` header alone is not sufficient to check whether a response is paginated or not. For the exceptions with the namespace, a fallback check for the route paths has to be added in order to normalize the response. You can check for the `total_count` property to be present, but must make sure it’s not [Get the combined status for a specific ref](https://docs.github.com/en/rest/reference/repos#get-the-combined-status-for-a-specific-reference) request, because it includes `total_count`, too, while not being a paginating response.

### Some “list” endpoint respond with `204` or `409` if the repository is empty

See [octokit/plugin-paginate-rest.js#158](https://github.com/octokit/plugin-paginate-rest.js/issues/158).

For example, if a repository is empty:

- `GET /repos/{owner}/{repo}/contributors` responds without a response body and a `204` status code
- `GET /repos/{owner}/{repo}/commits` responds without a `409` error

  ```
  {
    "message": "Git Repository is empty.",
    "documentation_url": "https://docs.github.com/rest/reference/repos#list-commits"
  }
  ```

A pagination method should return an empty array in both cases.

### Same route, different results

[Get contents](https://developer.github.com/v3/repos/contents/#get-contents)

    GET /repos/:owner/:repo/contents/:path

If `:path` points to a folder, the response is an array of files. Otherwise the response is an object.

### Supporting GitHub Enterprise

Octokit should support all GitHub Enterprise versions that are currently supported by GitHub. Besides newly introduced endpoints that are not backported to GitHub Enterprise, there are also some endpoints that are different, e.g. while endpoints on api.github.com just work, on some GHE versions (3.2 and below) they might still need preview headers.

Note that the base URLs are more complicated. For https://api.github.com, the root URL for all REST APIs is `https://api.github.com`, for GHE the root URL is `http(s)://[hostname]/api/v3`, except the [Management Console endpoints](https://docs.github.com/en/enterprise-server@3.0/rest/reference/enterprise-admin#endpoint-urls), which root URL is just `http(s)://hostname/`.

While not directly related to the REST API, it’s worth noting that for GraphQL, the endpoint URL for `api.github.com` is `https://api.github.com/graphql`, while for GHE it’s `http(s)://[hostname]/api/graphql`.

### Preview headers

The concept of _preview_ headers has been deprecated from api.github.com but it still exists in GHE version 3.2 and below. Instead of using _preview_ headers going forward, new features are now being tested using beta previews that users have to opt-in.

When _preview_ headers are set implicitly, a warning should be logged to make the user aware that these APIs are subject to change.

### `/repositories/:id` and `/users/:id` are currently not documented / not part of the OpenAPI specification

Repositories and usernames can change. For integrators, it would be more appropriate to use the repository ID as it does not change, but these endpoints are currently not documented, hence Octokit is currently not implementing them explicitly. It might be mentioned in the Octokit library’s documentation though.

### URL parameters should be encoded

For example: [`GET /repos/{owner}/{repo}/compare/{base}...{head}`](https://docs.github.com/en/rest/reference/repos#compare-two-commits). The `base` and `head` parameters are branch names, which can include `/` and other special characters.

### Rate & Abuse limits

See https://docs.github.com/en/rest/guides/best-practices-for-integrators#dealing-with-rate-limits and https://docs.github.com/en/rest/guides/best-practices-for-integrators#dealing-with-abuse-rate-limits

### 200 Response to HEAD requests

Some of GitHub's REST API endpoints respond with a `200` status to `HEAD` requests. See https://github.com/octokit/rest.js/pull/842. Arguably the response should be `204`, and it is in some cases, but not consistently.

# GraphQL

## Implementations:

- https://github.com/octokit/graphql.js

## Gotchas

### Pagination

_TBD_, see https://github.com/octokit/graphql.js/issues/61#issuecomment-643392492 for an idea how a generic pagination API could be implemented for GraphQL

### Preview headers

The concept of _preview_ headers is in the process of being deprecated.

### Differences between api.github.com and GHE

_TBD_

# OAuth Apps

If a 3rd party integration (“client”) needs to access protected resources of a GitHub user (“resource owner”), a GitHub OAuth App needs to be registered for the client. Once registered, a client can request a GitHub user to grant authorization to selected protected resources (“scope”) to the client. Once an authorization was granted, GitHub can issue OAuth access tokens to the client in order to send requests on behalf of the user.

<div style="overflow: hidden">

<a href="assets/oauth-request-access.png"><img width=300 align=right src="assets/oauth-request-access.png"></a>

In order to initiate a grant authorization request, the client redirects a user to GitHub’s [authorization endpoint](https://tools.ietf.org/html/rfc6749#section-3.1) on github.com, see [Request a user’s identity](https://developer.github.com/apps/building-oauth-apps/authorizing-oauth-apps/#1-request-a-users-github-identity).

The first time the user opens the page, all scopes requested by the client are displayed for the user to review. If the user grants the authorization request, they get redirected to the OAuth App’s redirect URL, with an additional `?code=` query parameter set to the grant’s [authorization code](https://tools.ietf.org/html/rfc6749#section-1.3.1). GitHub does not support [implicit grants](https://tools.ietf.org/html/rfc6749#section-4.2).

</div>
<div style="overflow: hidden">

<a href="assets/oauth-code-redirect.png"><img width=300 align=right src="assets/oauth-code-redirect.png"></a>

The client then exchanges the authorization code and the OAuth App’s `client_id` and `client_secret` for an OAuth access token using GitHub’s [token endpoint](https://tools.ietf.org/html/rfc6749#section-3.2). Note that GitHub’s OAuth Access tokens do not expire and there are no refresh tokens as described in the [OAuth 2.0 spec](https://tools.ietf.org/html/rfc6749#section-6).

</div>

## Implementations

- Authentication strategy for OAuth App: https://github.com/octokit/auth-oauth-app.js
- Authentication strategy for OAuth User: https://github.com/octokit/auth-oauth-user.js
- Authentication strategy for device flow: https://github.com/octokit/auth-oauth-device.js
- SDK: https://github.com/octokit/oauth-app.js
- Others: https://github.com/octokit/oauth-authorization-url.js

## GitHub’s API endpoints for OAuth App clients

1. **Authorization endpoint** ([OAuth 2.0 spec](https://tools.ietf.org/html/rfc6749#section-3.1)):
   [https://github.com/login/oauth/authorize](https://docs.github.com/en/developers/apps/authorizing-oauth-apps#1-request-a-users-github-identity)
2. **Redirect endpoint:** client URL, configured during the OAuth App registration. It can be changed in the OAuth App’s settings page on github.com
3. **Token endpoint** ([OAuth 2.0 spec](https://tools.ietf.org/html/rfc6749#section-3.2)):
   [`POST https://github.com/login/oauth/access_token`](https://docs.github.com/en/developers/apps/authorizing-oauth-apps#2-users-are-redirected-back-to-your-site-by-github)

Besides the endpoints required by the OAuth 2.0 specification, GitHub provides additional endpoints

1. Check if a token is still valid (think: check session)
   [`POST /applications/:client_id/token`](https://docs.github.com/en/rest/reference/apps#check-a-token)
2. Reset a single token(think: I’ve accidentally posted my token on Twitter for everyone to see, and I need to invalidate it and get a new one)
   [`PATCH /applications/:client_id/token`](https://docs.github.com/en/rest/reference/apps#reset-a-token)
3. Invalidate (revoke) a single token (think: sign out)
   [`DELETE /applications/:client_id/token`](https://docs.github.com/en/rest/reference/apps#delete-an-app-token)
4. Invalidate (revoke) all tokens for a single user (think: uninstall the app)
   [`DELETE /applications/:client_id/grant`](https://docs.github.com/en/rest/reference/apps#delete-an-app-authorization)

There are no endpoints for OAuth App clients to:

1. Retrieve its access tokens.
2. Retrieve its authorizations.
3. Revoke all authorizations at once.

The client owner(s) can revoke all authorizations at once on the OAuth App’s settings page. If the client requires to retrieve previously created access tokens it needs to persist them.

## Security considerations

- The `client_secret` must not be shared with user-accessible parts of the OAuth client, such as browser-based application clients or native applications, in order to prevent counterfeit clients ([compare Section 10.1 of OAuth 2.0 spec](https://tools.ietf.org/html/rfc6749#section-10.1)). If an attacker would manage to intercept the redirect after authorization was granted, they could use the known `client_id` and `client_secret` to generate a token themselves. GitHub does not support the [PKCE extension](https://tools.ietf.org/html/rfc7636).
- The client should request access tokens with the minimal scope necessary.
- Avoid passing access tokens as part of URLs. Browser history or request logs can expose tokens unknowingly. The client’s configured `redirect_url` must point to one of two things:
  1. The client back-end, which can directly retrieve the OAuth Access token and use it to authenticate as the user. The Authorization server can persist the token if future requests authenticated as the user will be necessary for the OAuth app.
  2. The client front-end, which then needs to send the authorization code to the client back-end in exchange for the OAuth Access token. The client back-end needs to expose a custom API endpoint for that operation. The client front-end can persist the OAuth Access token for future authentication against GitHub’s REST or GraphQL API.

See also: https://tools.ietf.org/html/rfc6749#section-10

## Resources

- [The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749)
  Applied to GitHub.com, the [OAuth 2.0 roles](https://tools.ietf.org/html/rfc6749#section-1.1) are:
  - **resource owner**: GitHub User
  - **resource server**: api.github.com
  - **authorization server**: github.com
  - **client**: the 3rd party integration that needs to send requests on behalf of the user. A server component that has access to both `client_id` and `client_secret` is required, in order to exchange an authorization grant for an access token.
- [Authorizating OAuth Apps on GitHub](https://docs.github.com/en/developers/apps/authorizing-oauth-apps#web-application-flow)

# Webhooks

## Implementations

- https://github.com/octokit/webhooks.js

## Gotchas

**Differences between api.github.com and GHE**

_TBD_

# GitHub Apps

## Implementations

- Authentication strategy: https://github.com/octokit/auth-app.js/
- SDK: https://github.com/octokit/app.js/

## Gotchas

### Replication lag after creating installation token

It is possible that when sending requests with an installation token that was created just before, the server might respond with a 404 (Not Found for GET/HEAD) or 403 (Bad Credentials), not because the installation token is invalid but because there is a short replication lag on GitHub’s site.

### Scopes are not supported

The `?scope` query parameter is not supported for the OAuth web flow. The OAuth tokens created by GitHub apps are constrained by the permission accepted by each installation.

### Permissions & repositories for OAuth tokens cannot be limited

When [`creating an installation access token`](https://docs.github.com/en/rest/reference/apps#create-an-installation-access-token-for-an-app), two parameters can be passed to limit the access for the token:

1. `permissions` - a subset of the installation's permissions
2. `repositories` or `repository_ids` - a subset of the installation's repositories.

There are no possibility to reduce permissions/repository access for user-to-server (OAuth) access tokens as of February 2021

### Differences between api.github.com and GHE

_TBD_

# Actions

Actions are run in Docker containers, which have access to the current source code as well as environment variables. The two most relevant are

1. `GITHUB_TOKEN`: An authentication token used to make API requests to GitHub
2. `GITHUB_EVENT_PATH`: Path to a JSON file containing the event payload that triggered the action.

See [GitHub Actions Environment variables](https://docs.github.com/en/actions/reference/environment-variables) for more.

The Octokit library reads out the token, instantiates an authenticated API to utilize the REST/GraphQL API and provides the event name and payload as a parameter, similar to the handling of Webhooks.

**Implementations**

- SDK: https://github.com/octokit/action.js
- toolkit: https://github.com/actions/toolkit

**Gotchas**

- The `GITHUB_TOKEN` cannot be used as expected for an authenticated git remote URL: `https://${GITHUB_TOKEN}@github.com/${GITHUB_REPOSITORY}.git`,
  instead it needs to be [used like an installation access token](https://docs.github.com/en/developers/apps/authenticating-with-github-apps#http-based-git-access-by-an-installation):
  `https://x-access-token:{GITHUB_TOKEN}@github.com/${GITHUB_REPOSITORY}.git`

# Shared resources

## OpenAPI

https://github.com/octokit/openapi

JSON API specs for all REST API endpoints for `api.github.com` as well as all currently supported GitHub Enterprise versions. [https://github.com/octokit/openapi](https://github.com/octokit/openapi) pulls changes from [GitHub's official OpenAPI spec](https://github.com/github/rest-api-description/) and adds a `x-octokit` extension with relevant information for Octokit libraries, such as a list of changes for each endpoint, so that OpenAPI operation ID changes don't need to result in breaking changes, but instead deprecations can be implemented.

In order to act on new releases, install the [openapi-release-notifier](https://github.com/apps/openapi-release-notifier) GitHub App. See the app’s description for further information and an example on how to trigger a GitHub Action Workflow each time a new OpenAPI spec version is released.

## Webhooks

https://github.com/octokit/webhooks

Always up-to-date, machine-readable schemas for all GitHub Events

## Fixtures

https://github.com/octokit/fixtures, https://github.com/octokit/fixtures-server

Always up-to-date fixtures for REST API endpoints that can be used for tests so they don’t depend on api.github.com or GitHub Enterprise Server installations

A mock-server binary can be downloaded from https://github.com/octokit/fixtures-server/releases/latest. It is also continuously deployed. See https://github.com/octokit/fixtures-server#usage

Fixtures are updated using a nightly cron job. All requests & responses are [normalized](https://github.com/octokit/fixtures/blob/master/HOW_IT_WORKS.md#normalizations) in order to detect changes. When a change occurs, a pull request is opened for review, see https://github.com/octokit/fixtures/pull/177. When merged, a new release is published automatically based on commit message conventions, see https://github.com/octokit/fixtures/blob/master/CONTRIBUTING.md#releases.

New fixtures can be added by creating a new scenario, see https://github.com/octokit/fixtures/blob/master/HOW_IT_WORKS.md.

## GraphQL Schema

https://github.com/octokit/graphql-schema

Always up-to-date GraphQL schema, that can be used to generate types for GraphQL APIs

## GitHub App Permissions

https://github.com/octokit/app-permissions/

Machine-readable, always up-to-date GitHub App permissions

# License

[CC BY 4.0](LICENSE.md)
