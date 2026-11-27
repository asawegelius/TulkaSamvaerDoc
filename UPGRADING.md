# Adopt or upgrade this template

Use a published release and review adoption in a dedicated project pull request. Release publication and project adoption require separate human decisions. Start with the source verification path that matches the content you will use. See the [changelog](CHANGELOG.md) for changes to shared instructions and obligations.

## Verify a published archive before using it

Use this path when starting from the ZIP or consuming it during an upgrade. Resolve the annotated tag and approved commit using the [Git identity checks](#verify-git-snapshots-for-adoption-and-upgrades). Download the named `documents-template-<version>.zip` from that release.

```sh
gh release download "$tag" --repo "$repo" --pattern "$asset"
gh release verify "$tag" --repo "$repo"
gh release verify-asset "$tag" "$asset" --repo "$repo"
```

For immutable releases, run both GitHub verification commands and use the digest for the named asset from the verified signed release attestation. For a release that legitimately lacks native immutable verification, use the SHA-256 in the maintainer's independently approved release record for the same tag, commit and asset name. Compare the download using `sha256sum "$asset"` or PowerShell `(Get-FileHash -Algorithm SHA256 -LiteralPath $asset).Hash.ToLowerInvariant()` before extraction. A failed expected attestation, unavailable authorized download, missing trusted digest or digest mismatch stops archive-based adoption. A locally reconstructed ZIP is evidence of byte equivalence only. [GitHub release integrity](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/verify-release-integrity)

Extract the verified asset into a fresh directory and check its inventory, dependencies and links. The archive contains the core template. Optional capability files and activation entry points require a separate approved installation from the pinned upstream distribution. Choose the local customizations and commit the accepted snapshot and provenance together.

## Verify Git snapshots for adoption and upgrades

Use a fresh, trusted upstream clone. Confirm its origin and the published, non-prerelease identity through authenticated release metadata when the repository is private. Fetch the exact release tag and compare the resolved full commit to the approved source identity.

```sh
git remote get-url origin
git fetch --no-tags origin "refs/tags/$tag:refs/tags/$tag"
git cat-file -t "refs/tags/$tag"
git rev-parse "refs/tags/$tag^{commit}"
git show "$commit:distribution.json"
```

Expect an annotated `tag` and the approved full commit. Read the inventory and source files from the committed snapshot. Resolve historical release anchors the same way. Existing-project adoption and upgrades using only Git objects use this gate without downloading or attesting a ZIP. Follow the archive gate whenever a step uses the asset. Use a tested Git and verification environment, and record actual versions in the evidence.

## Template compatibility

Template versions follow [Semantic Versioning](https://semver.org/). The contract covers distributed paths and formats, stable IDs, required metadata and relationships, validation and adoption interfaces, and mandatory policy, agent behavior, workflow, human approval and evidence obligations. A patch preserves published obligations, a minor release adds compatible capabilities or deprecations, and a major release can require migration. Review the notes for every intervening release. Optional capabilities require explicit adoption and their required triggers apply once activated. This contract governs the upstream template. Your project's own release policy is independent.

## Ownership and provenance

- **Template-managed** files come from the distributed inventory. Review each approved change and record intentional adaptations.
- **Shared** files, including `AGENTS.md`, `README.md` and local conventions, preserve project paths, roles, decisions and wording. Apply only authorized upstream changes. Review the behavior of any changed instructions.
- **Project-owned** files include live requirements, decisions, tests, reviews, and verification evidence. Preserve them during template upgrades. Any necessary project record migration follows the project’s own approved process. A copied review or test becomes a project record.
- **Maintainer-only** files cover release infrastructure, development records and regression fixtures. Keep them outside adopter content. Preserve the relative layout of adopted policy, skill and reference files. Adjust links relative to the final location when creating a copied project record.

Commit `template-provenance.yml` at the project root. See the [legacy](examples/template-provenance-legacy.yml) and [adapted](examples/template-provenance.yml) examples. The source is the canonical GitHub repository. Resolve tags to full commits, and list every currently managed destination in `managed_scope` with its upstream path and ownership class. Existing project files outside the scope require explicit reconciliation and retain their bytes and mode during that transition. A project with unverified legacy lineage uses `baseline: null` until reconciled.

`baseline` is the latest completely reconciled release for required changes in the approved managed scope. `exceptions` is keyed by project path. Its `anchor` is the last applied upstream source and can be `present`, verifiably `absent` from that release inventory, or `unknown`. Its `evaluated` release identifies the reviewed target. Record the disposition, reason and approval reference.

| Disposition | Anchor handling |
| --- | --- |
| Applied or adapted | Record the accepted target’s release, SHA and source path. A local adaptation retains that upstream base for future comparisons. |
| Deferred or declined | Preserve the previous effective anchor. Record the target evaluated and the reason or prerequisite. A required deferral prevents baseline advancement. |
| Unknown source | Use `reconcile` and an approved tagged target. Record evidence such as an exact untagged legacy tree match outside the provenance anchor. A tentative merge using that evidence may inform review. Results differing from the target require an approved `adapted` exception with the target anchor and the installed digest. |
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

1. Create a clean task branch in the project. Pin the project base, source URL, target tag and full SHA. Read the project instructions, distribution inventories and release notes for every intervening release. Account for mandatory migrations in order before using a cumulative diff. For a project without provenance, follow the [legacy initialization sequence](#initialize-a-legacy-project).
2. Make a plan with `schema_version`, `project_base`, `target` (`tag` and `commit`), an `approval` locator, and `items` keyed by stable item IDs. Each item lists `paths`, an `action` (`apply`, `adapt`, `reconcile`, `defer` or `decline`), `depends_on` item IDs and the intended `verification`. Allocate the stable decision locator before drafting. It gains authority only through the separate authenticated approval. Review the managed scope and group dependencies that must operate together.
3. Compare old and new sources with Git. For an eligible text file, extract the project version, the verified upstream base and the target to temporary files. Use `git merge-file -p project base target` and accept output after a zero exit status and review. Review renames, collisions, deletions and binary changes explicitly. An approved result can match the target or the clean merge output. For a manual resolution or other intentional difference, use an `adapted` exception pointing at the approved target and record `installed_sha256` of the committed result, along with the decision and rationale. This also covers an approved decision to keep the current project bytes. For a verified absent source use a reviewed new-file copy with a destination collision check.
4. Prepare candidate commits in authorized scratch work. Commit the accepted content and provenance together for each transition. Compare changed paths to the plan, check for conflict markers, review links, IDs, relationships and evidence, and run the project's actual checks. Existing project records remain outside a template-only change. If earlier execution evidence no longer applies, handle any reset through the project's approved record-change process.
5. Freeze the plans and exact result, record project approval, and verify the [review endpoints and evidence](#approve-verify-and-retain-evidence). The project owner approves scope, adaptations, capability activation and integration.

A repeated completed upgrade with the same approved target and scope should produce an empty diff and byte-identical provenance. If recovery is needed after merge, use a reviewed revert or targeted repair. Restore provenance with affected content, preserve later project changes and rerun verification. An adopter can approve a return to a previous verified release when the recovery plan calls for it.

## Initialize a legacy project

1. **Initialize.** Create the first commit from the original base. Add only `template-provenance.yml` with `baseline: null`, empty exceptions, and the explicitly approved existing core mappings at their canonical destinations. Exact mapping and ownership come from the target inventory. Leave all files and modes unchanged. Use a `reconcile` plan item for every enrolled path. Each effective anchor is unknown. Record optional capability state where applicable without activating or installing anything in this step.
2. **Reconcile.** Make the next commit a direct child. Reconcile each enrolled existing path against the tagged distribution. Use the approved target bytes or record the approved adapted bytes and digest, including when keeping local content. Add newly approved distributed files separately in the plan after checking for destination collisions. Include plan items for required unknown paths even when existing bytes already match the target. Advance the complete baseline only once every applicable required change is reconciled.
3. **Verify the range.** Supply separate plans and pin each before and after commit. Preserve the original base, exact initialization and reconciliation commits, and reviewed head. A flattened plan cannot verify the full legacy adoption. A merge commit preserving reviewed transitions is a straightforward way to keep evidence replayable. If the project uses squash or rebase, agree how the reviewed objects will remain accessible and verify the integrated tree against the approved result.

## Approve, verify and retain evidence

For a complete adoption range, create a frozen JSON run record with `record_type: run`, `schema_version: 1`, canonical `project` and `upstream` URLs, exact `base`, `head`, `target` tag and commit, stable `decision` locator, and ordered `transitions`. Each transition records `stage` (`initialize` or `reconcile`), exact `before` and `after`, `target`, SHA-256 of the exact plan bytes as `plan_sha256`, and complete resulting `managed_scope` and `capabilities`. Its `adaptations` maps each path with an installed digest to its `installed_sha256` and `decision`. Use empty maps where applicable.

Freeze these bytes before approval. The accountable human uses authenticated project review to approve the exact run SHA-256, project, base, head and target. Keep that decision separate from the run so approval does not change either plans or provenance. For a local binding check, provide a JSON record with `record_type: approval`, `schema_version: 1`, the same `project`, `base`, `head`, `target` and `decision`, the approved `run_sha256`, and the authenticated review permalink in `review`. Review any change to the base, head, plans, mappings, adaptations or capability decisions again before integration.

Run the verifier from a trusted checkout of the template repository containing the verification code. Supply the current integration target and reviewed head from the project's authenticated hosting service, fetched to exact local refs immediately before integration. Use an authoritative PR or equivalent review endpoint for the head and the actual target branch for the base. Confirm the refs correspond to that review and remain current when accepting it.

For a GitHub PR, confirm its target branch and PR number in the authenticated review. In the project checkout, fetch those two refs and use the resulting local refs as the verifier inputs.

```sh
git -C "$project" fetch --no-tags origin \
  "refs/heads/$target_branch:refs/doc011/integration-base" \
  "refs/pull/$pr_number/head:refs/doc011/reviewed-head"
actual_target_ref=refs/doc011/integration-base
actual_review_head_ref=refs/doc011/reviewed-head
```

```sh
python -m maintenance.adoption_run --project "$project" --upstream "$upstream" \
  --base-ref "$actual_target_ref" --head-ref "$actual_review_head_ref" \
  --run "$run" --plan "$plan1" --plan "$plan2" --approval "$approval"
```

Omit `--approval` for a diagnostic. The JSON reports `scope: contiguous_run` and either `verification: diagnostic_verified` or `verification: approval_binding_verified`. It always reports `authorized_acceptance: external_governance_required` and identifies the supplied Git refs as the endpoint evidence. A matching supplied record proves the binding only. The project's authenticated decision and integration controls establish ref freshness, reviewer identity, authority, current approval and permitted acceptance. For an ordinary single transition, the existing `python -m maintenance.verify_adoption --project "$project" --upstream "$upstream" --plan "$plan" --result "$head"` remains available. Add `--json` for a `single_transition` report. It does not verify a complete multi-transition range.

Keep the decision and evidence index in the project PR or equivalent durable review record. Retain exact plan and run bytes, the approval, commands, tool versions, changed paths, outputs, and tested commits in project-controlled durable storage. Retain a durable copy of any needed expiring CI output. Link the evidence from the review and identify the next pending prerequisite. The adoption diff contains approved template content and provenance.
