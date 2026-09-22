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

Repository roles: `@vanyathecyborg` owns review. Require one approving review and the `validate`
status check on main before accepting community submissions. Setup of those repository rules
must be verified in GitHub; CODEOWNERS alone does not enforce them.

Withdrawal removes a discoverable entry but retains records and packages. Installed copies
are not silently removed. Publish a higher corrective version rather than rolling back the
index. No automatic publishing workflow or third-party credential is needed by Studio users.
