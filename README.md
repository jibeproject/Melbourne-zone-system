# Melbourne-zone-system

This code is used to update the JIBE Melbourne input zonal data file
[zonesystem.csv](https://gitlab.lrz.de/ga78fel/melbourne/-/blob/5304cf021d3e1bda4969b55be5f14e3c9d89312b/input/zoneSystem.csv)
as uploaded to GitLab on 14 February 2025 with additional linkage attributes.

This [issue](https://github.com/jibeproject/silo/issues/1) has been addressed as per markdown code/log here:
https://github.com/jibeproject/Melbourne-zone-system/blob/master/update-zonesystem.csv-add-urban-rural-and-sa1_7dig.md

The code is also stored in the JIBE working group folder here, along with the required input data retrieved through running the `R` `.qmd` code, and the historical data file `zoneSystem - Qin - 2025-02-13.csv`.

The updated ouptut file `zoneSystem.csv`was commited to the Melbourne gitlab repo:
https://gitlab.lrz.de/ga78fel/melbourne/-/commit/e025eaaeeafb06d7a72b8d06e7383d2b353d9e87

The Quarto markdown document `update-zonesystem.csv-add-linkage-attributes.qmd` was rendered as Git-Flavoured Markdown and Word document formats by running

```{bash}
quarto render update-zonesystem.csv-add-linkage-attributes.qmd
```