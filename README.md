# elevation-sources

The DTM sources behind Freemap's elevation API, one directory per dataset.

Deployed to `/fm/storage1/backend.freemap.sk-data/elevation-sources` on fm6.

## Layout

A directory per dataset, named `NNN-slug`, holding a single `source.json`.

The numeric prefix orders the sources: the first one covering a point answers
for it, so a lower prefix outranks a higher one. `999-gedtm30` is the global
fallback and every other source outranks it. The slug is for readers only —
nothing parses it.

## source.json

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

- **`name`** — the model, as the API reports it: a country code for a national
  model, the model's own id otherwise. Several datasets may share one name;
  `es` is three, `fr` seven and `sonny` fourteen. Consumers merge them.
- **`file`** — the raster, absolute, as GDAL opens it.
- **`attributions`** — what to display for this dataset: `name` is the credit
  line verbatim as the licence asks for it, and `url` a page to link to where
  there is one. A list, because one dataset can carry several credits — the
  German Sonny mosaic carries seventeen, one per Land.

Every dataset needs at least one attribution. `dem-pyramid` refuses to serve a
model with none, since a silently uncredited source is a licence breach.

## Consumers

- Freemap's elevation API, for reads and for the credits it returns.
- [`dem-pyramid`](https://github.com/FreemapSlovakia/dem-pyramid), which reads
  the credit lines at startup and reports them per render in `meta.sources`.
  It ingests a subset of these sources; its own build metadata — projection,
  resolution, nodata, resampling — lives in that repo's `sources.yaml`.
