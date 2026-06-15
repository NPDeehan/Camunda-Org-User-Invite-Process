# Camunda Org User Invite Process

A Camunda 8 process for inviting users to a Camunda Cloud organization in bulk. Users can be entered as a comma-separated list or added one at a time via a repeating form.

---

## Important: Camunda 8.7+ Breaking Change

> **As of Camunda 8.7, invoking the "Invite Single User Process" directly via its published start form no longer works.**

In earlier versions, the `Invite Single User Process` had a public start form that could be used to invite one user at a time by sharing a URL. This approach is **no longer supported** in 8.7+. All user invitations must now go through the main **Invite Users Process**, which calls the single-user process internally as a subprocess for each user in the list.

---

## Process Overview

The solution is made up of three BPMN processes and three forms:

### Processes

| File | Process ID | Purpose |
|------|-----------|---------|
| `Invite Users Process.bpmn` | `UserInviteProcess` | Main entry point — collects emails and loops through invitations |
| `Invite Single User Process.bpmn` | `CreateUserProcess` | Called subprocess — authenticates and invites one user |
| `Create Activation Token.bpmn` | `CreateActivationToken` | Error-recovery subprocess — refreshes the API token when auth fails |

### Forms

| File | Purpose |
|------|---------|
| `Enter Users for Signup.form` | Starting form — enter emails via CSV or repeating list |
| `User to be signed up.form` | Used only internally by the subprocess (not a public entry point) |
| `Confirm that your Key has been updated.form` | Shown during token recovery to display the new access token |

---

## How the Process Works

1. **Start:** A user opens the published start form for `Invite Users Process` and enters email addresses using one of two methods:
   - **CSV:** Paste a comma-separated list of emails (e.g. `user1@example.com,user2@example.com`)
   - **List:** Add emails individually via a repeating form row

2. **Loop:** The process iterates over each email address and calls `CreateUserProcess` for each one.

3. **Authenticate:** For each user, `CreateUserProcess` calls the Camunda Cloud OAuth endpoint to obtain a bearer token using a client credentials flow.

4. **Invite:** Using that token, it calls the Camunda Cloud Members API (`POST https://api.cloud.camunda.io/members/{email}`) to send the invitation. The invited user is granted these default org roles: `taskuser`, `analyst`, `developer`, `modeler`.

5. **Error Handling:**
   - **401 Unauthorized:** The token has expired. The main process catches this and calls `CreateActivationToken`, which retrieves a new token and displays it so you can update the secret manually.
   - **403 Forbidden:** The API client has insufficient permissions or the org has no available seats. The process ends gracefully.

---

## Required Secrets

Secrets are configured in **Camunda Console** under your cluster's **Secrets** tab. The processes reference secrets using the `{{secrets.NAME}}` syntax.

You need **one API client** with the `Members` scope (to invite users). The secrets for this client are used in two processes with slightly different names — both must be configured.

### Secrets to configure

| Secret Name | Used In | Value |
|-------------|---------|-------|
| `creat_user_client_id` | `Invite Single User Process.bpmn` | Client ID of your API client |
| `create_user_secret` | `Invite Single User Process.bpmn` | Client Secret of your API client |
| `CamundaClientID` | `Create Activation Token.bpmn` | Client ID of your API client (same client) |
| `CamundaClientSecret` | `Create Activation Token.bpmn` | Client Secret of your API client (same client) |

> **Note:** `creat_user_client_id` is a typo in the process (missing the "e" in "create"). Use exactly this spelling when creating the secret, or update the BPMN to match your preferred name.

> **Note:** `CamundaClientID` / `CamundaClientSecret` and `creat_user_client_id` / `create_user_secret` can point to the same API client — there is no requirement for two separate clients.

---

## How to Get the API Client Credentials

1. Go to [Camunda Console](https://console.cloud.camunda.io)
2. Select your organization
3. Navigate to **Administration > API** (or **Organization > API clients**)
4. Click **Create new client**
5. Give it a name (e.g. `user-invite-process`)
6. Under **Scopes**, enable the **Members API** scope (required to invite users)
7. Click **Create** and note down:
   - **Client ID** → use for `creat_user_client_id` and `CamundaClientID`
   - **Client Secret** → use for `create_user_secret` and `CamundaClientSecret`

> The client secret is only shown once. Save it immediately.

---

## How to Set Secrets in Camunda Console

1. Go to [Camunda Console](https://console.cloud.camunda.io)
2. Select your cluster
3. Click the **Connectors** tab (or find **Secrets** in the cluster settings)
4. Click **Create** and add each secret by name and value

You need to add all four secrets listed in the table above.

---

## Deployment

1. Open **Camunda Modeler**
2. Deploy all three BPMN files to your cluster:
   - `Invite Users Process.bpmn`
   - `Invite Single User Process.bpmn`
   - `Create Activation Token.bpmn`
3. The forms are embedded in the BPMN files — no separate form deployment is required
4. Start a new instance of `UserInviteProcess` via Tasklist or the published start form

> All three processes must be deployed to the same cluster. The main process calls the others by Process ID, so the IDs must not be changed.

---

## Token Expiry and Recovery

OAuth tokens from Camunda Cloud expire. When a token expires mid-run, the subprocess throws a `401 Unauthorized` error. The main process catches this on a boundary event and automatically calls `CreateActivationToken`.

That process:
1. Requests a new token using the `CamundaClientID` / `CamundaClientSecret` secrets
2. Displays the new token in a user task
3. Prompts you to manually copy the token and update the relevant secret in Camunda Console

This is a manual recovery step by design — the process cannot update its own secrets at runtime.

---

## Assigned Roles

All invited users are granted the following org-level roles by default (hardcoded in the API call body):

- `taskuser`
- `analyst`
- `developer`
- `modeler`

To change the default roles, edit the HTTP connector body in `Invite Single User Process.bpmn`.
