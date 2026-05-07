# Arness

Arness is Andreas Jansson's minimal agent harness built on the Cloudflare Agents SDK. It is named after Andreas' favorite ice cream.

## Code mode bindings

Arness exposes capabilities to agents as Code Mode bindings: typed TypeScript classes that the model can call from generated JavaScript. Instead of loading one JSON schema per operation into context, Code Mode gives the model a compact typed API surface and runs the generated code in an isolated Dynamic Worker sandbox.

```ts
declare const File: {
  read(path: string, options?: { startLine?: number; endLine?: number }): Promise<string>;
  writeNew(path: string, contents: string, commitMessage: string): Promise<void>;
  replace(path: string, contents: string, commitMessage: string): Promise<void>;
  edit(
    path: string,
    edit:
      | { originalContent: string; newContent: string }
      | { startContent: string; endContent: string; newContent: string },
    commitMessage: string,
  ): Promise<void>;
  find(pattern: string, options?: { cwd?: string }): Promise<string[]>;
  grep(pattern: string, options?: { cwd?: string; include?: string; exclude?: string }): Promise<string>;
  rename(oldPath: string, newPath: string, commitMessage: string): Promise<void>;
  delete(path: string, commitMessage: string): Promise<void>;
};

declare const Web: {
  search(query: string, options?: { limit?: number }): Promise<WebSearchResult[]>;
  fetch(url: string, options?: { markdown?: boolean }): Promise<string>;
  request(url: string, init?: RequestInit): Promise<Response>;
};

declare const Publish: {
  file(path: string): Promise<{ url: string }>;
};

declare const Image: {
  read(path: string): Promise<ImageContent>;
};

declare const PDF: {
  read(path: string): Promise<PdfContent>;
};

declare const DB: {
  select<T = unknown>(sql: string, params?: unknown[]): Promise<T[]>;
  insert(table: string, row: Record<string, unknown>): Promise<void>;
  createTable(sql: string): Promise<void>;
};

declare const Github: {
  pull(repo: string): Promise<{ path: string }>;
  push(path: string): Promise<void>;
  search(query: string): Promise<GithubSearchResult[]>;
  issues: GithubIssues;
  pulls: GithubPullRequests;
  workflows: GithubWorkflows;
};

declare const Container: {
  shell(command: string, options?: { cwd?: string; timeoutSeconds?: number }): Promise<ShellResult>;
};

declare const Code: {
  run<T = unknown>(code: string): Promise<T>;
};
```

These bindings back the user-facing tools:

* `File`: `read-file`, `write-new-file`, `replace-file`, `edit-file`, `find-file`, `grep`, `rename-file`, `delete-file`
* `Web`: `web-search`, `web-fetch`, and arbitrary HTTP requests
* `Publish`: `publish-file`
* `Image`: `read-image`
* `PDF`: `read-pdf`
* `DB`: `db-select`, `db-insert`, `db-create-table`
* `Github`: GitHub repo, search, issue, PR, and workflow operations
* `Container`: `container-shell`
* `Code`: `code-mode`

## Integrations

* Slack
* Discord



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
* `rbac`

The `memory` table has 

## Role-based access control

## Architecture

Arness is designed to be easily extensible. Extensions may include
* Tools
* Remove Git folders
* Prompts
* Database tables
