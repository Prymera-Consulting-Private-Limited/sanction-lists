# Sanction lists

[![Freshness check](https://github.com/Prymera-Consulting-Private-Limited/sanction-lists/actions/workflows/freshness.yml/badge.svg)](https://github.com/Prymera-Consulting-Private-Limited/sanction-lists/actions/workflows/freshness.yml)
[![Last checked](https://img.shields.io/github/last-commit/Prymera-Consulting-Private-Limited/sanction-lists/main?label=last%20checked)](https://github.com/Prymera-Consulting-Private-Limited/sanction-lists/commits/main)
[![Latest version](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fraw.githubusercontent.com%2FPrymera-Consulting-Private-Limited%2Fsanction-lists%2Fmain%2Flatest.json&query=%24.version&label=latest%20version&color=blue)](latest.json)
[![Signed with Ed25519](https://img.shields.io/badge/signed-Ed25519-blue)](#signatures)
[![Licence](https://img.shields.io/github/license/Prymera-Consulting-Private-Limited/sanction-lists?label=licence)](LICENSE)

Eight official sanctions lists (US OFAC SDN and Consolidated, UN Security
Council, EU, UK, Switzerland, Australia and Canada) in one normalised JSON
format, checked every night, with a new signed version whenever a list changes.
For compliance teams and developers who screen customers and payments.

Published by Remitso, under CC BY 4.0 (see [Licence](#licence)).

This data is republished from public government sources solely for
anti-money-laundering and sanctions-compliance purposes. It is not the
authoritative source: before acting on an entry, consult the publisher listed
below. Remitso makes no warranty that this copy is complete or current.

## Quick start

Every list is one file, `data/<CODE>.ndjson`, one entry per line, with the codes
under [Sources](#sources). Each example below reads the current version and finds
the entries on the UN list with a name containing "yazd".

**cURL** (with [jq](https://jqlang.org)):

```sh
BASE=https://raw.githubusercontent.com/Prymera-Consulting-Private-Limited/sanction-lists

# The current version, and when the lists were last read
curl -s $BASE/main/latest.json | jq '{version, tag, checked_at}'

# Read the list at the version's tag, which never moves
TAG=$(curl -s $BASE/main/latest.json | jq -r .tag)
curl -s $BASE/$TAG/data/UNSC.ndjson \
  | jq -c 'select(any(.names[]; .normalized | contains("yazd"))) | {id, entity_type, name: .names[0].value}'
```

**Python** (3.8 or later, nothing to install):

```python
import json
import urllib.request

BASE = "https://raw.githubusercontent.com/Prymera-Consulting-Private-Limited/sanction-lists"


def fetch(ref, path):
    with urllib.request.urlopen(f"{BASE}/{ref}/{path}") as response:
        return response.read()


latest = json.loads(fetch("main", "latest.json"))
print("version", latest["version"], "checked at", latest["checked_at"])

query = "yazd"  # lower-case ASCII, like the normalized names
for line in fetch(latest["tag"], "data/UNSC.ndjson").splitlines():
    entry = json.loads(line)
    if any(query in name["normalized"] for name in entry["names"]):
        print(entry["id"], entry["entity_type"], entry["names"][0]["value"])
```

**Node.js** (18 or later, nothing to install; save as `quickstart.mjs` and run
`node quickstart.mjs`):

```js
const BASE = "https://raw.githubusercontent.com/Prymera-Consulting-Private-Limited/sanction-lists";

async function fetchText(ref, path) {
  const response = await fetch(`${BASE}/${ref}/${path}`);
  if (!response.ok) throw new Error(`${path}: HTTP ${response.status}`);
  return response.text();
}

const latest = JSON.parse(await fetchText("main", "latest.json"));
console.log("version", latest.version, "checked at", latest.checked_at);

const query = "yazd"; // lower-case ASCII, like the normalized names
for (const line of (await fetchText(latest.tag, "data/UNSC.ndjson")).split("\n")) {
  if (!line) continue;
  const entry = JSON.parse(line);
  if (entry.names.some((name) => name.normalized.includes(query))) {
    console.log(entry.id, entry.entity_type, entry.names[0].value);
  }
}
```

A substring match is enough to explore the data, but screening needs proper name
matching. `names[].normalized` is built for that: `SCHEMA.md`, under Normalised
names, says how, so you can normalise your own names the same way.

### Verify before you rely on it

The examples above trust whatever GitHub serves. Before screening with the data,
check the publisher's signature: verify `latest.json`, then the manifest it names,
then each file you read against the manifest's sha256. One verified manifest
vouches for every file in its version. If a check fails within a few minutes of a
publish, try again: GitHub caches each file for up to five minutes, so a file and
its signature can briefly disagree.

**Python** (`pip install pynacl`):

```python
import base64
import hashlib
import json
import urllib.request

from nacl.signing import VerifyKey

BASE = "https://raw.githubusercontent.com/Prymera-Consulting-Private-Limited/sanction-lists"

# The publisher's public keys, as listed under Signatures below.
KEYS = {
    "08e25c1704f9f816": "yG9Ko8VIceGlAVQw6CDdByDkA5SfbYumLbEy5puAnGY=",
}


def fetch(ref, path):
    with urllib.request.urlopen(f"{BASE}/{ref}/{path}") as response:
        return response.read()


def verified(ref, path):
    data = fetch(ref, path)
    signature = json.loads(fetch(ref, path + ".sig"))
    if signature["algorithm"] != "ed25519" or signature["key_id"] not in KEYS:
        raise ValueError(f"{path} is signed with an unknown key: {signature['key_id']}")
    # Raises nacl.exceptions.BadSignatureError if a single byte differs.
    VerifyKey(base64.b64decode(KEYS[signature["key_id"]])).verify(data, base64.b64decode(signature["signature"]))
    return data


latest = json.loads(verified("main", "latest.json"))
manifest_bytes = verified(latest["tag"], "data/manifest.json")
manifest = json.loads(manifest_bytes)
if manifest["version"] != latest["version"] or hashlib.sha256(manifest_bytes).hexdigest() != latest["manifest_sha256"]:
    raise ValueError("The manifest is not the one latest.json names.")

data = fetch(latest["tag"], "data/UNSC.ndjson")
if hashlib.sha256(data).hexdigest() != manifest["files"]["data/UNSC.ndjson"]:
    raise ValueError("data/UNSC.ndjson does not match the signed manifest.")
print("verified", latest["version"], "and data/UNSC.ndjson")
```

<details>
<summary><strong>Node.js</strong> (18 or later, nothing to install)</summary>

```js
import { createHash, createPublicKey, verify } from "node:crypto";

const BASE = "https://raw.githubusercontent.com/Prymera-Consulting-Private-Limited/sanction-lists";

// The publisher's public keys, as listed under Signatures below.
const KEYS = {
  "08e25c1704f9f816": "yG9Ko8VIceGlAVQw6CDdByDkA5SfbYumLbEy5puAnGY=",
};

async function fetchBytes(ref, path) {
  const response = await fetch(`${BASE}/${ref}/${path}`);
  if (!response.ok) throw new Error(`${path}: HTTP ${response.status}`);
  return Buffer.from(await response.arrayBuffer());
}

async function verified(ref, path) {
  const data = await fetchBytes(ref, path);
  const signature = JSON.parse((await fetchBytes(ref, `${path}.sig`)).toString());
  const key = KEYS[signature.key_id];
  if (signature.algorithm !== "ed25519" || !key) throw new Error(`${path} is signed with an unknown key: ${signature.key_id}`);
  const publicKey = createPublicKey({ key: { kty: "OKP", crv: "Ed25519", x: Buffer.from(key, "base64").toString("base64url") }, format: "jwk" });
  if (!verify(null, data, publicKey, Buffer.from(signature.signature, "base64"))) throw new Error(`${path}: the signature does not match.`);
  return data;
}

const sha256 = (bytes) => createHash("sha256").update(bytes).digest("hex");

const latest = JSON.parse((await verified("main", "latest.json")).toString());
const manifestBytes = await verified(latest.tag, "data/manifest.json");
const manifest = JSON.parse(manifestBytes.toString());
if (manifest.version !== latest.version || sha256(manifestBytes) !== latest.manifest_sha256) {
  throw new Error("The manifest is not the one latest.json names.");
}

const data = await fetchBytes(latest.tag, "data/UNSC.ndjson");
if (sha256(data) !== manifest.files["data/UNSC.ndjson"]) throw new Error("data/UNSC.ndjson does not match the signed manifest.");
console.log("verified", latest.version, "and data/UNSC.ndjson");
```

</details>

## How often it updates

Every list is read every night. The run is scheduled for 22:00 UTC; GitHub
usually starts it later, and its commit lands between midnight and 03:00 UTC.

- A night on which any list changed publishes a new version.
- A night on which nothing changed still records the check: `checked_at` in
  `latest.json` moves forward and the version stays the same. A quiet week can
  be told from a dataset nobody is maintaining.
- A night whose run fails publishes nothing, and the "last checked" badge above
  stops moving.

The "freshness check" badge comes from a check run here every day at 06:00 UTC
(`.github/workflows/freshness.yml`). It fails when `latest.json` or the manifest
is not signed with a key listed under [Signatures](#signatures), when
`latest.json` does not name the manifest beside it, or when the lists were last
read more than 24 hours earlier.

`sources` in `latest.json` says, for each list, when it was last read, the
publisher's own date for its file, and the version that last changed it.
`SCHEMA.md`, under Freshness, says how to judge each list against your own clock.

## Sources

| Code | List | Publisher |
| --- | --- | --- |
| `OFAC-SDN` | [Specially Designated Nationals (SDN) List](https://sanctionslist.ofac.treas.gov/Home/SdnList) | US Department of the Treasury, Office of Foreign Assets Control |
| `OFAC-CONSOLIDATED` | [Consolidated (non-SDN) List](https://sanctionslist.ofac.treas.gov/Home/ConsolidatedList) | US Department of the Treasury, Office of Foreign Assets Control |
| `UNSC` | [UN Security Council Consolidated List](https://main.un.org/securitycouncil/en/content/un-sc-consolidated-list) | United Nations Security Council |
| `EU-FINANCIAL-SANCTIONS` | [EU Financial Sanctions](https://data.europa.eu/data/datasets/consolidated-list-of-persons-groups-and-entities-subject-to-eu-financial-sanctions?locale=en) | European Commission, DG FISMA |
| `THEUKLIST` | [The UK Sanctions List](https://www.gov.uk/government/publications/the-uk-sanctions-list) | Foreign, Commonwealth & Development Office |
| `SECO` | [Swiss Sanctions/Embargoes](https://www.seco.admin.ch/seco/en/home/Aussenwirtschaftspolitik_Wirtschaftliche_Zusammenarbeit/Wirtschaftsbeziehungen/exportkontrollen-und-sanktionen/sanktionen-embargos.html) | State Secretariat for Economic Affairs |
| `DFAT` | [Australian Consolidated List](https://www.dfat.gov.au/international-relations/security/sanctions/consolidated-list) | Department of Foreign Affairs and Trade |
| `CSEMA` | [Canada Special Economic Measures Act](https://www.international.gc.ca/world-monde/international_relations-relations_internationales/sanctions/consolidated-consolide.aspx?lang=eng) | Global Affairs Canada |

`SOURCES.md` records, for every version, the exact source file each list was
built from (its sha256 and the publisher's own "data as of" date).

## Using the data

- `latest.json` names the current version and its git tag.
- `data/<CODE>.ndjson` is the full current snapshot of one list, one entry per
  line. `data/manifest.json` gives the sha256 of every file.
- `deltas/<version>.ndjson` lists what changed in that version: each line is
  `added`, `changed` (with the entry's full new state) or `removed`.
- `versions.json` lists every version in order, so a consumer that is several
  versions behind can apply the deltas in sequence.
- `data/validation.json` records the checks run on a version before it was
  published, whether each passed or was accepted, and every record in a
  publisher's file that could not be used. The
  manifest vouches for it, and says who accepted any list a check would have
  held back, and why. `SCHEMA.md`, under Validation, describes both.
- Every version is tagged `v<version>` and tags are never moved, so any past
  snapshot stays reachable.

Versions are dated `YYYY.MM.DD`, with `.2`, `.3` for a second or third publish
on the same day. The record format is described in `SCHEMA.md`.

## Signatures

`latest.json` and `data/manifest.json` are signed with Ed25519, in `.sig` files
beside them. `SCHEMA.md`, under Signatures, says how to verify one. These are the
keys the publisher signs with; refuse a signature whose `key_id` is not here.

| Key id | Public key (base64 of the raw 32 bytes) |
| --- | --- |
| `08e25c1704f9f816` | `yG9Ko8VIceGlAVQw6CDdByDkA5SfbYumLbEy5puAnGY=` |

## Licence

Remitso's compilation and normalisation are licensed under
[CC BY 4.0](LICENSE): attribute "Remitso sanction lists" and the original
publishers above. The underlying source data remains subject to each
publisher's own terms.

This repository is written only by Remitso's publishing process, apart from the
freshness check's workflow. Please report a problem by opening an issue rather
than a pull request.
