# Arness

Arness is Andreas Jansson's minimal agent harness built on the Cloudflare Agents SDK. It is named after Andreas' favorite ice cream.

## Tools

TODO: Rewrite as ts types that code-mode can use

* `read-file` (read full file or specific lines)
* `write-new-file` (throws error if file exists, avoids the footgun of accidentally overwriting a file)
* `replace-file` (throws error if file doesn't exist)
* `edit-file` (token-efficient editing using either originalContent or startContent/endContent)
* `find-file` (like the command line find command)
* `grep` (efficient grep in the artifacts repos stored in memfs)
* `rename-file`
* `delete-file`
* `web-search` (using exa)
* `web-fetch` (has option to automatically convert to markdown)
* `publish-file` (expose a file to a stable URL on a web server, the web server automatically renders markdown as html)
* `read-image` (reads an image into the context)
* `read-pdf` (reads a PDF into the context)
* `db-select` (run sql against a D1 database, databases are shared across threads)
* `db-insert`
* `db-create-table`
* `github` (pull and push repos; search; list, open, close issues; list, open, close PRs; list, read, log workflows; etc. -- basically everything that the gh-cli can do. pulling a repo creates a copy in Artifacts that's scoped to the thread)
* `code-mode` (spawn dynamic worker and execute javascript)
* `container-shell` (run command in sandbox container, containers are scoped to the thread)

`code-mode` has bindings to the following classes:
* `File` (used by `read-file`, `edit-file`, `find-file`, `grep`, etc.)
* `Web` (used by `web-search`, `web-fetch`, also has methods to do arbitrary HTTP calls)
* `Publish` (used by `publish-file`)
* `DB` (used by `db-select`, etc.)
* `Container` (used by `container-shell`)
* `Github` (used by `github`)

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
