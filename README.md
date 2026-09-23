# Sanction lists

International sanction lists, normalised into one JSON format and versioned
daily, published by Remitso.

This data is republished from public government sources solely for
anti-money-laundering and sanctions-compliance purposes. It is not the
authoritative source: before acting on an entry, consult the publisher listed
below. Remitso makes no warranty that this copy is complete or current.

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
- Every version is tagged `v<version>` and tags are never moved, so any past
  snapshot stays reachable.

Versions are dated `YYYY.MM.DD`, with `.2`, `.3` for a second or third publish
on the same day. The record format is described in `SCHEMA.md`.

## Licence

Remitso's compilation and normalisation are licensed under
[CC BY 4.0](LICENSE): attribute "Remitso sanction lists" and the original
publishers above. The underlying source data remains subject to each
publisher's own terms.

This repository is written only by Remitso's publishing process. Please report
a problem by opening an issue rather than a pull request.
