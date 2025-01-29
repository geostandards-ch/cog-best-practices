# About
This document summarizes **best practice on publishing cloud optimized data to BGDI**

# Choose your case
```mermaid
graph TD
  start[*Start*: Generate<br> cloud-optimized GeoTIFF] --> purpose{What's the main purpose?}

  purpose -->|View Data| view{Data Type?}
  purpose -->|Analyze Data| analyze{limited loss acceptable? e.g. cm instead of µm}

  view -->|8-bit<br>RGB/RGBa/gray| view8{Transparency Required?}
  view -->|16-bit<br>RGB/RGBa/gray| view16{Transparency Required?}
  view -->|Multi-Band| analyze

  view8 -->|Yes| jpeg-mask[Use JPEG compression with mask<br>-co COMPRESS=JPEG<br>-mask none]
  view8 -->|No| jpeg[Use JPEG compression<br>-co COMPRESS=JPEG]

  view16 -->|No| jpeg[Use JPEG compression<br>-co COMPRESS=JPEG]
  view16 -->|Yes| lerc_lossy[Use LERC_ZSTD compression<br>-co COMPRESS=LERC_ZSTD]

  analyze -->|Yes| lerc_lossy[Use LERC_ZSTD compression<br>-co COMPRESS=LERC_ZSTD]
  analyze -->|No| lerc_lossless[Use LERC_ZSTD compression<br>-co COMPRESS=LERC_ZSTD]

  click jpeg "https://github.com/geostandards-ch/cog-best-practices#lossy-visual-image"
  click jpeg-mask "https://github.com/geostandards-ch/cog-best-practices#lossy-visual-image"
  click lerc_lossy https://github.com/geostandards-ch/cog-best-practices#lossy-numerical-raster"
  click lerc_lossless "https://github.com/geostandards-ch/cog-best-practices#lossless-raster"
```


# Raster
Apply the optimization steps according to the use case and publish cloud-optimized GeoTIFF [COG](https://cogeo.org) using a recent version of [gdal](https://gdal.org). Windows users may use the OSGeo4WShell provided by [QGIS](https://qgis.org).

## lossless raster

Apply optimization steps for raster data
1. prepare input data at a reasonable precision (i.e. `cm` instead of `µm` or `nm` i.e. in the data source)
2. compress your data using
    ```
    gdal_translate -a_srs EPSG:2056 -of COG -co COMPRESS=LERC_ZSTD -co LEVEL=22 -co NUM_THREADS=ALL_CPUS -co BIGTIFF=YES -co STATISTICS=YES ... -ot <datatype> <input.tif> <output.tif>
    ```
    while
    - choosing the appropriate scale and offset for the data if feasible (i.e. scale `cm` instead of `m` using [-a_scale](https://gdal.org/en/stable/programs/gdal_translate.html#cmdoption-gdal_translate-a_scale)) and / or offset (using [-a_offset](https://gdal.org/en/stable/programs/gdal_translate.html#cmdoption-gdal_translate-a_offset)) to better fit the numbers
      - rescaling the data on the fly if not preprocessed already using [-scale](https://gdal.org/en/stable/programs/gdal_translate.html#cmdoption-gdal_translate-scale)
    - choosing to the most appropriate data type ([Byte/Int8/Int16/UInt16/UInt32/Int32/Float32](https://gdal.org/en/stable/programs/gdal_translate.html#cmdoption-gdal_translate-ot)) using the least amount of bits possible.

#### _Notes on `LERC_ZSTD` (lossy/lossless) compression_
- _`MAX_Z_ERROR` defaults to `0` (lossless)_
- _supported since [GDAL](https://gdal.org) v2.4+_ thus also in QGIS, etc.
- _supported by [geotiff.js](https://geotiffjs.github.io/) v2.1+_ thus also in openlayers, etc.
- _has been [developed and published by ESRI](https://github.com/esri/lerc/) and is supported in their products_
- _supported in STAC-browser v3.4_

## lossy visual image

1. compress your data using
    ```
    gdal_translate -a_srs EPSG:2056 -of COG -co COMPRESS=JPEG -co NUM_THREADS=ALL_CPUS -co BIGTIFF=YES -co STATISTICS=YES -co QUALITY=70 ... <input.tif> <output.tif>
    ```

## lossy numerical raster

1. compress your data using
    ```
    gdal_translate -a_srs EPSG:2056 -of COG -co COMPRESS=LERC_ZSTD -co LEVEL=22 -co NUM_THREADS=ALL_CPUS -co BIGTIFF=YES -co STATISTICS=YES -co MAX_Z_ERROR=<threshold> ... -ot <datatype> <input.tif> <output.tif>
    ```
    while
    - choosing a [threshold](https://gdal.org/en/stable/drivers/raster/cog.html#general-creation-options:~:text=MAX_Z_ERROR) (limited error tolerance i.e. `0.01` for `cm`) for better lossy compression ratio
