# Arness

Arness is Andreas Jansson's agent harness built on the Cloudflare Agents SDK. It is named after Andreas' favorite ice cream.

## Code mode bindings

Arness exposes capabilities to agents as Code Mode bindings: typed TypeScript classes that the model can call from generated JavaScript. Instead of loading one JSON schema per operation into context, Code Mode gives the model a compact typed API surface and runs the generated code in an isolated Dynamic Worker sandbox.

```ts
declare const File: {
  read(path: string, options?: { startLine?: number; endLine?: number }): Promise<string>;

  /**
   * Create a new file in Artifacts. Throws if the file already exists.
   */
  writeNew(path: string, contents: string, commitMessage: string): Promise<void>;

  /**
   * Replace an existing file in Artifacts. Throws if the file does not exist.
   */
  replace(path: string, contents: string, commitMessage: string): Promise<void>;

  /**
   * Edit a file by replacing either an exact content block or the range from startContent to endContent.
   * File mutations are committed to Artifacts using commitMessage.
   */
  edit(
    path: string,
    edit:
      | { originalContent: string; newContent: string }
      | { startContent: string; endContent: string; newContent: string },
    commitMessage: string,
  ): Promise<void>;

  find(pattern: string, options?: { cwd?: string }): Promise<string[]>;
  grep(pattern: string, options?: { cwd?: string; include?: string; exclude?: string }): Promise<string>;

  /**
   * Rename or move a file in Artifacts and commit the change.
   */
  rename(oldPath: string, newPath: string, commitMessage: string): Promise<void>;

  /**
   * Delete a file from Artifacts and commit the change.
   */
  delete(path: string, commitMessage: string): Promise<void>;
};

declare const Web: {
  /**
   * Search the web using Exa.
   */
  search(query: string, options?: { limit?: number }): Promise<WebSearchResult[]>;

  /**
   * Fetch a URL. When markdown is true, HTML is converted to markdown before being returned.
   */
  fetch(url: string, options?: { markdown?: boolean }): Promise<string>;

  request(url: string, init?: RequestInit): Promise<Response>;
};

declare const Publish: {
  /**
   * Publish an Artifact-backed file to a stable URL. Markdown files are rendered as HTML.
   */
  file(path: string): Promise<{ url: string }>;
};

declare const Image: {
  /**
   * Read an image file from Artifacts and return it as model-visible image content.
   */
  read(path: string): Promise<ImageContent>;
};

declare const PDF: {
  /**
   * Read a PDF file from Artifacts and return it as model-visible document content.
   */
  read(path: string): Promise<PdfContent>;
};

declare const DB: {
  /**
   * Run SQL against the thread's D1 database.
   */
  select<T = unknown>(sql: string, params?: unknown[]): Promise<T[]>;
  insert(table: string, row: Record<string, unknown>): Promise<void>;
  createTable(sql: string): Promise<void>;
};

declare const RBAC: {
  /**
   * Grant a role to a user. Only admins can modify role assignments.
   */
  grant(user: string, role: string): Promise<void>;

  /**
   * Revoke a role from a user. Only admins can modify role assignments.
   */
  revoke(user: string, role: string): Promise<void>;

  /**
   * List user-to-role assignments. When user is omitted, returns all assignments visible to the caller.
   */
  listAssignments(user?: string): Promise<RbacAssignment[]>;

  /**
   * Allow a role to call a binding class, method, or method with constrained inputs.
   */
  allow(policy: RbacPolicyInput): Promise<void>;

  /**
   * Deny a role from calling a binding class, method, or method with constrained inputs.
   */
  deny(policy: RbacPolicyInput): Promise<void>;

  /**
   * List role-to-call policies. When role is omitted, returns all policies visible to the caller.
   */
  listPolicies(role?: string): Promise<RbacPolicy[]>;
};

declare const Github: {
  /**
   * Clone a GitHub repository into Artifacts, scoped to the current thread.
   */
  pull(repo: string): Promise<{ path: string }>;

  /**
   * Push an Artifact-backed repository checkout back to GitHub.
   */
  push(path: string): Promise<void>;

  search(query: string): Promise<GithubSearchResult[]>;
  issues: GithubIssues;
  pulls: GithubPullRequests;
  workflows: GithubWorkflows;
};

declare const Container: {
  /**
   * Run a shell command in the thread's sandbox container.
   * Artifact repos are pulled into the container before every command; conflicts are reported on stderr.
   */
  shell(command: string, options?: { cwd?: string; timeoutSeconds?: number }): Promise<ShellResult>;
};
```

## Integrations

* Slack
* Discord (will be implemented later)
* Google Chat (will be implemented later)


## Tool environments and persistence

All tools except `container-shell` run as lightweight dynamic workers. Only when you need heavy tools like ffmpeg is a sandbox container used.

