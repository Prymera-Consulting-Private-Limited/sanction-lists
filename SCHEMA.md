# Schema

Schema version 1. Name normalisation version 1.

## Files

| File | What it is |
| --- | --- |
| `latest.json` | The current version: `version`, `tag`, `previous_version`, `generated_at`, `manifest_sha256`. |
| `versions.json` | Every version in order, oldest first, each with its delta's path, sha256 and counts. |
| `data/manifest.json` | For one version: per list, the source file it was built from (name, sha256, size, the publisher's "data as of" date), the number of entries, the number of rejected records, the data files with their sha256, and `carried_forward_from` when the list was not rebuilt in this version. `files` maps every data and delta file to its sha256. |
| `data/<CODE>.ndjson` | The full current snapshot of one list, one entry per line, ordered by `id`. |
| `deltas/<version>.ndjson` | What changed in that version, one operation per line, ordered by list then `id`. |

Every file uses LF line endings and every NDJSON file ends with a newline.

## Versions

`YYYY.MM.DD` (UTC), then `YYYY.MM.DD.2`, `.3` for further publishes the same day.
Each version is an annotated git tag `v<version>`. Tags are never moved or deleted,
so the files of any version can be fetched at its tag.

To stay current, a consumer reads `latest.json` and then either:

- loads every `data/<CODE>.ndjson` at the tag (a full load), or
- applies, in order, the `deltas/<version>.ndjson` of every version after the one it
  holds, taking the chain from `versions.json`.

A full load should remove entries the snapshot no longer contains. After applying a
delta chain, the consumer holds exactly what a full load of the last version gives.

Some versions have no delta: the first one (there is nothing to compare it with),
and any whose delta would be too large to publish (every entry changed at once).
Their `delta` is null in `versions.json` and in the manifest, and `changes` still
gives the counts. A chain that crosses such a version cannot be applied; load the
full snapshot instead.

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
