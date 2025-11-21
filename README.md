# Jenkins Single Sign-On (SSO) with Keycloak using OpenID Connect

This repository demonstrates how to secure Jenkins with Keycloak using OpenID Connect (OIDC) protocol for Single Sign-On authentication. This setup allows you to centralize authentication management and provide a seamless login experience for Jenkins users.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration Guide](#configuration-guide)
  - [Part 1: Keycloak Setup](#part-1-keycloak-setup)
  - [Part 2: Create a New Realm](#part-2-create-a-new-realm)
  - [Part 3: Create a Client for Jenkins](#part-3-create-a-client-for-jenkins)
  - [Part 4: Create Users and Groups](#part-4-create-users-and-groups)
  - [Part 5: Create Client Scopes for Groups](#part-5-create-client-scopes-for-groups)
  - [Part 6: Assign the Client Scope to Jenkins Client](#part-6-assign-the-client-scope-to-jenkins-client)
  - [Part 7: Verify the Token](#part-7-verify-the-token)
  - [Part 8 & 9: Configure OIDC in Jenkins](#part-8--9-configure-oidc-in-jenkins)
  - [Part 10-12: Testing the SSO Flow](#part-10-12-testing-the-sso-flow)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)

---

## Overview

**What is Single Sign-On (SSO)?**

Single Sign-On is an authentication mechanism that allows users to access multiple applications with a single set of credentials. Instead of managing separate usernames and passwords for each application, users authenticate once and gain access to all connected systems.

**Why Keycloak?**

Keycloak is an open-source Identity and Access Management (IAM) solution that provides:
- Centralized user management
- Support for multiple authentication protocols (OIDC, SAML, OAuth 2.0)
- Role-based access control (RBAC)
- Social login integrations
- User federation (LDAP, Active Directory)

**Why OpenID Connect?**

OpenID Connect (OIDC) is a modern authentication protocol built on top of OAuth 2.0. It provides:
- Simple integration with modern applications
- Secure token-based authentication
- User profile information via standardized claims
- Better support for mobile and single-page applications

---

## Architecture

This setup consists of three main components working together to provide secure Single Sign-On authentication:

### Architecture Overview

![Jenkins & Keycloak Architecture](Jenkins%20&%20Keycloak.png)

The diagram above illustrates the overall architecture of the Jenkins-Keycloak SSO integration, showing how the components interact with each other.

### Detailed Authentication Flow

![Jenkins & Keycloak Authentication Flow](Jenkins%20&%20Keycloak%202.png)

This diagram provides a detailed view of the authentication flow between users, Jenkins, and Keycloak.

---

**Component Relationship:**

1. **Keycloak (Identity Provider)**: Acts as the central authentication server
   - Manages users, groups, and roles
   - Issues authentication tokens (JWT)
   - Handles login/logout flows
   - Runs on port 8080

2. **Jenkins (Service Provider/Client)**: The application being secured
   - Delegates authentication to Keycloak
   - Validates tokens received from Keycloak
   - Grants access based on user claims
   - Runs on port 8000

3. **PostgreSQL**: Database backend for Keycloak
   - Stores realm configurations, users, and sessions
   - Runs on port 5433

**Authentication Flow:**

1. User attempts to access Jenkins
2. Jenkins redirects unauthenticated users to Keycloak
3. User enters credentials on Keycloak's login page
4. Keycloak validates credentials and issues an ID token (JWT)
5. User is redirected back to Jenkins with the token
6. Jenkins validates the token and extracts user information
7. Jenkins grants access based on the user's roles/groups

---

## Prerequisites

- Docker and Docker Compose installed
- Ports 8000, 8080, and 5433 available
- Basic understanding of OAuth 2.0/OIDC concepts

---

## Quick Start

1. Clone this repository:
```bash
git clone https://github.com/ibrahimelothmani/Jenkins-Single-Sign-On-SSO-with-Keycloak-using-OpenID-Connect.git
cd Jenkins-Single-Sign-On-SSO-with-Keycloak-using-OpenID-Connect
```

2. Start all services:
```bash
docker-compose up -d
```

3. Access the services:
   - **Keycloak Admin Console**: http://localhost:8080 (admin/admin)
   - **Jenkins**: http://localhost:8000

4. Follow the [Configuration Guide](#configuration-guide) below to set up the integration.

---

## Configuration Guide

### Part 1: Keycloak Setup

After starting the Docker containers, access Keycloak at http://localhost:8080 and log in with the admin credentials (admin/admin).

![Keycloak Setup](Part%201-Keycloak%20Setup.png)

The Keycloak admin console is where you'll configure all authentication settings, including realms, clients, users, and roles.

---

### Part 2: Create a New Realm

A **realm** in Keycloak is a logical container that manages a set of users, credentials, roles, and groups. It's isolated from other realms.

**Steps:**
1. Click the dropdown in the top-left corner (currently showing "master")
2. Click "Create Realm"
3. Enter a realm name (e.g., "jenkins-realm")
4. Click "Create"

![Create a New Realm](Part%202-Create%20a%20New%20Realm.png)

> [!TIP]
> Use a descriptive realm name that reflects your organization or project. The master realm should only be used for administrative purposes.

---

### Part 3: Create a Client for Jenkins

A **client** represents an application that wants to use Keycloak for authentication. In this case, Jenkins is our client.

**Steps:**
1. Navigate to "Clients" in the left sidebar
2. Click "Create Client"
3. Configure the client:
   - **Client ID**: `jenkins` (this will be used in Jenkins configuration)
   - **Client Protocol**: `openid-connect`
   - **Client Authentication**: ON
   - **Valid Redirect URIs**: `http://localhost:8000/securityRealm/finishLogin`
   - **Web Origins**: `http://localhost:8000`

![Create a Client for Jenkins](Part%203-Create%20a%20Client%20for%20Jenkins.png)

> [!IMPORTANT]
> The **Redirect URI** must exactly match the Jenkins callback URL. Any mismatch will cause authentication to fail.

**Client Credentials:**
After creating the client, go to the "Credentials" tab and copy the **Client Secret**. You'll need this for Jenkins configuration.

---

### Part 4: Create Users and Groups

**Users** are individual accounts that can authenticate. **Groups** allow you to organize users and assign permissions collectively.

**Creating a User:**
1. Navigate to "Users" in the left sidebar
2. Click "Add User"
3. Fill in user details:
   - **Username**: The login identifier
   - **Email**: User's email address
   - **First Name** and **Last Name**
4. Click "Create"
5. Go to the "Credentials" tab and set a password

![Create Users and Groups](Part%204-Create%20Users%20and%20Groups.png)

**Adding a New User:**

![Add a New User](Part%204.1-add%20a%20new%20user.png)

**Creating Groups:**
1. Navigate to "Groups"
2. Create groups like "jenkins-admins" or "jenkins-developers"
3. Assign users to appropriate groups

> [!TIP]
> Use groups to manage Jenkins permissions efficiently. For example, create separate groups for administrators, developers, and viewers.

---

### Part 5: Create Client Scopes for Groups

**Client Scopes** define what information (claims) will be included in the authentication token sent to Jenkins.

**Steps:**
1. Navigate to "Client Scopes"
2. Click "Create Client Scope"
3. Configure:
   - **Name**: `jenkins-groups`
   - **Protocol**: `openid-connect`
   - **Include in Token Scope**: ON

![Create Client Scopes for Groups](Part%205-Create%20Client%20Scopes%20for%20Groups.png)

**Add Group Mapper:**

![Add Mapper](Part%205-Add%20mapper.png)

1. Go to the "Mappers" tab
2. Click "Add Mapper" → "By Configuration" → "Group Membership"
3. Configure:
   - **Name**: `groups`
   - **Token Claim Name**: `groups`
   - **Full group path**: OFF (unless you need nested group paths)
4. Save

This mapper ensures that user group information is included in the token sent to Jenkins.

---

### Part 6: Assign the Client Scope to Jenkins Client

Now we need to attach the client scope we created to our Jenkins client.

**Steps:**
1. Go to "Clients" → "jenkins"
2. Navigate to the "Client Scopes" tab
3. Click "Add Client Scope"
4. Select "jenkins-groups"
5. Add as "Default" scope

![Assign the Client Scope to Jenkins Client](Part%206-Assign%20the%20Client%20Scope%20to%20Jenkins%20Client.png)

This ensures that every authentication token issued for Jenkins will include group information.

---

### Part 7: Verify the Token

It's important to verify that the token contains the expected claims before configuring Jenkins.

**Steps:**
1. Go to "Clients" → "jenkins" → "Client Scopes" tab
2. Click "Evaluate"
3. Select a user to evaluate
4. View the "Generated Access Token" or "Generated ID Token"
5. Verify that the `groups` claim appears with the correct group memberships

![Verify the Token](Part%207-Verify%20the%20Token.png)

> [!NOTE]
> The token is a JWT (JSON Web Token) containing user information and group memberships. Jenkins will use this information to grant appropriate permissions.

---

### Part 8 & 9: Configure OIDC in Jenkins

Now we'll configure Jenkins to use Keycloak for authentication.

**Prerequisites:**
1. Access Jenkins at http://localhost:8000
2. Complete the initial setup wizard (if first time)
3. Install the **OpenID Connect Authentication Plugin**:
   - Go to "Manage Jenkins" → "Manage Plugins"
   - Search for "OpenID Connect Authentication"
   - Install and restart Jenkins

**Configuration Steps:**

![Configuring OIDC in Jenkins - Part 8](Part%208-Configuring%20OIDC%20in%20Jenkins.png)

1. Go to "Manage Jenkins" → "Configure Global Security"
2. Under "Security Realm", select "Login with OpenID Connect"
3. Configure the following:

   - **Client ID**: `jenkins` (must match Keycloak client ID)
   - **Client Secret**: [Paste the secret from Keycloak client credentials]
   - **Configuration Mode**: "Automatic Configuration"
   - **Well-known endpoint**: `http://localhost:8080/realms/jenkins-realm/.well-known/openid-configuration`

![Configuring OIDC in Jenkins - Part 9](Part%209-Configuring%20OIDC%20in%20Jenkins.png)

4. Advanced Settings:
   - **User name field**: `preferred_username`
   - **Full name field**: `name`
   - **Email field**: `email`
   - **Groups field**: `groups`

5. Click "Save"

> [!WARNING]
> Before saving, make sure to configure authorization properly. Otherwise, you might lock yourself out of Jenkins. It's recommended to keep "Logged-in users can do anything" enabled during initial testing.

**Authorization Configuration:**

For role-based access control:
1. Under "Authorization", select "Role-Based Strategy" or "Matrix-based security"
2. Grant permissions based on groups:
   - `jenkins-admins` group → Full admin access
   - `jenkins-developers` group → Build and configure jobs
   - `jenkins-viewers` group → Read-only access

---

### Part 10-12: Testing the SSO Flow

Let's verify that the integration works correctly.

**Step 1: Initiate Login**

![Testing the SSO Flow](Part%2010-Testing%20the%20SSO%20Flow.png)

1. Log out of Jenkins (if logged in)
2. Access http://localhost:8000
3. You should see a "Login with OpenID Connect" button

**Step 2: Redirect to Keycloak**

![Redirected to Keycloak's Login Page](Part%2011-redirected%20to%20Keycloak's%20login%20page.png)

1. Click "Login with OpenID Connect"
2. You'll be redirected to Keycloak's login page
3. Enter the credentials of the user you created earlier

**Step 3: Successful Authentication**

![After Successful Authentication](Part%2012-After%20successful%20authentication.png)

1. After successful authentication, you'll be redirected back to Jenkins
2. You should now be logged in as the Keycloak user
3. Your permissions will be based on your group memberships

> [!TIP]
> Check the Jenkins user dropdown to confirm you're logged in with the correct username from Keycloak.

---

## Troubleshooting

### Common Issues

**1. "Invalid redirect URI" error**
- **Cause**: Mismatch between configured redirect URI in Keycloak and actual Jenkins callback
- **Solution**: Ensure the redirect URI in Keycloak is exactly `http://localhost:8000/securityRealm/finishLogin`

**2. Groups not appearing in Jenkins**
- **Cause**: Group mapper not configured or client scope not assigned
- **Solution**: Review Part 5 and Part 6, verify token contains groups claim (Part 7)

**3. "Client authentication failed" error**
- **Cause**: Incorrect client secret or client ID
- **Solution**: Verify client secret in Keycloak matches Jenkins configuration

**4. Cannot access Jenkins after configuration**
- **Cause**: Authorization rules too restrictive
- **Solution**: Access Jenkins container directly and edit `config.xml` to reset security settings:
```bash
docker exec -it jenkins bash
vi /var/jenkins_home/config.xml
# Set <useSecurity>false</useSecurity> temporarily
```

**5. Keycloak unreachable from Jenkins**
- **Cause**: Network issues between containers
- **Solution**: Verify containers are on the same Docker network, check `docker-compose.yml`

### Debugging Tips

**View Jenkins Logs:**
```bash
docker logs -f jenkins
```

**View Keycloak Logs:**
```bash
docker logs -f keycloak
```

**Check Token Contents:**
Use jwt.io to decode tokens and verify claims are present

**Network Connectivity:**
```bash
docker exec -it jenkins curl http://keycloak:8080/realms/jenkins-realm/.well-known/openid-configuration
```

---

## Security Considerations

> [!CAUTION]
> This setup is for **development and learning purposes only**. For production use, implement the following security measures:

### For Production Deployment

1. **Use HTTPS/TLS**
   - Configure SSL certificates for both Jenkins and Keycloak
   - Use proper DNS names instead of localhost
   - Update redirect URIs to use HTTPS

2. **Strong Credentials**
   - Change default Keycloak admin password
   - Use strong, unique passwords for all users
   - Enable password policies in Keycloak

3. **Network Security**
   - Place Jenkins and Keycloak behind a reverse proxy (nginx, Traefik)
   - Use internal Docker networks
   - Implement firewall rules

4. **Token Security**
   - Configure appropriate token expiration times
   - Enable refresh token rotation
   - Use token introspection for better security

5. **Database Security**
   - Use strong PostgreSQL credentials
   - Enable SSL for database connections
   - Regular backups

6. **Additional Keycloak Hardening**
   - Enable brute force detection
   - Configure account lockout policies
   - Enable audit logging
   - Regular security updates

7. **Jenkins Security**
   - Keep Jenkins and plugins updated
   - Implement proper authorization matrix
   - Enable CSRF protection
   - Configure agent-to-controller security

---

## Additional Resources

- [Keycloak Documentation](https://www.keycloak.org/documentation)
- [OpenID Connect Specification](https://openid.net/connect/)
- [Jenkins OIDC Plugin](https://plugins.jenkins.io/oic-auth/)
- [OAuth 2.0 and OIDC Explained](https://auth0.com/docs/authenticate/protocols/openid-connect-protocol)

---

## Docker Services Summary

This project uses the following services:

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| Keycloak | `quay.io/keycloak/keycloak:26.4` | 8080 | Identity Provider (IdP) |
| PostgreSQL | `postgres:17` | 5433 | Database for Keycloak |
| Jenkins | `jenkins/jenkins:latest` | 8000 | CI/CD Server (Client) |

**Starting Services:**
```bash
docker-compose up -d
```

**Stopping Services:**
```bash
docker-compose down
```

**Viewing Logs:**
```bash
docker-compose logs -f
```

**Rebuilding Services:**
```bash
docker-compose up -d --build
```

---

## License

This project is for educational purposes. Feel free to use and modify as needed.

## Author

Ibrahim Elothmani - [GitHub](https://github.com/ibrahimelothmani)

---

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -am 'Add some improvement'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Open a Pull Request
