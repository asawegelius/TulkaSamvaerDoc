# Adopt or upgrade this template

Use a published release and review adoption in a dedicated project pull request. Release publication and project adoption require separate human decisions. See the [versioning contract](git-strategy/05-versioning-and-tagging.md) and [changelog](CHANGELOG.md).

## Verify the archive before extraction

From the canonical repository’s GitHub Release, choose the named `documents-template-<version>.zip` asset. Resolve the release tag to its full annotated-tag commit in a fresh trusted upstream clone and compare it with the release's approved source commit.

```sh
git fetch --no-tags origin "refs/tags/$tag:refs/tags/$tag"
git cat-file -t "refs/tags/$tag"
git rev-parse "refs/tags/$tag^{commit}"
gh release view "$tag" --repo "$repo" --json tagName,isDraft,isPrerelease,assets
gh release download "$tag" --repo "$repo" --pattern "$asset"
gh release verify "$tag" --repo "$repo"
gh release verify-asset "$tag" "$asset" --repo "$repo"
```

Expect type `tag`, the approved SHA, a published release and the named archive. Use both GitHub CLI verification commands for an immutable release. If immutable release verification is unavailable, obtain the SHA-256 digest from the maintainer's independently approved release record for the same tag, commit and asset name. Compare it to `sha256sum "$asset"` or PowerShell `(Get-FileHash -Algorithm SHA256 -LiteralPath $asset).Hash.ToLowerInvariant()`. An unavailable approved digest, mismatch, or failed verification of an immutable release stops adoption. A checksum bundled with an untrusted download does not establish its identity. [GitHub release integrity](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/verify-release-integrity)

Extract the verified asset into a fresh directory. The default archive contains the core template. An optional capability’s files and activation entry points require a separate approved installation from the pinned upstream distribution. Choose the local customizations and commit the accepted snapshot and its provenance together. Read `distribution.json` directly from the pinned upstream commit when deciding what to copy or upgrade.

## Ownership and provenance

- **Template-managed** files come from the distributed inventory. Review each approved change and record intentional adaptations.
- **Shared** files, including `AGENTS.md`, `README.md` and local conventions, preserve project paths, roles, decisions and wording. Apply only authorized upstream changes. Review the behavior of any changed instructions.
- **Project-owned** files include live requirements, decisions, tests, reviews, and verification evidence. Preserve them during template upgrades. Any necessary project record migration follows the project’s own approved process. A copied review or test becomes a project record.
- **Maintainer-only** files cover release infrastructure, development records and regression fixtures. Keep them outside adopter content. Preserve the relative layout of adopted policy, skill and reference files. Adjust links relative to the final location when creating a copied project record.

Commit `template-provenance.yml` at the project root. Start from the [example](examples/template-provenance.yml). The source is the canonical GitHub repository. Resolve tags to full commits, and list every currently managed destination in `managed_scope` with its upstream path and ownership class. Existing project files outside that scope require explicit reconciliation and retain their bytes during that reconciliation. A project with unverified legacy lineage uses `baseline: null` until reconciled.

`baseline` is the latest completely reconciled release for required changes in the approved managed scope. `exceptions` is keyed by project path. Its `anchor` is the last applied upstream source and can be `present`, verifiably `absent` from that release inventory, or `unknown`. Its `evaluated` release identifies the reviewed target. Record the disposition, reason and approval reference.

| Disposition | Anchor handling |
| --- | --- |
| Applied or adapted | Record the accepted target’s release, SHA and source path. A local adaptation retains that upstream base for future comparisons. |
| Deferred or declined | Preserve the previous effective anchor. Record the target evaluated and the reason or prerequisite. A required deferral prevents baseline advancement. |
| Unknown source | Block a three-way merge. Identify a trustworthy old source or reconcile against an exact release with the project owner. |
| Renamed source | Approve the old-to-new upstream and downstream mapping. Read the old upstream path for the merge base. |
| Deleted source | Review the deletion. Move a retained copy out of managed scope. Record verified absence and review collisions if the path reappears. |

After reviewing every intervening release and dependency, compare each required managed path’s effective source with the target. Add a plan item for every changed source in scope, even when the project already has the target content. Advance `baseline` only when each item is reconciled and anchored to the approved target. During partial adoption, paths outside the selected work retain their effective anchors. Review removal of a still-distributed required path as a separate scope reconciliation before advancing the baseline. Keep unresolved exceptions with their original anchors and preserve historical decisions in PR evidence.

