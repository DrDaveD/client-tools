# Client Tools for WLCG Tokens: Technical Investigation 
_Authored by the WLCG AuthZ Working Group_

**Pending v0.1 27.07.2020**

This document is to record requirements and discussions regarding Client and Command Line Tools for token based WLCG use. 
It will be used to help the WLCG Authorization Working Group decide on the future path for client tools and define any necessary enhancement work.

Any decisions or recommendations described here refer strictly to command line client tools. For example, a recommendation against 
Public Clients does not preclude their use in other suitable situations (such as Single-Page-Applications).

## Introduction
In the X.509 based WLCG infrastructure, users were able to install a Grid User Certificate (valid for one year) and generate proxies on demand using a tool called ``voms-proxy-init`` 
that authenticated the user, checked authorisations against VOMS and produced an X.509 proxy with authorisation extensions. In the new OAuth based WLCG Infrastructure there is no direct 
equivalent for such a flow. Access Tokens (which grant access to OAuth protected WLCG Resources) may be held directly by Users, but are only valid for 20 minutes. Clients (which typically are secure services themselves) can store Refresh 
Tokens on behalf of Users and use them to request additional Access Tokens when required. Consequently, there must be a Client interacting with the Users to provision Access 
Tokens on demand. A further challenge is providing Users with initial Access Tokens required for many flows; unless Users are recorded e.g. in an LDAP service trusted by the Authorization Server, a round trip to a browser 
is required to authenticate the User. An acceptable balance must be found in terms of User Friendliness in the frequency of these browser flows. This document pulls together
the WLCG Authorization Working Group's requirements for a command line tool that provisions Users with Access Tokens, and identifies relevant tools.

## Glossary/Terminology

| OAuth Term | Definition | Closest existing WLCG Analog |
| --- | --- | --- |
| Client | An application making protected resource requests on behalf of the user and with its authorization. The term “client” does not imply any particular implementation characteristics (e.g., whether the application executes on a server, a desktop, or other devices). | |
| Access token | From RFC6749 "Access tokens are credentials used to access protected resources.  An access token is a string representing an authorization issued to the client.  The string is usually opaque to the client.  Tokens represent specific scopes and durations of access, granted by the resource owner, and enforced by the resource server and authorization server."| VOMS proxy credential |
| Refresh token | From RFC6749 "Refresh tokens are credentials used to obtain access tokens.  Refresh tokens are issued to the client by the authorization server and are used to obtain a new access token when the current access token becomes invalid or expires, or to obtain additional access tokens with identical or narrower scope (access tokens may have a shorter lifetime and fewer permissions than authorized by the resource owner)."| Long lived proxy stored in MyProxy (or an EEC) | 
| Public Client | From RFC6749 "Clients incapable of maintaining the confidentiality of their credentials (e.g., clients executing on the device used by the resource owner, such as an installed native application or a web browser-based application), and incapable of secure client authentication via any other means." | voms-proxy-init (i.e. no explicit registration required, user credentials are sufficient); if run on a public UI like lxplus.cern.ch, the confidentiality of the credentials cannot be _guaranteed_ |
| Confidential Client | From RFC6749 "Confidential clients are typically issued (or establish) a set of client credentials used for authenticating with the authorization server (e.g., password, public/private key pair)." | myproxy-logon |
| Dynamic client Registration | From RFC791, this is a mechanism for registering OAuth clients on demand, rather than static (manual) configuration in the OP. “Registration requests send a set of desired client metadata values to the authorization server.  The resulting registration responses return a client identifier to use at the authorization server and the client metadata values registered for the client.  The client can then use this registration information to communicate with the authorization server using the OAuth 2.0 protocol.” Dynamic Client Registration may optionally be initiated using an Initial Access Token.| Manual intervention to register a trusted client DN as an authorized proxy renewer or retriever in the configuration of a MyProxy service |

