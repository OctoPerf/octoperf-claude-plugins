# On-premise providers & agents — capacity-first playbook

Set up and run OctoPerf load tests from **your own machines** instead of the
Cloud. This is the deeper playbook behind the on-premise tools; read it before
creating a provider or installing an agent.

## Mental model — a provider is a *localized load capacity*

Think in **virtual users (VUs) per location** — from machines that can reach
the target — not megabytes.

- A **provider** *supplies* load generators **through agents** you install on
  your own machines. It does **not** itself run load generators — it is the
  configuration (memory sizing) plus the set of locations the agents attach to.
- Each **location** is a named place (a city, region or country) with map
  coordinates. A test can spread load across locations (e.g. Paris + Madrid).
- An **agent** is one Docker container you run on one machine. At run time an
  agent **starts load generators on demand**, several per machine as memory
  allows — so **one agent handles many VUs**, not one.
- The provider's memory config is inherited by every load generator, so
  `RAM per machine × usable % ÷ memory-per-VU ≈ VUs per agent`. Total provider
  capacity = that, times the number of agents, per location.

Capacity is what the user cares about ("I want 10 000 VUs in Paris, 2 000 in
Madrid"). The memory config is just the means.

## Accessibility — agents must reach the target (and stay off it)

On-premise exists mostly for **reach**: the OctoPerf Cloud cannot hit an app
that is internal, behind a firewall or on a private network. The fix is to run
agents on machines that sit **inside that network**.

- Every agent machine **must be able to reach the application under test** —
  same LAN / VPN / VPC, DNS resolves, ports open. An agent that can't reach the
  target only produces connection errors. This reachability, more than
  geography, is usually the real reason a location exists.
- **Never install an agent on the machine that hosts the app under test.** The
  agent spawns load generators there; their CPU/RAM steal from the app server
  and skew every result. Use a *separate* machine on the same network.

## ⚠️ The one mistake to prevent: several agents on one machine

Installing **more than one agent on the same machine does NOT add capacity** —
it is bounded by that machine's RAM, it **skews the capacity accounting** (each
agent assumes it owns the whole machine), and it causes memory problems.

**Rule: one machine = one agent.** For more capacity, add more **machines**.

Detect it with `list_provider_agents`: an agent with `duplicateOnHost=true`
(more than one running `octoperf/docker-agent` container on its host) is a
duplicate. Do **not** trust `remoteAddr` for this — it is often a Docker-internal
`172.17.0.x` address shared by unrelated agents.

## Sizing dialogue — ask, propose, default

Drive the conversation around capacity; keep memory tuning for edge cases.

1. **Ask** the essentials: target VUs and on which locations; from which network
   the app under test is reachable (internal/private targets drive on-premise
   and where the agent machines must live); how much RAM each machine can
   dedicate to tests; the dominant test type (JMeter HTTP vs real browser /
   Playwright).
2. **Propose**: from RAM-per-machine and memory-per-VU, estimate VUs-per-agent,
   then the number of agents to install per location
   (`agents = ceil(target VUs / VUs-per-agent)`). Say it plainly, e.g.
   "~800 VUs per 8 GB machine → about 13 agents in Paris". Be honest: on-premise
   capacity is the machines *you* provide.
3. **Default** everything else. On-premise defaults are sensible (RAM 4096 MB,
   90 % usable, 8 MB/VU JMeter, 1024 MB/VU real-browser, no slot limit). Do
   **not** interrogate the user about individual memory fields.
4. **Tune only on symptoms** (edge case): raise `vuMb` / `heavyVuMb` /
   `realBrowserVuMb` only when the user reports a very heavy VU, or sees load
   generator memory/CPU pegged during a validation or a small trial run. Then
   recreate/adjust with the higher per-VU value.

## Workflow 1 — create a provider and install agents

1. `list_workspaces` → pick the workspace; `list_docker_providers_by_workspace`
   to see existing providers (filter `driver=ON_PREMISE`).
2. `create_onpremise_docker_provider` with the target `locations` (fill each
   location's latitude/longitude from its city/region name; names are lowercase
   `a-z0-9_-`, e.g. `paris`, `new_york`, `ile_de_france`). Leave memory params
   unset unless the sizing dialogue called for a change.
3. For **each machine** at a location, `get_onpremise_agent_command`
   (providerId + location) → give the user the `docker run` command. Agents run
   on **Linux** — strongly recommend a Linux host (the install guide is
   Linux-only). The machine needs a container runtime: if Docker is not
   installed, tell them to run `wget -qO- https://get.docker.com/ | sh` first.
   The same command also works with **Podman** (Docker-CLI compatible) — see
   <https://api.octoperf.com/doc/on-premise-agent/provider-type/on-premise/#im-using-podman-instead-of-docker>.
   Repeat per machine to reach the planned agent count — **one agent per
   machine**.
4. Before handing over a command, run `list_provider_agents` and warn on any
   `duplicateOnHost=true`. Remind the user to run each command on a machine that
   has no OctoPerf agent yet.
5. Agents take a minute to register; re-run `list_provider_agents` (or
   `get_provider_usage`) until they appear `UP`. If an agent never shows up, the
   command embeds a `SERVER_URL` — check it is the reachable backend host, not a
   local/incorrect address.

## Workflow 2 — check current capacity and load

`get_provider_usage(providerId)` reports, per location, how many agents run and
how much memory is in use, plus the per-machine memory config. This is the
**real** capacity (it depends on the agents actually installed). Compare it to
the plan: too few agents at a location → install more; a location at high memory
use during a run → the machines are near saturation.

## Workflow 3 — manage locations (and reinstall affected agents)

- **Add** a location: `add_provider_location`, then install an agent for it
  (Workflow 1, step 3). Adding a location alone adds no capacity.
- **Change coordinates only**: `update_provider_location` with new lat/long and
  no new name — agents stay attached, nothing to reinstall.
- **Rename** a location: `update_provider_location` with `newName`. Agents are
  bound to the location *name*, so a rename **orphans** them — the result lists
  `orphanedAgents`. For each: run its `uninstallCommand`
  (`sudo docker rm -f <name>`) on its machine, then `get_onpremise_agent_command`
  for the new name and reinstall.
- **Remove** a location: `remove_provider_location`. Same orphaning — remove the
  returned `orphanedAgents` from their machines.

## Removing an agent

An on-premise agent runs on the user's own machine, so the clean way to remove
one is the `uninstallCommand` from `list_provider_agents`
(`sudo docker rm -f <name>`), run on that machine. (The REST terminate path is
peer-operated — it needs a second agent — so it does not fit a single local
agent.)

## Out of scope (for now)

Docker on Linux only (strongly recommended host). Podman works with the
generated command too — see
<https://api.octoperf.com/doc/on-premise-agent/provider-type/on-premise/#im-using-podman-instead-of-docker>.
Kubernetes and Windows (Vagrant) agents are not covered. For those, point the
user to the OctoPerf on-premise docs:
<https://api.octoperf.com/doc/on-premise-agent/providers/>.
