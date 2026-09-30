# check-fmriprep-logs

`check-fmriprep-logs` is a lightweight tool for auditing fMRIPrep logs: point
it at a directory of `stdout`/`stderr` logs and it summarizes which runs
succeeded, failed (and why), are still in progress, or never finished --
as a console table, a CSV, and an HTML report.

## What it does

- Scans fMRIPrep `stdout`/`stderr` log files, flags and categorizes errors,
  and (optionally) checks whether the T1w derivatives a successful run
  claims to have produced actually exist on disk.
- Tells apart a run that's still actively writing to its log from one that
  simply stopped without finishing.
- Produces a console table, a CSV (`fmriprep_run_summary.csv`, one row per
  run), a per-subject rollup (`fmriprep_subject_summary.csv`, latest status
  + attempt count), and an HTML report with clickable log links, a status
  filter, and sortable columns.

## Sample output

```
                                  WASABI fMRIPrep Log Audit
┏━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Subject ┃ Run Date ┃ Study       ┃     Status      ┃ Message                              ┃
┡━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
│ 001651  │ 20260807 │ 1102_MedMap │    ❌ Failed    │ File not found | No T1w images found │
│ 001651  │ 20260810 │ 1102_MedMap │  ✔ Successful   │ successful                           │
│ 002480  │ 20260807 │ 1102_MedMap │    ❌ Failed    │ Job cancelled                        │
│ 003153  │ 20260807 │ —           │    ❌ Failed    │ Container error                      │
│ 003156  │ 20260807 │ 1102_MedMap │  ✔ Successful   │ successful                           │
└─────────┴──────────┴─────────────┴─────────────────┴──────────────────────────────────────┘
```

The CSV carries the same rows (plus `stdout_path`/`stderr_path`/
`derivatives_found` columns); the HTML report adds clickable links to each
log and lets you filter/sort by status.

## Caveats

fMRIPrep logs have some standard content (and you can make your run
scripts add more), but their *naming* is entirely under your control, so
the main challenge is identifying your logs and mapping each one to a
subject and date. The tool tries to help (see "Log discovery" below), but
things go smoothest if your `sbatch` invocation names logs something like:

```
-o ~/logfiles/<project>/${curTime}_${subjid}.txt \
-e ~/logfiles/<project>/${curTime}_${subjid}_err.txt
```

with `curTime=$(date +"%Y%m%d-%H%M%S")`, so dates sort and filter
correctly and subject IDs are unambiguous in the filename.

## Log discovery: matching a file to a subject and date

`sid_pattern` and `date_pattern` are two independent regexes (each with
one named group, `sid` or `date`) searched against each log's *filename*.
They're kept separate, rather than one combined pattern, because naming
conventions vary in more than just their regex: which one comes first,
what separates them, whether the ID has a prefix like `sub-` or `sid`, etc.
Override whichever one doesn't already match your files. The built-in
defaults are:

```
sid_pattern:  sub-(?P<sid>[A-Za-z0-9]+)
date_pattern: (?P<date>\d{8})
```

**If a filename doesn't match `sid_pattern`,** the script doesn't just
give up on it: it checks whether the file's content actually looks like
an fMRIPrep log (a signature check for lines like `Running fMRIPrep
version` or `nipype.workflow`), and if so, tries a short list of common
subject-id conventions (`sub-XXX`, `sidXXX`, `participant XXX`) against
the file's content. This is a best-effort guess, not a substitute for a
correct `sid_pattern` -- run with `--debug` to see when it had to fall
back this way, and set `sid_pattern` explicitly once you know your
convention. Files that don't look like fMRIPrep logs at all, and files
that do but still yield no ID, are reported once each in a summary at the
end rather than one line per file.

### Migrating a config from the original (pre-2026-09) version

The original version had a single `log_pattern` with both named groups in
one regex, e.g. for logs named `20260801-143000_sid1234_....txt`:

```yaml
log_pattern: '(?P<date>\d{8})-\d{6}[_-]?(?i:sid)(?P<sid>\d+).*\.txt$'
```

Split that into the two independent patterns instead:

```yaml
date_pattern: '^(?P<date>\d{8})-\d{6}'
sid_pattern: '(?i:sid)(?P<sid>\d+)'
```

`wasabi.example.yaml` in this repo is a full worked example of this
migration for the original WASABI config.

## Default configuration

The built-in defaults (`DEFAULT_CFG` in the script) are deliberately
project-agnostic -- nothing points at a specific project's paths. With no
`log_root` configured at all, an interactive terminal will ask whether to
use the current directory, let you type a path, or abort; a non-interactive
run (cron, a batch job, or `--no-input`) gets a clear error telling you
what to pass instead. See `wasabi.example.yaml` for a config that
reproduces the original WASABI-specific defaults exactly, if you need
those again.

## Configuration file (YAML / JSON)

A config file overrides any of the defaults; keys you omit keep their
default value. Resolution order is: CLI flags > `--config FILE` > a
`fmriprep_log_check.yaml`/`.yml`/`.json` (or dot-prefixed) file found
automatically in the current directory > built-in defaults.

