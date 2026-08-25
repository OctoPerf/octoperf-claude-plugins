# Workspace — who works where, and with what

A workspace is the container everything else lives in. Read this before adding
someone to one, changing what they can do, or wondering why a provider a
colleague swears exists is nowhere to be found.

## What a workspace scopes

Two levels, and confusing them is the most common source of "I can't see it":

| Scoped to the **workspace** | Scoped to a **project** |
|---|---|
| Members and their roles | Virtual users, scenarios, variables, files |
| Load-generator providers and their agents | Monitors (they *run* through a workspace agent) |
| Notifications (email, Slack, Teams, JIRA, …) | Bench results and reports |

So a provider added to workspace A cannot run a scenario of a project in
workspace B, a notification fires for every project of its workspace, and
`list_workspace_agents` takes the workspace that owns the monitor's project —
not the project. Deep dives live in `octoperf://skills/onpremise-agent`
(providers and agents), `octoperf://skills/notifications` (channels and events)
and `octoperf://skills/monitoring` (monitors and the agents that carry them).

## Create sparingly, because deleting is not yours to do

Anyone can create a workspace; nobody can casually remove one. Deletion takes
the **platform administrator** on-premise, or an **OctoPerf support request** on
SaaS — deliberately, since removing a workspace takes every project, provider
and past result it holds. Being admin *of the workspace* is not enough: that is
the platform role, a different thing.

Before `create_workspace`, call `list_workspaces` and reuse. A new workspace
earns its place when a **separate set of members or load generators** is needed
— a client, a team, an isolated environment. Merely grouping work is what
`create_project` is for. The creator becomes admin of what they create.

`update_workspace` renames or re-describes one. The name is what every member
sees in their switcher, so a rename is visible to all of them at once.

Names are 80 characters at most, descriptions 2000 — the web form's own bounds.
Past them the edit form is invalid the moment it opens, so the write is refused
rather than truncated.

## Knowing who you are before touching who they are

Members are designated by **e-mail**, including you. `get_current_user` gives the
address the session is connected under, so `remove_workspace_member` or
`set_workspace_member_role` on that address is you demoting or evicting
*yourself* — which no server rule catches: the two rules below are about the
workspace keeping an admin, never about who is calling. It also returns
the role held on every workspace, which is what tells you whether the change you
are about to make is even permitted: the same account is `ADMIN` on one
workspace and `VIEWER` on the next.

Its `platformAdmin` flag is a different axis. It says the account administers
the OctoPerf *installation*, and the server lets it through every workspace
permission check whatever role sits next to that workspace — so a platform
admin writes where its role alone says it could not.

## Members

`list_workspace_members` answers "who has access, and to do what". Members are
named by the **e-mail their account is registered under**; every member tool
addresses them that way. `list_workspaces` carries the connected account's own
role, one line per workspace.

| Role | What it allows |
|---|---|
| `ADMIN` | Everything, including managing members and workspace settings |
| `TESTER` | Create projects, reach every resource of the existing ones |
| `TEST_EXECUTOR` | Full project control, except editing virtual users |
| `VIEWER` | Read virtual users and reports, change nothing |
| `CUSTOM` | An access assembled from fine-grained authorizations — read-only here, only the web UI edits it |

**`add_workspace_member` is not an invitation.** The account must already exist
on the platform: an unknown address is refused and nothing is e-mailed, so a
colleague without an OctoPerf account has to sign up first. Without a role they
arrive as `VIEWER`. Access is workspace-wide — the new member reaches every
project, provider and notification it holds — so confirm before granting
anything above `VIEWER`.

`set_workspace_member_role` replaces a member's access; `remove_workspace_member`
takes it away entirely, leaving their account and their other workspaces alone.

**Two rules the server enforces**, worth knowing before the call rather than
after the refusal:

- **A workspace keeps at least one admin.** The last one cannot be demoted or
  removed.
- **A user stays admin of at least one workspace.** Demoting yourself fails when
  this is the only workspace you administer — even if someone else admins it too.

Both come back as a refusal carrying that sentence. Read the roster first,
decide who takes over, then demote. When the member being changed is the
connected account itself — compare with `get_current_user` — say so and get a
confirmation before the call: nothing server-side stops you, and losing your own
access is not undoable by you.

## A workspace from scratch

1. `get_current_user` — state which account is doing this, and what it may do.
2. `list_workspaces` — reuse rather than create, if anything fits.
3. `create_workspace` — a separate set of members or load generators is the
   reason to make one.
4. `add_workspace_member` per colleague, with the role each one needs.
5. `create_onpremise_docker_provider` if the tests must run from your own
   machines (`octoperf://skills/onpremise-agent`), otherwise the shared
   OctoPerf Cloud providers are already there.
6. `create_email_notification` and friends so the workspace tells someone when a
   test ends (`octoperf://skills/notifications`).
7. `create_project` — and from there the design tools take over.