Files are tracked in [Artifacts](https://developers.cloudflare.com/artifacts/). In dynamic workers, they are backed by [memfs](https://github.com/streamich/memfs), and in the container they're backed by [artifact-fs](https://github.com/cloudflare/artifact-fs).

Every file modification by the `File` class (i.e. all the native tools like `edit-file`, `rename-file`, etc.,) is automatically tracked in Artifacts, hence all file modification tools take a required "commit-message" argument. This serves two purposes: Every change gets documented with a human-readable string, and changes are automatically synced to the artifacts, in case memfs is wiped.

Artifacts repos are automatically fetched to the container, _every_ time you run `container-shell`. If there are conflicts, those are written to stderr in the `container-shell` output, in an actionable format that makes it obvious for the agent where the divergence is.

The `container-shell` tool description has language that tells the agent to always commit changes if they make changes to any of the files.

Artifact repos are stored in two folders (and subfolders):
* `/workspace` -- Temporary files, etc.
* `/src/github.com/<repo-owner>/<repo-name>` -- Repos pulled from Github

## Databases

By default, Arness provides the following tables:
* `memory`
* `rbac_assignments`
* `rbac_policies`

The `memory` table has the following fields:
* `created_at` -- autogenerated on create
* `updated_at` -- autogenerated on update
* `title` -- Terse string which will be included in the system prompt, required
* `body` -- Arbitrary text, required
* `tags` -- List of short dash-separated strings, required
* `references` -- List of strings referencing other notes, web URLs, etc.

The `rbac_assignments` table maps users to roles:
* `created_at` -- autogenerated on create
* `updated_at` -- autogenerated on update
* `user` -- Principal identifier, formatted as `<provider>:<provider-user-id>`
* `role` -- Role granted to the principal
* `granted_by` -- Principal identifier for the admin who granted the role

The `rbac_policies` table maps roles to allowed binding calls:
* `created_at` -- autogenerated on create
* `updated_at` -- autogenerated on update
* `role` -- Role the policy applies to
* `effect` -- `allow` or `deny`
* `class` -- Binding class name, such as `File`, `DB`, or `Github`
* `method` -- Optional binding method name. When omitted, the policy applies to the entire class.
* `input` -- Optional JSON object that constrains the allowed method inputs. When omitted, the policy applies to every call matching `class` and `method`.
* `granted_by` -- Principal identifier for the admin who created the policy

## Role-based access control

RBAC is enforced on every Code Mode binding call. Each binding method checks the calling user's roles and the role-to-call policies before executing.

Bootstrap admins are configured in `arness.yaml`, and admins can assign or revoke roles for other users through the `RBAC` binding. The `admin` role can also be granted through `RBAC.grant`, so additional admins are stored in `rbac_assignments` after bootstrap.

Policies can target three levels:
* Entire binding classes, such as allowing `Github` for the `github` role.
* Specific methods, such as allowing `DB.select` for the `user` role.
* Specific method inputs, such as allowing `DB.select` and `DB.insert` for the `user` role only when the target table is `memory`.

Initial roles:
* `admin` -- Can grant and revoke roles, manage policies, and use every binding.
* `user` -- Can use normal agent capabilities such as `File`, `Web`, `Publish`, `Image`, `PDF`, and `DB` according to the deployment policy. By default, `user` can select and insert on the `memory` table, but not on RBAC tables.
* `github` -- Can use the `Github` binding.
* `container` -- Can use the `Container` binding.

## Architecture

Arness is designed to be easily extensible. Extensions may include
* Tools
* Remove Git folders
* Prompts
* Database tables

## CLI

### Deploy

```
arness deploy
```

This will read `arness.yaml` and deploy 

### Run locally in a CLI

```
arness cli
```


## Configuration

arness.yaml:

```yaml
account-id: ${CF_ACCOUNT_ID}
name: arness

admins:
  - slack:<slack-user-id>

access:
  team-domain: ${CF_ACCESS_TEAM_DOMAIN}
  audience: ${CF_ACCESS_AUDIENCE}  # Access Application Audience / AUD tag

ai-gateway:
  model: <llm-model>
  account-id: <ai-gateway-account-id>  # optional, defaults to account-id above
  gateway-id: ${CF_AIG_GATEWAY_ID}

artifacts:
  namespace: arness  # optional

d1:
  database-name: arness  # optional
  database-id: ${CF_D1_DATABASE_ID}

github:
  oauth-client-id: ${GITHUB_OAUTH_CLIENT_ID}
  oauth-client-secret: ${GITHUB_OAUTH_CLIENT_SECRET}
  callback-path: /oauth/github/callback

slack:
  bot-token: ${SLACK_BOT_TOKEN}
  signing-secret: ${SLACK_SIGNING_SECRET}
  app-token: ${SLACK_APP_TOKEN}  # optional; only needed for Socket Mode/local development
```

Env vars can be read from the environment at `arness deploy`-time, or from `.env`.

