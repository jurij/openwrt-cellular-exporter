# Issue tracker: GitHub

Issues, specifications and decision maps live in
`jurij/openwrt-cellular-exporter` on GitHub. Use the `gh` CLI.
Confirm the repository from `git remote -v` and use an explicit `--repo`
when outside its clone. Never post private environment notes or raw telemetry.

## Conventions

- Create: `gh issue create --repo jurij/openwrt-cellular-exporter --title "..." --body-file <file>`.
- Read: `gh issue view <number> --repo jurij/openwrt-cellular-exporter --json number,title,body,labels,comments,assignees`.
- List: `gh issue list --repo jurij/openwrt-cellular-exporter --state open --json number,title,labels,assignees`.
- Comment: `gh issue comment <number> --repo jurij/openwrt-cellular-exporter --body-file <file>`.
- Label: `gh issue edit <number> --repo jurij/openwrt-cellular-exporter --add-label <label>`.
- Close: `gh issue close <number> --repo jurij/openwrt-cellular-exporter`.

Use pagination when claiming a complete issue inventory. Read back the exact
target after writes. An API success alone is not verification.

When a skill says "publish to the issue tracker", create a GitHub issue.
When it says "fetch the relevant ticket", read its full body and comments.

## Pull requests as a triage surface

**PRs as a request surface: no.**

External pull requests are not automatically treated as feature requests.
GitHub issues and pull requests share a number space; determine the object
type rather than assuming a bare number identifies an issue.

## Wayfinding operations

Wayfinder plans decisions before implementation. Its canonical map is one
issue labelled `wayfinder:map`. Decision tickets are GitHub sub-issues and
carry a type label: `wayfinder:research`, `wayfinder:prototype`,
`wayfinder:grilling` or `wayfinder:task`. Do not add triage labels to them.
Create missing Wayfinder labels when charting the map.

Refer to maps and decisions by linked title, not bare issue number.

### Parent-child relationship

Obtain the child's numeric database ID:

`gh api repos/jurij/openwrt-cellular-exporter/issues/<child-number> --jq .id`

Attach it to the map:

`gh api --method POST repos/jurij/openwrt-cellular-exporter/issues/<map-number>/sub_issues -F sub_issue_id=<child-database-id>`

Read children using:

`gh api --paginate repos/jurij/openwrt-cellular-exporter/issues/<map-number>/sub_issues`

If native sub-issues are demonstrably unavailable, document that limitation,
use a child task list in the map and `Part of #<map-number>` in each child.
Never use an invented placeholder issue number.

### Blocking relationships

Use native issue dependencies. The blocker database ID is its `.id`, not its
issue number or GraphQL node ID.

`gh api --method POST repos/jurij/openwrt-cellular-exporter/issues/<child-number>/dependencies/blocked_by -F issue_id=<blocker-database-id>`

Read back the child's dependency list:

`gh api --paginate repos/jurij/openwrt-cellular-exporter/issues/<child-number>/dependencies/blocked_by`

Only if native dependencies are demonstrably unavailable, document the
limitation and use `Blocked by: #<number>` references in the issue body.

### Frontier and claims

The frontier is the map's open, unassigned children with no open blockers.
Query the actual sub-issues and dependency states; do not infer readiness
from label names. Choose the first available child in map order.

Claim before work:

`gh issue edit <number> --repo jurij/openwrt-cellular-exporter --add-assignee @me`

### Resolution

Post the decision as a resolution comment, close the ticket and add a short
linked pointer under the map's "Decisions so far". Keep detailed reasoning
in the decision ticket rather than duplicating it in the map.

Research evidence must distinguish documented behavior, runtime observations
and unverified assumptions. Wayfinder completion is not implementation or
production-deployment approval. After Wayfinder, use the agreed specification
and ticket flow before implementation.
