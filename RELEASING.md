# Maintainer release procedure

Only reviewed releases belong in this catalog. Package upload credentials are never available
in pull-request CI. The catalog checker does not execute plugin code.

1. Build and doctor the package in its source repository; record the exact source commit.
   Review all package contents and dependency licenses. Pack the prebuilt files into the
   base64 JSON envelope and retain its SHA-256. Do not repack an existing version.
2. In this repository, add `records/<id>/<version>.json` and update `index.json`.
   Keep every historical record. Include source, compatibility and manual acceptance evidence
   in the PR. Changes to reserved identities, origins or workflows require separate review.
3. Validate locally with the trusted checker:
   `node scripts/check-catalog.mjs --root . --artifacts /absolute/path/to/uploads`.
4. Using a maintainer's existing Wrangler installation and Cloudflare login, upload each new
   object. Check that the destination is absent first; if already present, verify its digest
   and reuse it. Never replace a different payload. Substitute the actual immutable object key:

   ```sh
   CLOUDFLARE_ACCOUNT_ID=d77cfbca817ed65e0f033ddb32f3c8a2 wrangler r2 object put \
     studio-plugin-releases/releases/PLUGIN/VERSION/SHA256.json \
     --file /absolute/path/to/package.json --remote \
     --content-type application/json --cache-control 'public, max-age=31536000, immutable'
   ```

5. Run `node scripts/check-catalog.mjs --root . --remote`. This downloads all historical
   artifacts anonymously, enforces size limits and verifies their hashes and manifests.
   Then test clean install, a harmless real tool call and update/data retention in Studio.
6. Merge the reviewed catalog PR only after its `validate` job and manual gates pass.
7. Publish the exact reviewed index last:

   ```sh
   CLOUDFLARE_ACCOUNT_ID=d77cfbca817ed65e0f033ddb32f3c8a2 wrangler r2 object put \
     studio-plugin-releases/catalog/v1/index.json --file index.json --remote \
     --content-type application/json --cache-control 'public, max-age=60, must-revalidate'
   ```

8. Read the public index anonymously and compare it to the reviewed file. Check Studio's
   Refresh action from a clean profile. A failed upload does not authorize changing package
   versions or regenerating assets. Retry only the same reviewed payload after reconciliation.

## Community submissions

A submitter's pull request adds `records/<id>/<version>.json` and its `index.json` entry (written
by Studio's `npm run plugin:submit`) and links the artifact attached to their source repository's
release. CI's "Metadata and history" step must pass; "Artifacts published" fails until step 5.

1. In a checkout of the pull request, stage the asset:
   `node scripts/stage-artifact.mjs --root . --record records/<id>/<version>.json --from <asset URL> --out /absolute/path/to/uploads`.
   It refuses bytes that do not match the record's digest or envelope, and runs nothing.
2. In a Studio checkout, `npm run plugin:unpack -- <staged file> <new directory>`, then compare it
   with the record's `repo` (and `subdir`) at its `sha`. Only authoring files (`AGENTS.md`,
   `jsconfig.json`, `plugin-sdk/`, dotfiles) may be missing; every shipped file must be
   explainable from that source.
3. In an isolated environment without maintainer credentials, run `npm run plugin:doctor` on the
   unpacked folder (it executes the backend) and load it in an owned Studio development profile.
   Check the capabilities, network hosts and evidence against the pull request.
4. Check the release identity is new: a new id, or a higher version from the same publisher,
   repository and subdirectory. The validator enforces this; ownership transfers are separate.
5. Upload the staged object as in step 4 above, then re-run the pull request's checks until
   "Artifacts published" passes.
6. Approve and merge, then publish the index (steps 7 and 8 above).

Repository roles: `@vanyathecyborg` owns review. Require one approving review and the `validate`
status check on main before accepting community submissions. Setup of those repository rules
must be verified in GitHub; CODEOWNERS alone does not enforce them.

Withdrawal removes a discoverable entry but retains records and packages. Installed copies
are not silently removed. Publish a higher corrective version rather than rolling back the
index. No automatic publishing workflow or third-party credential is needed by Studio users.
