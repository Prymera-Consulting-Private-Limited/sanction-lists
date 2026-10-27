# Schema

Schema version 1. Name normalisation version 1.

## Files

| File | What it is |
| --- | --- |
| `latest.json` | The current version: `version`, `tag`, `previous_version`, `generated_at`, `checked_at`, `manifest_sha256`, and `sources`, how fresh each list is (see Freshness). |
| `versions.json` | Every version in order, oldest first, each with its delta's path, sha256 and counts. |
| `data/manifest.json` | For one version: per list, the source file it was built from (name, sha256, size, the publisher's "data as of" date), the code that built it (`parser`, see Parser versions), the number of entries (`entities`), how many of them are in force (`listed`: those without `delisted_on`), how many are of each `entity_type` (`entity_types`), how many records the source repeated and were merged into an entry (`merged_duplicates`), the number of rejected records, the data files with their sha256, and `carried_forward_from` when the list was not rebuilt in this version. `files` maps every data and delta file to its sha256. `validation` gives the path, sha256 and size of the version's `data/validation.json`, and `acceptances` who accepted a list a check would have held back, and why (see Validation). Older manifests lack `listed`, `entity_types` and `merged_duplicates`, and so does a list carried forward from one of them. Manifests published before `validation` and `acceptances` existed lack both. |
| `data/validation.json` | How one version was checked before it was published, and every record in the publishers' files that could not be used (see Validation). It belongs to a version only through that version's manifest. |
| `data/<CODE>.ndjson` | The full current snapshot of one list, one entry per line, ordered by `id` compared byte by byte (`UNSC:10` comes before `UNSC:9`). |
| `deltas/<version>.ndjson` | What changed in that version, one operation per line. The lines are grouped by list, in the order the codes are given for `list` under Entry, and ordered by `id` within each list as in a snapshot. That list order is not alphabetical, so the file as a whole need not be in `id` order. A list that did not change has no lines. |
| `latest.json.sig`, `data/manifest.json.sig` | The publisher's signatures of `latest.json` and of the manifest. See Signatures. |

Every file uses LF line endings and every NDJSON file ends with a newline.

## Versions

`YYYY.MM.DD` (UTC), then `YYYY.MM.DD.2`, `.3` for further publishes the same day.
Each version is an annotated git tag `v<version>`. Tags are never moved or deleted,
so the files of any version can be fetched at its tag.

To stay current, a consumer reads `latest.json` and then either:

- loads every `data/<CODE>.ndjson` at the tag (a full load), or
- applies, in order, the `deltas/<version>.ndjson` of every version after the one it
  holds, taking the chain from `versions.json`.

`versions.json` is not signed. Before applying a chain, check it against the signed
manifests: each version's manifest, fetched and verified at its own tag, must name
the version before it as `previous_version`. If any does not, load the full snapshot
of the latest version instead.

A full load should remove entries the snapshot no longer contains. After applying a
delta chain, the consumer holds exactly what a full load of the last version gives.
From the first version whose `SCHEMA.md` carries this sentence, the publisher checks
this before it publishes a version with a delta: applied to the data files of the
version before, the delta must give every one of the version's data files byte for
byte, or the version is not published. Earlier versions were not checked when they
were published. The deltas of 2026.09.25 to 2026.10.20 were checked afterwards, and
each gives its version's data files.

A version is published only when something changed, or when a person accepted a
list a check would have held back (see Acceptances). `checked_at` in `latest.json`
is when every source was last read, whether or not that produced a version: on a
day when nothing changed it moves forward and the version stays the same. A consumer
can tell a quiet week from a dataset nobody is maintaining by its age. Versions
published before `checked_at` existed lack it; read `generated_at` in its place.

## Freshness

A dataset checked today can still hold a list whose publisher stopped updating it.
`sources` in `latest.json` says, per list code:

| Field | What it is |
| --- | --- |
| `checked_at` | When the list was last read. A partial publish that did not read it leaves the earlier value. |
| `data_as_of` | The publisher's own date for its file, as last read (`YYYY-MM-DD`), or null when the file carried none. It can move on a day with no new version, when a publisher re-dates an unchanged file. |
| `last_changed_in` | The version that last changed the list's entries. |
| `max_data_age_days` | How old `data_as_of` may get before the list should count as stale. Null for a list whose publisher dates nothing. |
| `max_unchanged_days` | For a list with no date: how long it may go unchanged before a warning. Undated lists are often quiet for long stretches, so this is a warning, not a fault. |

The limits are the publisher's judgement of each list's usual pace. The block
carries no verdict: a status worked out at publish time would itself be stale by
the time you read it, so judge each list against your own clock. DFAT's date is
when its workbook was saved, because its sheet title often carries none.

A version is never published from a file dated earlier than the one it replaces,
or from one that lost the date the last one had, unless a person has checked and
accepted it. That version's manifest then says who accepted it and why, in
`acceptances` (see Acceptances). Versions published before `sources` existed lack
it.

Some versions have no delta: the first one (there is nothing to compare it with),
and any whose delta would be too large to publish (every entry changed at once).
Their `delta` is null in `versions.json` and in the manifest, and `changes` still
gives the counts. A chain that crosses such a version cannot be applied; load the
full snapshot instead.

A delta can also be empty (0 bytes), in a version that changed no entry: one that
changed only how many records a list rejected, or one a person accepted with no
entry changed. Its `changes` are all 0, and applying it changes nothing.

## Parser versions

`parser` in a list's block in the manifest names the code that built the list's
entries from its source file: `name`, the parser's name, and `version`, a number
that starts at 1 and goes up whenever what that parser makes of the same file
changes. Both OFAC lists are built by one parser and share its version. Each
`name` counts its own versions, so a list moved to another parser can show the
same `version` under a different `name`. When a list's entries change between
two versions and its `parser` changed as well, in its `name` or its `version`,
some of those changes may have come from the code rather than from the source
file. The reverse does not hold: an unchanged `parser` does not prove that a
change came from the source. The publisher's tests hold each version to what the
parser makes of samples of each source, not of every record, so a code change
that alters only records outside those samples, in the parser or in code or
reference data the parsers share, can reach the entries with no version moving.
And the number follows the code, not the machine it runs on: a different ICU
there can change `normalized` names with no version moving.

It is information only. Do not refuse a version, or load it differently, because
of its `parser`. A parser change reaches you as ordinary changes to the entries,
in the delta like any other.

A list carried forward keeps the block it was built with, so its `parser` names
the code that built its entries, which can be older than the code that built the
other lists in the same version. Manifests published before `parser` existed lack
it, and so does a list carried forward from one of them. A version is published
only when something changed or a person accepted a list, and one can rebuild some
lists and carry the rest forward, so a new parser version that changes nothing in
the output first appears in the next version that rebuilds that list.

## Validation

Each version since `validation` appeared in the manifest publishes
`data/validation.json`, which says how the version was checked before it was
published and which records in the publishers' files could not be used:

```json
{
    "checks": [
        {"list": "UNSC", "check": "delta_replay", "outcome": "passed", "figures": {"against": "<the version before>"}},
        {"list": "UNSC", "check": "rejections", "outcome": "passed", "figures": {"rejected": 0, "previous": 0}}
    ],
    "excluded": [
        {"list": "SECO", "locator": "target ssid=…", "reason": "no listing record on this occurrence; treated as a stale copy"}
    ],
    "kept_with_warning": [
        {"list": "THEUKLIST", "locator": "Designation …", "reason": "unreadable date of birth \"…\", kept as raw text"}
    ]
}
```

The file belongs to a version only through that version's manifest: its
`validation` block gives the file's `path`, `sha256` and `bytes`, and the file is
that version's record only when its sha256 matches. The file stays in the
repository from one version to the next, so a `data/validation.json` can sit
beside a manifest with no `validation` block. It is then another version's
record, not that one's, and says nothing about it.

### Checks

`checks` lists every check the publisher ran, in the order it ran them: first
`delta_replay` for every list, in the order the codes are given for `list` under
Entry; then, for each list rebuilt in this version that the version before also
carried, in the same order, `rejections`, `removals` and `data_as_of`. A list
carried forward was not read, so the change checks do not run for it, and the
first version has nothing to compare with, so it records no checks at all.

| `check` | `figures` | It would hold the list back when |
| --- | --- | --- |
| `delta_replay` | `against`: the version the delta applies to. | Never: a version whose delta, applied to the data files of the version before, does not give every one of its data files byte for byte is not published, and no person can accept it. It is recorded as passed even when the delta was then left out for being too large, because it was replayed before it was dropped. |
| `rejections` | `rejected` and `previous`: how many records were rejected, whether left out or kept with a warning (see Rejected records), in this version and in the version before. | `rejected` is greater than `previous`. |
| `removals` | `removed`, `previous_entities` (the list's entries in the version before) and `max_share_basis_points`, the largest share it may lose at once in hundredths of a percent (200 is 2%). | `removed` is more than that share of `previous_entities`. |
| `data_as_of` | `data_as_of`, the publisher's date for this file, and `previous`, the date it is compared with: the last one read, which can be newer than the version before's when a daily check saw a re-dated file. Each is `YYYY-MM-DD` or null. | `data_as_of` is earlier than `previous`, or is null where `previous` was a full date. |

| `outcome` | Meaning |
| --- | --- |
| `passed` | The check found nothing to hold the list back for. |
| `accepted` | The check would have held the list back, and a person checked the list and accepted it as it is (see Acceptances). |
| `refused` | The check held the list back. No published version carries it: only the record a trial run builds, and does not publish, can. |

Every number is an integer and the record names no time, version being
published or person. So the same source files, checked against the same commit
of this repository (the version before, with any daily checks since) and with
the same lists rebuilt and accepted, give the same bytes.

### Rejected records

`excluded` and `kept_with_warning` hold every record in a publisher's file that
could not be used, each as `list`, `locator` (where it is in the publisher's file,
in a form that depends on the publisher) and `reason`, by list in the order the
codes are given for `list` under Entry, then in the order the parser read them.
For a list rebuilt in this version the two hold as many records as its
`rejections` in the manifest.

- `excluded`: this record of the publisher's file was not used. That is not
  always the same as the entry being absent from the data. SECO repeats some
  targets, and a copy with no listing record is excluded as a stale copy while
  the target is published from its other occurrence. A DFAT row naming a further
  name whose reference does not continue the entry above it is excluded, and the
  entry it may belong to is published without that name.
- `kept_with_warning`: the entry was published, but a detail of this record
  could not be read and is kept only as raw text, such as a UK date of birth with
  no year.

A list carried forward was not read, so its records are the ones the version
before recorded for it, copied once that version's `data/validation.json` has
matched its manifest. When the version before has no `validation` block, there
is nothing to copy: a list carried forward has no records here, nor in any later
version that carries it forward, until one rebuilds it. That is the case for a
list carried forward since a version published before `validation` existed,
whose `carried_forward_from` manifest has no `validation` block. Its
`rejections` in the manifest still stands.

### Acceptances

`acceptances` in the manifest has one object for each list a person accepted,
and is empty when nobody accepted anything:

| Field | What it is |
| --- | --- |
| `list` | The list code. |
| `checks` | The checks the acceptance overrode, by their names under Checks, in the order they ran. Only a check that would have held the list back is named, and a list none of whose checks would have is not listed. |
| `reason` | Why the person judged the list right as it is: one line in Unicode NFC, with line breaks and other separators and control characters replaced by spaces and characters shown as nothing (format characters, variation selectors and fillers) removed, of at most 200 characters (the current limit, which can change). |
| `accepted_by` | The GitHub login of the person who accepted. In a workflow run, whoever started that attempt of the run, which on a re-run is the person who re-ran it rather than the person who first started it. |
| `run_url` | The workflow run that published the version: an internal run reference, which readers outside the publisher may not be able to open. Null when the version was published outside a workflow run. |

The version's commit message carries the same, one line per list. A version is
published whenever a person accepted a list, even when no entry changed: its delta
is then empty.

## Signatures

From the first version published after 2026-09-26, `latest.json` and
`data/manifest.json` each have a detached Ed25519 signature beside them, in a file
of the same name ending `.sig`:

```json
{
    "algorithm": "ed25519",
    "key_id": "<first 16 hex characters of the sha256 of the raw 32-byte public key>",
    "signature": "<base64 of the 64-byte signature>"
}
```

The signature is over the file's exact bytes as fetched, with no canonicalisation.
The manifest gives the sha256 of every data and delta file in `files`, and of
`data/validation.json` in `validation`, so one verified manifest vouches for the
whole version. To verify:

1. Fetch the file and its `.sig` from the same ref (the tag for a manifest, `main`
   for `latest.json`).
2. Look up `key_id` among the public keys listed in `README.md`. Refuse an unknown key.
3. Verify the signature over the bytes you fetched, and only then parse them. For
   example, in PHP: `sodium_crypto_sign_verify_detached($signature, $bytes, $publicKey)`;
   in Python with PyNaCl: `VerifyKey(public_key).verify(bytes, signature)`.
4. Check that the manifest's `version` is the version you asked for. For the latest
   version, also check that the manifest's sha256 is the `manifest_sha256` in
   `latest.json`. Then check each file you download against the manifest's sha256:
   a data or delta file against its entry in `files`, and `data/validation.json`
   against `validation.sha256`.
5. Refuse a file with no `.sig` beside it, unless you load a version published
   before signing began. Every version since 2026.09.29 is signed.

`latest.json` is signed again every day, when `checked_at` moves. Versions published
before signing began have no `.sig` files.

## Entry

```json
{"addresses":[],"content_hash":"…","dates_of_birth":[],"delisted_on":null,"entity_type":"entity","gender":null,"id":"OFAC-SDN:36","identifications":[],"last_updated_on":null,"list":"OFAC-SDN","listed_on":"1986-12-10","measures":["Block","SDN List"],"names":[{"language":null,"normalized":"aerocaribbean airlines","publisher_reference":"19574","script":"Latin","strength":null,"type":"primary","value":"AEROCARIBBEAN AIRLINES"}],"nationalities":[],"places_of_birth":[],"positions":[],"programmes":["CUBA"],"publisher_reference":"36","references":[],"remarks":null,"source":{"list":"OFAC-SDN","url":"https://sanctionssearch.ofac.treas.gov/Details.aspx?id=36"},"titles":[]}
```

Every key is always present. Lists may be empty; scalars may be null.

| Key | Type | Meaning |
| --- | --- | --- |
| `id` | string | `<list code>:<publisher_reference>`. Stable across versions. |
| `list` | string | One of `UNSC`, `SECO`, `OFAC-SDN`, `OFAC-CONSOLIDATED`, `THEUKLIST`, `EU-FINANCIAL-SANCTIONS`, `DFAT`, `CSEMA`. |
| `entity_type` | string | `individual`, `entity`, `vessel` or `aircraft`. |
| `publisher_reference` | string | The publisher's own identifier (see below). |
| `names` | list | `value`; `type` (`primary`, `alias`, `former`, `variation`, `original_script`); `strength` (`strong`, `good`, `weak` or null, the publisher's own view of how reliable an alias is); `script` and `language` as the publisher states them; `normalized` (below); `publisher_reference`. |
| `dates_of_birth` | list | `raw` as printed; `year`, `month`, `day` when readable; `approximate`; `from`/`to` (`YYYY-MM-DD`) for a stated range. |
| `places_of_birth` | list | `raw`, `city`, `region`, `country`, `country_iso2`. |
| `nationalities` | list | `raw`, `iso2`. Nationality and citizenship both. |
| `addresses` | list | `raw`, `street`, `city`, `region`, `postal_code`, `country`, `country_iso2`. |
| `identifications` | list | `type` (the publisher's label, lower-cased), `number`, `country`, `country_iso2`, `issued_on`, `expires_on`, `note`. |
| `programmes` | list of strings | Sanctions programmes or regimes. |
| `measures` | list of strings | Measures imposed, where the publisher states them. |
| `gender` | string or null | `male` or `female`, where stated. |
| `titles`, `positions` | lists of strings | Honorifics and roles held. |
| `remarks` | string or null | The publisher's free-text notes. |
| `references` | list | `scheme`, `value`: the same designation's key on another list (`un_reference_number`, `eu_reference_number`, `ofsi_group_id`, `ofac_identity_id`, `seco_foreign_identifier`). |
| `listed_on`, `delisted_on`, `last_updated_on` | `YYYY-MM-DD` or null | As the publisher states them. An entry with `delisted_on` is no longer in force. |
| `source` | object | `list`, and `url` of the entry or of the list. |
| `content_hash` | string | sha256 of the entry's canonical form without `content_hash`. |

`iso2` codes are ISO 3166-1 alpha-2 (plus `XK` for Kosovo) and are null when the
publisher's country could not be matched; `raw` always keeps what they printed.

### Publisher references

OFAC: `entity@id`. UN: `DATAID`. EU: `logicalId`. UK: `UniqueID`. SECO: `ssid`.
DFAT: the numeric `Reference`. Canada publishes no identifier, so its reference is
derived: `<country>/<schedule or ->/<item>`, each part slugged. A renumbering by the
publisher appears as a removal and an addition.

### Canonical form and hash

Object keys are sorted; strings are Unicode NFC; lists are de-duplicated and
ordered by the canonical encoding of their elements, with `primary` names first;
encoding is JSON without whitespace, with Unicode and slashes unescaped. Every
line in a data file is the canonical form of its entry. To verify an entry,
remove `content_hash`, re-encode canonically and compare the sha256.

### Normalised names

`normalized` is: Unicode NFC → ICU `Any-Latin; Latin-ASCII` transliteration →
lower case → apostrophes and the ayn/hamza marks removed → every run of
punctuation or symbols replaced by a space → whitespace collapsed. Titles are not
removed. It is meant for matching, not display.

## Delta line

```json
{"entity":{…},"id":"UNSC:6907993","list":"UNSC","op":"changed","previous_content_hash":"…"}
```

`op` is `added`, `changed` or `removed`. `entity` is the entry's full new state,
or null when removed. `previous_content_hash` is null when added.

An `added` entry is one the previous version did not hold. A `changed` or
`removed` entry is one it did, and `previous_content_hash` is the `content_hash`
it had there. An `id` has at most one line in a delta.
