# Assignment: Orchestrating a Multi-Container AI Agent with the Docker CLI

__IMPORTANT:__ Please read the following instructions before commencing this assignment, as it may significantly impact your grade:

+ Read the entire specification before commencing development, and clarify any aspects you do not understand.
+ Clone this repo as it represents the starting point for your work on this assignment.
+ Set the visibility of your repo to Private to prevent potential plagiarism of your work.
+ Your repo's Git log must have a clear, understandable and coherent history of the work on this assignment. The commit message must summarise the work performed (the task you were working on) at each stage.  See the Deliverables section below.
+ You are expected to spread the effort required to complete the assignment over the period involved; do not cram the work into a few days just before the deadline, as this is not conducive to proper learning and will result in a minimal mark.


## Overview

You will deploy an **AI agent that answers natural-language questions about a music-store database**. The agent works out what SQL to write, runs it against a PostgreSQL database through a tool gateway, and reports the answer.

The system has four cooperating containers. Your task is to identify the Docker CLI commands that build, configure, and start the whole system. Add the command sequence to a simple script file (`run-system.sh`) that starts from a clean machine and finishes with the agent printing an answer.


## The application

All four containers share a private, user-defined Docker network. On that network, each container can reach the others by container name. You choose the container names; the labels in the table below describe each container's role.

![][arch]

| Container      | Role | Image |
|---|---|---|
| `Database` | PostgreSQL server that holds the Chinook music-store dataset | Official `postgres` image from Docker Hub |
| `Seeder` | One-shot job that copies the provided SQLite file into Postgres using `pgloader`, then exits | Built from the **`seeder`** stage of the provided `Dockerfile` |
| `MCP Gateway` | A Model Context Protocol (MCP) gateway. It exposes an SQL query tool to the agent over HTTP using Server-Sent Events (SSE). The tool is attached to an MCP server running as a container created by the Gateway. | `docker/mcp-gateway:latest` from Docker Hub |
| `AI Agent` | Python agent. It takes a question, calls the gateway's query tool as often as it needs, and prints its reasoning and final answer | Built from the **`agent`** stage of the provided `Dockerfile` |

For inference, the agent uses an LLM (Large Language Model) hosted on **Ollama Cloud** through its OpenAI-compatible API. You must create an account on this service and generate an API key for this assignment - [see here][ollama].

FYI: SSE is a standard where a client opens one long-lived HTTP connection and the server keeps it open to push a stream of text events to the client as they happen.

### Data flow

```
question ──► agent ──(MCP over SSE)──► Gateway ──(SQL)──► database
                │                                               ▲
                └──(OpenAI-compatible API)──► Ollama Cloud      │
                                                    seeder ─────┘ (loads data once at startup)
```
This flow implies a dependency graph for the system; for example, the database container must start before the seeder container. The MCP Server container does not appear in the flow for simplicity; it is internally managed by the Gateway container.

## Configuration files

The following files are provided in the project directory:

| File | Purpose |
|---|---|
| `Dockerfile` | Multi-stage build with two named targets, `seeder` and `agent` |
| `Chinook.db` | The SQLite source dataset for the music store |
| `mcp_config.yaml` | Tells the gateway how to connect to the database (host, port, database name, user). It passes these to the cockroachdb MCP server which actually talks to the database |
| `mcp_secret.env` | The database password, formatted as a gateway secret |
| `secret.ollama-api-key` | Contains only your Ollama Cloud API key. Never commit it to version control. |

__Important:__ You must edit the last three files in the above list by replacing the placeholders with your settings, e.g. username: [DATABASE_USERNAME] in `mcp_config.yaml`. The square brackets should be removed in all cases. 

## Container specifications

### Postgres Database

- Environment:
  - `POSTGRES_USER=[DATABASE_USERNAME]`
  - `POSTGRES_PASSWORD=[DATABASE_PASSWORD]`
  - `POSTGRES_DB=[DATABASE_NAME]`

### Database seeder

