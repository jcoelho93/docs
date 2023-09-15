# Organizations

A collection of common tasks with organizations using the GraphQL API.

You can test out the Buildkite GraphQL API using the [Buildkite explorer](https://graphql.buildkite.com/explorer). This includes built-in documentation under the _Docs_ panel.

## List organization members

List the first 100 members in the organization.

```graphql
query getOrgMembers{
  organization(slug: "organization-slug") {
    members(first: 100) {
      edges {
        node {
          role
          user {
            name
            email
            id
          }
        }
      }
    }
  }
}
```

## Search for organization members

Look up organization members using their email address.

```graphql
query getOrgMember{
  organization(slug: "organization-slug") {
    members(first: 1, search: "user-email") {
      edges {
        node {
          role
          user {
            name
            email
            id
          }
        }
      }
    }
  }
}
```

## Get the most recent SSO sign-in for all users

Use this to get the last sign-in date for users in your organization, if your organization has SSO enabled.

```graphql
query getRecentSignOn {
  organization(slug: "organization-slug") {
    members(first: 100) {
      edges {
        node {
          user {
            name
            email
          }
          sso {
            authorizations(first: 1) {
              edges {
                node {
                  createdAt
                  expiredAt
                }
              }
            }
          }
        }
      }
    }
  }
}
```

## Update the default SSO provider session duration

You can control how long the session can go before the user must revalidate with your SSO. By default that's indefinite, but you can reduce it down to hours or days.

```graphql
mutation UpdateSessionDuration {
  ssoProviderUpdate(input: { id: "ID", sessionDurationInHours: 2 }) {
    ssoProvider {
      sessionDurationInHours
    }
  }
}
```

## Update inactive API token revocation

On the Enterprise plan, you can control when inactive API tokens are revoked. By default, they are never (`NEVER`) revoked, but you can set your token revocation to either 30, 60, 90, 180, or 365 days.

```graphql
mutation UpdateRevokeInactiveTokenPeriod {
  organizationRevokeInactiveTokensAfterUpdate(input: {
    organizationId: "organization-id",
    revokeInactiveTokensAfter: DAYS_30
  }) {
    organization {
      revokeInactiveTokensAfter
    }
  }
}
```

## Pin SSO sessions to IP addresses

You can require users to re-authenticate with your SSO provider when their IP address changes with the following call, replacing `ID` with the GraphQL ID of the SSO provider:

```graphql
mutation UpdateSessionIPAddressPinning {
  ssoProviderUpdate(input: { id: "ID", pinSessionToIpAddress: true }) {
    ssoProvider {
      pinSessionToIpAddress
    }
  }
}
```

## Query the usage API

Use the usage API to query your organization's usage by pipeline or test suite at daily granularity.

```graphql
query Usage {
  organization(slug: "organization-slug") {
    id
    name
    usage(
      aggregatedOnFrom: "2023-04-01"
      aggregatedOnTo: "2023-05-01"
      resource: [JOB_MINUTES, TEST_EXECUTIONS]
    ) {
      edges {
        node {
          __typename ... on JobMinutesUsage {
            aggregatedOn
            seconds
            pipeline {
              name
              id
            }
          }
        }
        node {
          __typename ... on TestExecutionsUsage {
            Time: aggregatedOn
            executions
            suite {
              name
              id
            }
          }
        }
      }
      pageInfo {
        endCursor
        hasNextPage
      }
    }
  }
}
```

## Create a user, add them to a team, and set user permissions

Invite a new user to the organization, add them to a team, and set their role.

First, get the organization and team ID:

```graphql
query getOrganizationAndTeamId {
  organization(slug: "organization-slug") {
    id
    teams(first:500) {
      edges {
        node {
          id
          slug
        }
      }
    }
  }
}
```

Then invite the user and add them to a team, setting their role to 'maintainer':

```graphql
mutation CreateUser {
  organizationInvitationCreate(input: {
    organizationID: "organization-id",
    emails: ["user-email"],
    role: MEMBER,
    teams: [
      {
        id: "team-id",
        role: MAINTAINER
      }
    ]
  }) {
    invitationEdges {
      node {
        email
        createdAt
      }
    }
  }
}
```



## Delete an organization member

This deletes a member from an organization. It does not delete their Buildkite user account.

First, find the member's ID:

```graphql
query getOrganizationMemberIds {
  organization(slug: "organization-slug") {
    members(search: "organization-member-name", first: 10) {
      edges {
        node {
          role
          user {
            name
          }
          id
        }
      }
    }
  }
}
```

Then, use the ID to delete the user:

```graphql
mutation deleteOrgMember {
  organizationMemberDelete(input: { id: "organization-member-id" }){
    organization{
      name
    }
    deletedOrganizationMemberID
    user{
        name
    }
  }
}
```

## Get organization audit events

Query your organization's audit events. Audit events are only available to Enterprise customers.

```graphql
query getOrganizationAuditEvents{
  organization(slug:"organization-slug"){
    auditEvents(first: 500){
      edges{
        node{
          type
          occurredAt
          actor{
            name
          }
          subject{
            name
            type
          }
        }
      }
    }
  }
}
```

To get all audit events in a given period, use the `occurredAtFrom` and `occurredAtTo` filters like in the following query:

```graphql
query getTimeScopedOrganizationAuditEvents{
  organization(slug:"organization-slug"){
    auditEvents(first: 500, occurredAtFrom: "2023-01-01T12:00:00.000", occurredAtTo: "2023-01-01T13:00:00.000"){
      edges{
        node{
          type
          occurredAt
          actor{
            name
          }
          subject{
            name
            type
          }
        }
      }
    }
  }
}
```

## Create & delete system banners

To create a system banner vie that GraphQL API  you can use the `organizationBannerCreate`, `organizationBannerUpdate`
& `organizationBannerDelete` mutations.

To create a banner, pass in the organization GraphQL id.

```graphql
mutation createBanner {
  organizationBannerCreate(
    input: {
      organizationId: "organization-id"
      message: "**Deploy Freeze**: Please do not deploy code over the holiday period. November 24th => November 28th"
    }
  )
  {
    banner {
      id
      message
    }
  }
}
```

To update the banner, call `organizationBannerUpdate` with the banner's GraphQL id.

```graphql
mutation updateBanner {
  organizationBannerUpdate(
    input: {
      id: "banner-id"
      message: "Updated deploy documentation available [here](https://example.com)"
    }
  )
  {
    banner {
      id
      message
    }
  }
}
```

To remove the banner call `organizationBannerDelete` with the banner's GraphQL id.

```graphql
mutation deleteBanner {
  organizationBannerDelete(
    input: {
      id: "banner-id"
    }
  )
  {
    deletedBannerId
  }
}
```
