# elevation-sources

The DTM sources behind Freemap's elevation API, one directory per dataset.

Deployed to `/fm/storage1/backend.freemap.sk-data/elevation-sources` on fm6,
where it is a checkout of this repository. Edit it here and pull there; a
deploy of [`dem-pyramid`](https://github.com/FreemapSlovakia/dem-pyramid)
refuses to run if that checkout is dirty or behind.

## Layout

```
010-sk/source.json
020-cz/source.json
...
999-gedtm30/source.json
```

One directory per dataset, named `NNN-slug`. Anything without a `source.json`
is ignored, which is how `.git` and this README stay out of the way.

### Directory name

| Part | Meaning |
| --- | --- |
| `NNN` | Precedence. The first source covering a point answers for it, so a **lower number outranks a higher one**. `999-gedtm30` is the global fallback and every other source outranks it. |
| `slug` | For readers only. Nothing parses it. |

Numbers are spaced so a source can be inserted between two others without
renaming them. They need not be contiguous, but they must be unique, and the
relative order must match the `priority` column in `dem-pyramid`'s
`sources.yaml` for the datasets that appear in both — `dem-tool check`
compares the two and fails on disagreement.

## source.json

Every field below is required unless marked optional. No other keys are read.

```json
{
  "name": "sk",
  "file": "/fm/storage1/dtm/dmr5_jtsk03.tif",
  "attributions": [
    {
      "name": "DMR 5.0: ÚGKK SR",
      "url": "https://www.skgeodesy.sk/gku/produkty-sluzby/na-stiahnutie/zbgis.html#lls"
    }
  ]
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `name` | string | The **model** this dataset belongs to, as the API reports it: an ISO country code for a national model (`sk`, `cz`, `fr`), the model's own id otherwise (`gedtm30`, `sonny`). Not unique — several datasets share one name, and consumers merge them under it. `es` is three, `fr` seven, `sonny` fourteen. |
| `file` | string | The raster, **absolute**, exactly as GDAL opens it. Unique across the repository: this, not `name`, is what identifies a dataset. A `.vrt` is fine. |
| `attributions` | array | What to display when this dataset is used. Must hold at least one entry — a dataset with none stops `dem-pyramid` from starting, because serving uncredited data is a licence breach. |
| `attributions[].name` | string | The credit line **verbatim** as the licence asks for it, including any required rights-holder wording. Displayed as given; consumers do not reformat it. |
| `attributions[].url` | string, optional | A page to link the credit to. Omit the key entirely when there is nowhere sensible to point. |
| `pyramid` | bool, optional | `true` if `dem-pyramid` builds this dataset into its terrain pyramid. Omit it otherwise. Everything the pyramid needs beyond this — resolution, extent, nodata, resampling — it measures from the raster; this flag is the one judgement it cannot make, since no header says whether a dataset is worth ingesting. |

Several entries are normal where one dataset merges data from several
providers: `250-sonny-de` carries seventeen, one per Land, and `240-be` two,
for Flanders and Wallonia.

## Adding a dataset

1. Add `NNN-slug/source.json` here, with its credit, and push.
2. Pull on fm6.
3. If the pyramid should build from it, add `"pyramid": true` and run
   `dem-tool refresh` on the data host.

Step 3 is optional: this list is the superset. The Sonny datasets are served by
the elevation API but not built into the pyramid — they bridge the gap between
national data and GEDTM30, and fall back to SRTM where Sonny found no national
lidar, which no field here distinguishes.

The directory name is what identifies a dataset downstream: `010-sk` becomes
`sk`, `090-es-29` becomes `es_29`. Renaming one renames the pyramid's storage
for it, so pick the name once.

## Consumers

- **Freemap's elevation API** — reads the rasters and returns these credits.
- **[`dem-pyramid`](https://github.com/FreemapSlovakia/dem-pyramid)** — reads
  the credits at startup and reports, per render, which models answered it.
  Its own build metadata — projection, resolution, nodata, resampling, pyramid
  levels — is not here; it lives in that repository's `sources.yaml`.