### Example YAML (`config.yml`)

```yaml
log_root:   /path/to/your/logfiles
deriv_root: /path/to/your/derivatives
study_subdirs:
  - study_a
  - study_b
deriv_patterns:
  - "sub-(?P<sid>\\w+).*_desc-preproc_T1w\\.nii\\.gz$"
html_title: "My fMRIPrep audit"
date_start: "20251101"
date_end:   "20251130"
```

### Equivalent JSON (`config.json`)

```json
{
    "log_root":   "/path/to/your/logfiles",
    "deriv_root": "/path/to/your/derivatives",
    "study_subdirs": ["study_a", "study_b"],
    "deriv_patterns": [
        "sub-(?P<sid>\\w+).*_desc-preproc_T1w\\.nii\\.gz$"
    ],
    "html_title": "My fMRIPrep audit",
    "date_start": "20251101",
    "date_end":   "20251130"
}
```

You may also need `bids_path_map` entries if the same subject ID can show
up under more than one dataset and the study isn't otherwise discoverable
from the log directory or filename -- each entry maps a BIDS root path (as
printed by fMRIPrep in its `* BIDS dataset path: ...` line) to a study
label.

## Derivatives cross-check

If `deriv_root` is set, a run that prints a success marker is also
checked against derivatives actually present on disk (matched via
`deriv_patterns`, indexed once up front rather than per-subject, so this
doesn't get slower the more subjects you have). A "successful" run with no
matching derivative file gets flagged in its message rather than reported
as a clean success. Leave `deriv_root` unset to skip this entirely.

## Command-line options

CLI flags are parsed after any config file and always take priority.

```
-c, --config PATH     Path to a YAML or JSON config file.
--log-root PATH       Directory containing the logs (overrides config).
--log-glob GLOB        Glob for log files, e.g. '*.txt' (default: *.txt).
--recursive            Search log-root recursively for matching files.
--out-dir PATH         Where to write the CSV/HTML report (default: next to log-root).
--start-date DATE      Only consider logs on or after this date (YYYYMMDD).
--end-date DATE        Upper bound for log dates, inclusive (YYYYMMDD).
--include STR          Only process log files whose name contains this string.
--exclude STR          Skip log files whose name contains this string.
--title TITLE          Title used in the HTML report.
--only {all,successful,failed,in-progress,unfinished}
                       Filter the terminal/HTML report to one status (the CSV always has everything).
--no-input             Never prompt interactively; fail with an error if log-root can't be resolved.
-d, --debug            Print verbose debugging information.
```

Example:

```
python check_fmriprep_logs.py --config config.yml --start-date 20251101 --end-date 20251130 --title "November 2025 audit"
```

## A note on error parsing

Both `stdout` and `stderr` are scanned line-by-line against an ordered
catalog of patterns (`ERROR_CATALOG` in the script) -- specific rules
first (FreeSurfer license, SLURM cancellation reason, OOM-kill, disk-full,
etc.), a generic `ERROR` catch-all last. Some rules are based on actual
crash examples; others are speculative, added on the assumption that a
single project's failure history is unlikely to be exhaustive (e.g. no
WASABI run has hit "disk quota exceeded" yet, but it's common enough
elsewhere to be worth flagging if it ever comes up). `ERROR_CATALOG`
itself isn't part of the YAML/JSON configuration -- edit the script to add
project-specific rules -- but known-harmless noise (e.g. a particular
Fontconfig warning) can be suppressed via the configurable
`ignore_patterns` list without touching the catalog.

As a performance note: a run that reports success is detected by checking
just the last ~64KB of its stdout log first, and the full (potentially
huge) file is only read and scanned line-by-line against the catalog when
that quick check comes up empty -- i.e. for runs that failed or are still
in progress.

## A full example

Suppose your logs are under `/data/logs`, derivatives under
`/data/derivatives`, you're using the naming convention described above,
and you only want to audit runs from 1-15 Nov 2025:

```yaml
# config.yml
log_root:   /data/logs
deriv_root: /data/derivatives
study_subdirs:
  - my_study
html_title: "Nov 2025 fMRIPrep audit"
date_start: "20251101"
date_end:   "20251115"
```

```
python check_fmriprep_logs.py -c config.yml --title "Nov 2025 audit"
```

No `--start-date`/`--end-date` on the command line, so the dates from the
config are used; `--title` on the command line overrides the config's
`html_title`.

## History

- Developed for the WASABI project: May 2026
- DBIC public version released: August 2026
- September 2026: generalized -- removed WASABI-specific defaults in favor
  of config discovery and an interactive fallback, split `log_pattern`
  into independent `sid_pattern`/`date_pattern`, expanded the error
  catalog, wired up the derivatives cross-check (previously parsed but
  unused), added the per-subject rollup, content-based log discovery for
  filenames that don't match `sid_pattern`, and a large performance
  improvement for scanning big successful logs.
