# Workspace notifications — channel + events + filters playbook

Manage a workspace's load-test **notifications** — OctoPerf's **runtime
alerting layer**. A notification fires a **channel** (email, Slack, Teams,
Google Chat, Webex, HTTP webhook, or JIRA) when a performance **test run**
reaches a chosen event (started / ended / passed / failed / error), optionally
narrowed by **filters**. Their only job is to tell people about test-run
outcomes (JIRA additionally comments on / transitions board issues), so
they belong to the runtime side of OctoPerf: they pair with `run_scenario` and
`schedule_scenario_*`, and a scenario surfaces the notifications that will fire
for its runs (`list_notifications_matching_scenario`). Read this before creating
or updating one.

## Mental model

A notification = **channel** + **events** (>=1) + **filters** (optional).

- **Channel** — where the message goes. One typed create tool per channel:
  `create_email_notification`, `create_slack_notification`,
  `create_teams_notification`, `create_google_chat_notification`,
  `create_webex_notification`, `create_http_notification`,
  `create_jira_notification` (multi-step — see Workflow 4).
- **Events** (`eventIds`, at least one): `TEST_STARTED`, `TEST_ENDED`,
  `TEST_PASSED`, `TEST_FAILED`, `TEST_ERROR`.
- **Filters** (optional) restrict *which* tests fire it. Omit → fires for every
  test in the workspace.

## Secrets are write-only

Tokens, webhook URLs and HTTP header values are **write-only**: you supply them
in plaintext on create, and they are **never returned** by `get_notification` /
`list_notifications_by_workspace` (only a non-secret `target` marker and a
`details` map of non-secret attributes). Consequence: **on update you must
resend the secret** — an update overwrites the whole notification and cannot
preserve a secret it cannot read back.

## Filters shape

Pass an optional flat `filters` object; any omitted part means "no such filter":

```
filters: {
  scenarioName?:        { search: "...", matchType?: CONTAINS|EQUALS|EQUALS_IGNORECASE|CONTAINS_IGNORECASE|STARTS_WITH|ENDS_WITH },
  testDurationMinutes?: { min?: <n>, max?: <n> },
  concurrentUsers?:     { min?: <n>, max?: <n> }
}
```

`matchType` defaults to `CONTAINS`. A range with only `min` means "at least",
only `max` means "at most".

## Workflow 1 — create and test a notification

1. `list_workspaces` → pick the workspace.
2. Pick the channel and call its `create_*_notification` tool with
   `workspaceId`, the channel secret/target, `eventIds` (>=1) and optional
   `filters`. For email, set `attachPdfReport: true` to attach the PDF report.
3. `test_notification(notificationId)` sends a **real** dummy message to the
   channel so the user can confirm it is wired correctly. It is rate-limited
   server-side to one test per user every 5 seconds; the result reports whether
   the dispatch was **accepted**, not delivered.
4. Reachability: Teams / Google Chat / HTTP webhook URLs must be reachable
   **from OctoPerf's servers** (public internet on SaaS).

## Workflow 2 — inspect what exists / what would fire

- `list_notifications_by_workspace(workspaceId)` — all notifications, secret-free.
- `get_notification(notificationId)` — one, with its non-secret `details`.
- `list_notifications_matching_scenario(workspaceId, scenarioId)` — the
  **scenario ↔ notification link** (the same set the scenario page shows):
  which notifications a given scenario's runs would fire (by their filters).
  Run it **before launching (`run_scenario`) or scheduling
  (`schedule_scenario_*`) a scenario** to confirm the right people will hear
  about the outcome. If it returns empty, consider adding a `TEST_FAILED` /
  `TEST_ERROR` notification so runs don't fail silently.

## Workflow 3 — update / delete

- `update_<channel>_notification(notificationId, …)` **overwrites the whole
  notification** (events, filters, channel config). Same params as the matching
  create **plus** `notificationId`. **Resend the secret** — it is write-only.
  Use the SAME channel's update tool as the notification's channel.
- `delete_notification(notificationId)` — permanent, cannot be undone.

## Workflow 4 — JIRA notifications (multi-step)

A JIRA notification comments on the issues in a board's source status column
when the chosen events fire, and (for Passed / Failed / Error) transitions them
to a target column. It needs board / status / transition **ids** that only JIRA
knows, so build the config step by step with the read-only lookup tools before
calling `create_jira_notification`:

1. `check_jira_notification_connection(connection)` → confirm `reachable: true`.
   The `connection` is `{ url, connectionType, email?, apiKey }`:
   - `JIRA_CLOUD` — `url` ends in `.atlassian.net`, `email` = account email,
     `apiKey` = API token.
   - `JIRA_DATA_CENTER_BASIC` — `email` = username, `apiKey` = password.
   - `JIRA_DATA_CENTER_ACCESS_TOKEN` — no `email`, `apiKey` = personal access token.
2. `list_jira_notification_boards(connection)` → pick a board; keep its
   `boardId` **and** `projectId` (the project is derived from the board).
3. `list_jira_notification_statuses(connection, projectId)` → pick the source
   "To do" `statusId`.
4. `list_jira_notification_transitions(connection, boardId)` → pick the
   `passTransitionId` / `failTransitionId` / `errorTransitionId` for whichever
   of TEST_PASSED / TEST_FAILED / TEST_ERROR you enable. (Transitions are read
   from existing issues, so the board must contain at least one issue.)
5. `create_jira_notification(workspaceId, connection, board, eventIds, filters?)`
   where `board` groups `{ boardId, projectId, statusId, issueNameFilter?,
   passTransitionId?, failTransitionId?, errorTransitionId? }`. `issueNameFilter`
   is a summary word (empty = all issues). A transition id only matters when its
   event is selected.

`apiKey` is write-only like every other secret — resend it on
`update_jira_notification`. All four lookup tools contact JIRA (open-world) and
may take a few seconds.

Note: `test_notification` only posts a **dummy comment** on the source-column
issues to confirm wiring — it does **not** transition anything. Issues are
actually moved (via the pass/fail/error transitions) only on a **real test
run's** Passed/Failed/Error event.

## Notes

- The lookup tools and `test_notification` reach an external target (JIRA / the
  configured channel) — the open-world tools in this set. Everything else stays
  within the OctoPerf dataset.
