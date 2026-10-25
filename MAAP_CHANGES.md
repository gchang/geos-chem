# Changes from upstream

Upstream: https://github.com/geoschem/geos-chem, tag `14.8.0` at commit a4551f9442183bb572b23e9c2d362e2d34420d3a (2026-09-15), by the Developers of GEOS-Chem, HEMCO, and Related Software (see `AUTHORS.txt`), MIT License (`LICENSE.txt`). This is the GEOS-Chem science codebase that GEOS-Chem Classic 14.8.0 pins as its `src/GEOS-Chem` submodule.

This copy starts from upstream's file tree at a4551f9 without its git history. It is unchanged except for the items below; no model source code (Fortran) was changed. Left out: `.github/` (issue templates and CI workflows). The `KPP/standalone` submodule still points to geoschem/KPP-Standalone at 6cd8fdb, as upstream.

It is used as the `src/GEOS-Chem` submodule of https://github.com/gchang/geos-chem-classic, whose `MAAP_CHANGES.md` describes the whole bundle.

## 2026-10-25: download_data.py builds the right restart URL for the HTTP S3 portals

For the `geoschem+http` portal (https://geos-chem.s3-us-west-2.amazonaws.com) the bucket has no `ExtData/` folder, so `extract_pathnames_from_log` in `run/shared/download_data.py` already appends `/ExtData` to the local prefix. `get_remote_restart_filename` then added a second `ExtData`, and `create_download_script` cut the path after the first one, so the restart request went to `https://geos-chem.s3-us-west-2.amazonaws.com/ExtData/GEOSCHEM_RESTARTS/GC_14.8.0/...` and got 404 Not Found. The run directory was left without its initial conditions file. `nested+http` is set up the same way.

`get_remote_restart_filename` now adds `ExtData` only when the local prefix does not already end in it. The restart now comes from `https://geos-chem.s3-us-west-2.amazonaws.com/GEOSCHEM_RESTARTS/GC_14.8.0/...` and is copied to `Restarts/` as intended. The `geoschem+aws`, `nested+aws` and `rochester` portals get the same paths as before.

## 2026-10-25: download_data.py exits with an error when a download fails

`download_the_data` already raises an error when the generated download script returns a non-zero status, but the script had no error checking and its last command (the `CHEM_INPUTS` mkdir check) always succeeds, so a failed download (such as the 404 above) was only visible in the wget output, and `download_data.py` exited 0. The generated script now keeps going after a failed command, as before, so the other files still download, but records the failure with an `ERR` trap and exits with status 1 at the end; `download_data.py` then stops with `RuntimeError: Error downloading data from <portal>` (exit status 1). Successful downloads are unchanged.

Note: if a dry run lists a file that the portal does not have and the model does not really need, `download_data.py` now reports it as an error where it used to say nothing.

## 2026-10-25: first-time user registration is skipped when input is not a terminal

`createRunDir.sh` (GEOS-Chem Classic and GCHP) calls `registerNewUser` in `run/shared/newUserRegistration.sh` until `GC_USER_REGISTERED` is set. When the menu answers are piped in, as the integration tests do (`printf "7\n1\n1\n..." | ./createRunDir.sh`), the registration questions read those answers instead, post them to https://gc-dashboard.org/registration as name, email, institution and so on, and leave the run directory menus with the wrong answers. With no input at all (for example a batch job), `userInput` loops forever, because `read` keeps returning an empty line at end of input.

`registerNewUser` now asks the questions only when standard input is a terminal. Otherwise it prints `Skipping first-time user registration (input is not a terminal).` and continues without registering or setting `GC_USER_REGISTERED`, so the questions come up again the next time `createRunDir.sh` is run interactively. Interactive runs are unchanged, and so is the existing opt-out (`export GC_USER_REGISTERED=true` in `~/.geoschem/config` or the environment).

## Not changed

On a machine whose kernel name contains `aws`, `createRunDir.sh` copies the sample restart with `aws s3 cp --request-payer requester` from `s3://gcgrid` (`run/GCClassic/setupForRestarts.sh`), which uses whatever AWS credentials the machine has and fails without them. That is left as it is; `download_data.py` (with the fix above) fetches the same restart from the public bucket.

## Check

Done on 2026-10-25 as part of the GEOS-Chem Classic bundle (details in its `MAAP_CHANGES.md`), with 4x5 MERRA-2 TransportTracers run directories:

- `printf '7\n1\n1\n1\n<dir>\n<name>\nn\n' | ./createRunDir.sh` with no `GC_USER_REGISTERED` set printed the skip note, created the run directory, and contacted no registration server.
- `python download_data.py log.dryrun geoschem+http` fetched all inputs and `https://geos-chem.s3-us-west-2.amazonaws.com/GEOSCHEM_RESTARTS/GC_14.8.0/GEOSChem.Restart.TransportTracers.20190101_0000z.nc4` (md5 3dfb1ff7c89e7b45f727dea4c2fd7330), copied it to `Restarts/GEOSChem.Restart.20190101_0000z.nc4`, and exited 0. Upstream 14.8.0 requested `.../ExtData/GEOSCHEM_RESTARTS/...`, got 404 and still exited 0.
- With one extra file in the dry-run log that the portal does not have, this copy exits 1 (`RuntimeError: Error downloading data from geoschem+http`) after downloading everything else; upstream exits 0.
- The model output is identical to upstream 14.8.0 (no Fortran code changed here).
