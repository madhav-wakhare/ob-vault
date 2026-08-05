Vaultwarden is the company's password manager for storing and accessing shared credentials, such as logins for common tools and services.

Access to Vaultwarden requires:

1. Connecting to the company network (Headscale).
2. Signing in with the shared Vaultwarden account.

The steps below explain the process.

---

 Step 1 — Connect to the Company Network

Before accessing Vaultwarden, connect your device to the company network by following the **Headscale setup guide**.

Once your device is connected to the internal network, continue to **Step 2**.

---

Step 2 — Open Vaultwarden

Open the following URL in your browser:

[](https://vaultwarden.apps.one2n.io/)**[https://vaultwarden.apps.one2n.io](https://vaultwarden.apps.one2n.io)**

If the page does not load, verify that your device is connected to the company network and then try again.

---

Step 3 — Sign In

Sign in using the shared company account:

- **Email:** `shared@one2n.in`
- **Master password:** Provided to you separately through a secure channel. It is **not** included in this document.

After signing in, you will have access to the shared credentials stored in the vault.

> **Note:** If you do not have the shared password, contact the team through a secure communication channel.

---

Browser Extension (Optional)

For a better day-to-day experience, you can use the Bitwarden browser extension, which is fully compatible with Vaultwarden.

1. Install the **Bitwarden** extension from your browser's extension store.
    
2. Open the extension. Before signing in, open **Settings** (gear icon) on the login screen.
    
3. Change **Region** to **Self-hosted**.
    
4. Set the **Server URL** to:
    
    ```
    <https://vaultwarden.apps.one2n.io>
    ```
    
5. Save the configuration and sign in using:
    
    - **Email:** `shared@one2n.in`
    - **Master password:** Shared separately

Your device must remain connected to the company network while using the extension.

> You can also access Vaultwarden directly through the web interface without installing the browser extension.

---

Important Guidelines

Since everyone uses the same Vaultwarden account, please follow these guidelines:

- **Do not change the master password.** Any password change must be coordinated by the team, as it affects all users.
- **Do not enable two-factor authentication (2FA)** on the shared account. This would prevent other employees from accessing the vault.
- **Do not share the master password outside one2n** or post it in public Slack channels, tickets, emails, or documents.
- Ensure your device remains connected to the company network while using Vaultwarden.
- Treat all credentials stored in the vault as confidential company information.

---

Troubleshooting

| Issue                               | Resolution                                                                                                        |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Vaultwarden page does not load      | Verify that your device is connected to the company network (Headscale).                                          |
| Unable to connect to Headscale      | Follow the Headscale setup guide or contact the team.                                                             |
| Login fails with "Invalid password" | Verify that you are using the latest shared password. Contact the team if necessary.                              |
| Browser extension cannot connect    | Ensure the Server URL is `https://vaultwarden.apps.one2n.io` and your device is connected to the company network. |

---

Future Support for Individual Accounts

Google Workspace Single Sign-On (SSO) has already been configured for Vaultwarden. It is not used in the current shared-account workflow, but it enables a future transition to individual user accounts if needed—for example, to manage team-specific credentials such as HR's Zoho Sign access.

No action is required from users at this time.

---

Reference

| Item                | Value                                             |
| ------------------- | ------------------------------------------------- |
| Vaultwarden URL     | `https://vaultwarden.apps.one2n.io`               |
| Network Requirement | Connected to the company network via Headscale    |
| Shared Account      | `shared@one2n.in`                                 |
| Master Password     | Shared securely by the team (not documented here) |