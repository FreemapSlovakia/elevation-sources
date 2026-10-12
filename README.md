# elevation-sources

The DTM sources behind Freemap's elevation API, terrain shading
([`terrain-tiles`](https://github.com/FreemapSlovakia/terrain-tiles)) and
terrain pyramid, one directory per dataset.

Deployed to `/fm/storage1/backend.freemap.sk-data/elevation-sources` on fm6,
where it is a checkout of this repository. Edit it here and pull there; a
deploy of [`dem-pyramid`](https://github.com/FreemapSlovakia/dem-pyramid)
refuses to run if that checkout is dirty or behind.

## Layout

```
245-de_by/
  source.json
  source.vrt   -> /fm/storage2/dtm/de_by/all.vrt
  mercator.tif -> /fm/storage2/terrain/de_by.tif
```

One directory per dataset, named `NNN-slug`. Anything without a `source.json`
is ignored, which is how `.git` and this README stay out of the way.

The two symlinks point at rasters on fm6, so they dangle anywhere else:

| Link | Target |
| --- | --- |
| `source.<ext>` | The original raster, the one `file` names; `.vrt` or `.tif` after its target. |
| `mercator.tif` | Where one is built: the dataset in EPSG:3857 on the XYZ pixel grid of its zoom, Int32 heights in 2 mm steps, nodata `-2147483648`, average overviews down to z8 — what `terrain-tiles` serves. A dataset with this link is in `dem-pyramid`'s terrain pyramid too. |

GDAL opens a VRT through the link with its relative sources intact.

### Directory name

| Part | Meaning |
| --- | --- |
| `NNN` | Precedence. The first source covering a point answers for it, so a **lower number outranks a higher one**. `999-gedtm30` is the global fallback and every other source outranks it. `terrain-tiles` orders its sources by it too. |
| `slug` | The dataset's id downstream; see [Adding a dataset](#adding-a-dataset). |

Numbers are spaced so a source can be inserted between two others without
renaming them. They need not be contiguous, but they must be unique.

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
| `name` | string | The **model** this dataset belongs to, as the API reports it: an ISO country code for a national model (`sk`, `cz`, `fr`), the model's own id otherwise (`gedtm30`, `sonny`). Not unique — several datasets share one name, and consumers merge them under it. `es` is three, `sonny` twelve. |
| `file` | string | The raster, **absolute**, exactly as GDAL opens it; `source.<ext>` links the same file. Unique across the repository. A `.vrt` is fine. |
| `attributions` | array | What to display when this dataset is used. Must hold at least one entry — a dataset with none stops `dem-pyramid` from starting, because serving uncredited data is a licence breach. |
| `attributions[].name` | string | The credit line **verbatim** as the licence asks for it, including any required rights-holder wording. Displayed as given; consumers do not reformat it. |
| `attributions[].url` | string, optional | A page to link the credit to. Omit the key entirely when there is nowhere sensible to point. |

Several entries are normal where one dataset merges data from several
providers: `340-sonny-gb` carries eight, one per survey it was compiled from.

## Adding a dataset

1. Add `NNN-slug/source.json` here, with its credit, and the `source.<ext>`
   link to its raster; add `mercator.tif` once it is built for shading. Push.
2. Pull on fm6.
3. Once `mercator.tif` is linked, the dataset is in the pyramid: on the data
   host run `dem-tool refresh`, `bin/index.sh` and restart `terrain`.

**Removing `mercator.tif` from a dataset the pyramid reads needs the same third
step.** `dem-pyramid` refuses to start while its cache still holds a dataset
that no longer links one, since it then has no credit to show for it — so unlink
it here, then run `dem-tool refresh` on the data host. Otherwise the next
restart of a running service is where it surfaces.

This list is the superset. The Sonny datasets have no `mercator.tif`: they are
served by the elevation API only — they bridge the gap between national data and
GEDTM30, and fall back to SRTM where Sonny found no national lidar, which no
field here distinguishes.

The directory name without its number is what identifies a dataset
downstream: `010-sk` becomes `sk`, `090-es-29` becomes `es_29` in the pyramid,
and `terrain-tiles` credits `245-de_by` as `de_by`. Renumbering changes only
the precedence.

## Consumers

- **Freemap's elevation API** (`freemap-v3-api`) — reads the rasters and
  returns these credits.
- **[`terrain-tiles`](https://github.com/FreemapSlovakia/terrain-tiles)** —
  serves elevation and shading tiles from the `mercator.tif` rasters and credits
  them from here.
- **[`dem-pyramid`](https://github.com/FreemapSlovakia/dem-pyramid)** — indexes
  the `mercator.tif` of every dataset that links one, in directory order, and
  reports per render which models answered it, with these credits. It holds no
  source list of its own: each raster's zoom and extent are measured by
  `dem-tool refresh` and cached on the data host.
