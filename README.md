# Melbourne-zone-system

This code is used to update the JIBE Melbourne input zonal data file
[zonesystem.csv](https://gitlab.lrz.de/ga78fel/melbourne/-/blob/5304cf021d3e1bda4969b55be5f14e3c9d89312b/input/zoneSystem.csv)
as uploaded to GitLab on 14 February 2025 with urban and rural
classification using Australian Bureau of Statistics [Sections of
State](https://www.abs.gov.au/ausstats/abs@.nsf/Lookup/by%20Subject/1270.0.55.004~July%202016~Main%20Features~Section%20of%20State%20(SOS)%20and%20Section%20of%20State%20Range%20(SOSR)~4)
classifications from the Australian Statistical Geography Standard
(Volume 4) for 2016. This data is available in zipped
[CSV](https://www.abs.gov.au/ausstats/subscriber.nsf/log?openagent&1270055004_sa1_ucl_sosr_sos_2016_aust_csv.zip&1270.0.55.004&Data%20Cubes&EE5F4698A91AD2F8CA2581B1000E09B0&0&July%202016&09.10.2017&Latest)
format for SA1 areas.

This [issue](https://github.com/jibeproject/silo/issues/1) has been addressed as per markdown code/log here:
https://github.com/jibeproject/Melbourne-zone-system/blob/master/update-zonesystem.csv-add-urban-rural-and-sa1_7dig.md

The code is also stored in the JIBE working group folder here, along with the sections of state data retrieved through running the `R` `.qmd` code and the output `zoneSystem.csv` file:
`melbourne\travel_demand_model_mito\Melbourne-zone-system`

The updated ouptut file was commited to the Melbourne gitlab repo:
https://gitlab.lrz.de/ga78fel/melbourne/-/commit/e025eaaeeafb06d7a72b8d06e7383d2b353d9e87