## Requirements
* Public clients: typically not acceptable since they are un-authenticated. An exception to this is helper clients that may be used to bootstrap a dynamic client registration. Refresh tokens must **not** be issued to public clients.
* Dynamic registration: acceptable, though should be limited to VO members (i.e. OAuth protected endpoint) to help prevent DoS or other attacks.
* Token Audience: tokens without an audience might pose a security risk we wish to avoid. Fine grained guidance on audience requirements will be developed as we gain experience.
* Should not rely on Kerberos authentication: requiring all users to be in a trusted LDAP service undermines Federated Identity to a certain extent. However, Kerberos could be provided as a convenience for those who have it, e.g. at big institutes like CERN and FNAL.
* Usability
    * Users should not have to remember additional passwords
    * Users should only have to authenticate every X days (current guidance is for 7 days, this will evolve as the WG gains experience)
    * Users should not have to each install a client 
    * Users should not need to install tools for each submit host

## Potential Tools

| Tool | Status | Comment |
| -- | -- | --- |
| oidc-agent | https://github.com/indigo-dc/oidc-agent | Active deployment. Does not support restricted audience. Additional password required. |
| Vault with htgettoken | https://www.vaultproject.io and  ­https://github.com/DrDaveD/htgettoken | Demo to WG on 23rd of July. Fulfils most requirements, missing support for requested scopes and audiences |
| oidc-agent with my-token | Planned | Proposed tool, enhancing oidc-agent. Would allow central store for tokens | 

### Vault with htgettoken Workflows

1. Vault server is statically registered with token issuer as an Oauth client, and client is configured on the Vault server oidc plugin in a vault role (which is typically a VO name)
1. User runs htgettoken to obtain an access token, passing the vault role name and Vault server name, which depending on circumstances runs through one of 4 different workflows:
    1. OIDC authentication workflow
        1. If no other flow below is available, htgettoken contacts Vault with an OIDC device flow request for the vault role, which interacts with token issuer and user’s web browser and returns an Oauth refresh token and Oauth subject name. Vault returns those to htgettoken along with a vault token that has privileges to write and read paths in Vault that include the subject name (and no other paths).  htgettoken stores the vault token in /tmp and stores the subject name for the role under $HOME/.config/htgettoken **<-- THAT PATH LOOKS WRONG** .
        1. htgettoken uses the vault token to store the refresh token back in Vault oauth secrets plugin at a path based on the subject name and vault role.
        1. htgettoken uses the vault token to request an access token for the subject and vault role, which Vault obtains from the token issuer using the stored refresh token.
    1. Reuse access token workflow
        1. If there is an existing unexpired access token (which are short-lived, about an hour) on the local disk, htgettoken simply verifies that it has some time remaining and does no more.
    1. Reuse vault token workflow
        1. If an access token is expired but the vault token is not (longer lived, about a week), htgettoken asks Vault for a new access token, exactly like step i.c above 
    1. Kerberos authentication workflow
        1. If there is no valid vault token but there are a stored subject name and Kerberos credentials available, htgettoken does Kerberos authentication with vault to obtain a new vault token and stores it
        1. htgettoken asks Vault for a new access token, exactly like step i.c above
1. Job management servers can be issued separate vault tokens to be able to obtain new access tokens on behalf of users  
See separate [google doc](https://docs.google.com/presentation/d/19BosYQ-OKlSwNkHe9j1Oc1-zi38gyiDcGEyu7WDRjVo/edit#slide=id.p) with detailed diagrams of flows

### Oidc-agent One-Off Workflow

1. Script is installed that acts as a public client
    1. User authenticates using Device Code Flow (browser) and gets an Access Token with the ability to register dynamic clients
    1. Script uses Access Token to register a new Dynamic Client
1. Authorization Server returns client ID and Secret
1. Script adds Client ID and Secret to oidc-agent configuration
1. Script obtains Refresh Token using client ID and Secret - client ID, Secret and Refresh Token are stored on disk and encrypted using an additional passphrase supplied by the user
1. New access tokens can be obtained using the refresh token stored in oidc-agent

_Note: the user will need to go to the web browser twice.   When oidc-agent is restarted, the user is required to supply the decryption passphrase._