## Optional capabilities

Record each capability in the reviewed release inventory in `capabilities`. The key identifies the upstream capability. Its value contains an explicit state and `configuration` listing the actual project paths that activate that capability. Non-inactive states also identify the selected release and a decision reference.

- `inactive` means the project has no installed members or active configuration.
- `partial` means an approved subset is present. Keep automatic entry points inactive until all mandatory dependencies are ready.
- `deferred` means the capability remains inactive pending a prerequisite or later decision.
- `declined` records an explicit opt-out. Reassess if the capability’s terms or dependencies change.
- `adopted` means the chosen release’s files, prerequisites and approved configuration are present. Its documented triggers now run automatically on applicable work. Human approval gates still apply.

Baseline advancement never changes a capability state by itself. An adoption PR may retain a capability on its previously approved release by keeping the corresponding member anchors. Verify an opted-out or partial configuration, an applicable activation case and an unrelated case using the supported native clients before accepting activation. Human review establishes the behavior of shared instructions and settings.

## Install canonical skills for selected clients

Canonical skill files and their bundled references live under `skills/` in the template source. The optional capability inventory identifies the canonical entry points and any other required files. Select the capability and supported clients in the adoption PR. Copy the skill subtree from the approved source and preserve its relative layout within each selected client’s native discovery root.

In `capabilities.<id>.clients`, record each selected client and its discovery root, for example `codex: .agents/skills` and `claude-code: .claude/skills`. For every installed file, list its actual destination under `managed_scope` with the canonical `source_path`, ownership and `client`. Two destination records may refer to the same canonical source. Record the activated native entry points in `configuration`. Partial adoption can record staged files and selected clients while keeping the native entry points absent.

Keep references and assets beside each installed skill when they use relative paths. Install required shared policies in the approved project location and verify any links that cross the skill directory from each installed copy. Record approved link rewrites or other local adaptations as a per-destination `adapted` exception with the original upstream anchor and `installed_sha256` of the approved committed file. The verifier compares every installed skill file to its exact source blob or approved adapted digest and resolves its Markdown links in the reviewed project commit.

To upgrade, compare the effective source anchor for each installed destination with the new canonical version. Review replacements, moves and deletions in the same plan. Remove obsolete native entry points and associated files when a client is deselected or upstream removes them. Preserve an intentionally forked file as a separately approved project record outside the obsolete discovery location. Verify that the installed tree contains the accepted versions, references and activation paths for every selected client.

## Prepare and verify the upgrade

1. Create a clean task branch in the project. Pin the project base, source URL, target tag and full SHA. Read the project instructions, distribution inventories and release notes for every intervening release. Account for mandatory migrations in order before using a cumulative diff.
2. Make a plan with `schema_version`, `project_base`, `target` (`tag` and `commit`), an approval reference, and `items` keyed by stable item IDs. Each item lists `paths`, an `action` (`apply`, `adapt`, `reconcile`, `defer` or `decline`), `depends_on` item IDs and the intended `verification`. Review the approved managed scope and group dependencies that must operate together.
3. Compare old and new sources with Git. For an eligible text file, extract the project version, the verified upstream base and the target to temporary files. Use `git merge-file -p project base target` and accept output after a zero exit status and review. Review renames, collisions, deletions and binary changes explicitly. An approved result can match the target or the clean merge output. For a manual resolution or other intentional difference, use an `adapted` exception pointing at the approved target and record `installed_sha256` of the committed result, along with the decision and rationale. This also covers an approved decision to keep the current project bytes. For a verified absent source use a reviewed new-file copy with a destination collision check.
4. Commit the accepted content and provenance together. Compare changed paths to the approved plan, check for conflict markers, review links, IDs, relationships and evidence, and run the project’s actual checks. Reset affected execution summaries where old evidence no longer applies. The upstream repository provides a maintainer-side scope and provenance verifier if the reviewer wants to reproduce the structural checks.
5. Attach the plan, commands, exact revisions, observed results and unresolved items to the project PR. Store accepted outputs with their inputs and next prerequisite so the next executor can resume without repeating completed work. The project owner approves any exception, capability activation or merge.

A repeated completed upgrade with the same approved target and scope should produce an empty diff and byte-identical provenance. If recovery is needed after merge, use a reviewed revert or targeted repair. Restore provenance with affected content, preserve later project changes and rerun verification. An adopter can approve a return to a previous verified release when the recovery plan calls for it.
