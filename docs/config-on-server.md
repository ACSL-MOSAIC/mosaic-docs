---
title: Config on Server
nav_order: 6
---

# Config on Server

This page describes how to sign up and register robots on the MOSAIC dashboard.

The hosted open-use server is available at **[https://mosaic.acslgcs.com](https://mosaic.acslgcs.com)**.

---

## Account Types

MOSAIC supports two account types:

| Type | Sign-up | Robot & dashboard sharing |
|:---|:---|:---|
| **Personal** | Self sign-up with email | Not shared |
| **Organization** | Admin adds users manually | Shared across all members of the organization |

---

## Sign Up

### Personal Account

1. Go to **[https://mosaic.acslgcs.com](https://mosaic.acslgcs.com)**.
2. Click **Sign Up**.
3. Enter your email and password to create an account.
4. Log in and proceed to register your robot.

### Organization Account

Organization accounts are managed by an admin. Users cannot self-register — the organization admin must add them:

1. The admin logs in and navigates to **Settings → Members**.
2. The admin adds the user's email.
3. The invited user receives credentials and can log in.

All members of the same organization share the same robots and dashboard configuration.

---

## Robot Registration

1. Log in to the dashboard.
2. Navigate to **Robots → Add Robot**.
3. Give the robot a name. The server generates:
   - A **Robot UUID** — used as `robot_id` in the robot's `mosaic_config.yaml`
   - An **Auth Token** — used as `params.token` in the robot's `mosaic_config.yaml`

Copy both values into the robot's configuration file. See [Config on Robot](./config-on-robot) for the full YAML format.