- Mount `Chinook.db` from the project directory into the container at `/app/Chinook.db`.
- Environment:
  - `SQLITE_FILE=/app/Chinook.db`
  - `DATABASE_URL=[CONNECTION_STRING]`

The format for the database connection string is: *postgres://[DATABASE_USERNAME]:[DATABASE_PASSWORD]@[DATABASE_DOMAIN_NAME]:5432/[DATABASE_NAME]*.

### MCP Gateway

The gateway is a proxy. It doesn't talk to the database itself; it pulls the mcp/cockroachdb MCP server image from Docker Hub and starts it as a separate (internal) container, using the Docker socket. The cockroachdb MCP server is what actually connects to the database.
- Mounts:
  - `/var/run/docker.sock` → `/var/run/docker.sock`  (Docker Socket connection)
  - `mcp_config.yaml` → `/mcp_config.yaml`
  - `mcp_secret.env` → `/run/secrets/database-url`
- The following set of command-line arguments must be passed to the Gateway image, in this order, at runtime:
  ```
  --transport=sse
  --secrets=/run/secrets/database-url
  --allow-unauthenticated
  --servers=cockroachdb
  --tools=execute_query
  --config=/mcp_config.yaml
  ```
- The `cockroachdb` MCP server speaks the Postgres wire protocol, so it works with a standard Postgres server.
- The gateway listens on port **8811** inside the network. That port does not need to be published to the host.

### AI Agent

- Mount `secret.ollama-api-key` → `/run/secrets/openai-api-key`. The agent's entrypoint reads the API key from this path, and uses it to authenticate with the Ollama Cloud service.
- Environment:

  | Variable | Value |
  |---|---|
  | `MCP_SERVER_URL` | `http://[GATEWAY_DOMAIN_NAME]:8811/sse` |
  | `DATABASE_DIALECT` | `PostgreSQL` |
  | `QUESTION` | `What are the top 3 albums by sales?` |
  | `OPENAI_API_BASE_URL` | `https://ollama.com/v1` |
  | `OPENAI_MODEL_NAME` | `gpt-oss:20b` |


__NOTE:__ Placeholders are used several times in the above details - [Placeholder]. You must replace them with your settings - do not include the square brackets in the setting.

## Deliverables

You should submit to Moodle a text file containing the URL of your GitHub repo (the clone of this repo) - use [this link][submit].

Apart from editing the configuration files described above, the required changes to the starter repo are as follows:

1. **`run-system.sh`**, which runs the full system end to end, i.e. starts the set of containers on Docker Desktop.
2. **`cleanup.sh`**, which removes every Docker construct created by the run script so that it can be rerun cleanly.
3. **Git history**, which shows a credible history of the progress made to complete this assignment. Each docker command added to the two scripts should be committed separately, and have a suitable commit message stating its purpose. Bug fix and improvement commits are allowed but the commit messages must clearly explain the reasons. 


__Completion date:__ 26/10/2026, 6pm

__Weighting:__ 45%.

## Grading spectrum

- **40–50%:** Most containers start on a shared user-defined network, but the agent does not yet produce an answer. `cleanup.sh` removes every Docker construct the run script creates. The Git history shows some step-by-step progress.
- **51–65%:** Starting from a clean machine, `run-system.sh` brings up the full system and the agent prints an answer to the question. Each Docker command is committed separately with a clear message.
- **66–80%:** As above, plus the containers start in the right order: the seeder runs only after the database has started, and the gateway and agent start only after the seeder has finished (e.g. using `docker wait`, or by running the seeder in the foreground). Running `cleanup.sh` then `run-system.sh` works every time. `cleanup.sh` removes all containers, volumes, the network and built images.
- **81–100%:** As above, plus good Docker CLI practice: files are mounted read-only where possible, one-shot containers remove themselves (`--rm`) where this doesn't conflict with how they are waited on, and all Docker constructs are labelled so `cleanup.sh` can remove them with label filters.

[ollama]: https://ollama.com/
[arch]: ./img/arch.jpg
[submit]: https://moodle.setu.ie/mod/assign/view.php?id=4861067