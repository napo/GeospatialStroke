## —- Libraries and Census Data

``` r
library(tidyverse)
#> ── Attaching packages ────────────────────────────────── tidyverse 1.2.1 ──
#> ✔ ggplot2 3.1.0     ✔ purrr   0.2.5
#> ✔ tibble  1.4.2     ✔ dplyr   0.7.8
#> ✔ tidyr   0.8.2     ✔ stringr 1.3.1
#> ✔ readr   1.2.1     ✔ forcats 0.3.0
#> ── Conflicts ───────────────────────────────────── tidyverse_conflicts() ──
#> ✖ dplyr::filter() masks stats::filter()
#> ✖ dplyr::lag()    masks stats::lag()
library(sf)
#> Linking to GEOS 3.7.0, GDAL 2.3.2, PROJ 5.2.0
library(units)
#> udunits system database from /usr/share/udunits
library(tmaptools)
postcodeboundariesAUS <- 
    file.path(here::here(), "ABSData", "Boundaries/POA_2016_AUST.shp") %>%
    sf::read_sf ()

basicDemographicsVIC <- file.path(here::here(), "ABSData",
                                  "2016 Census GCP Postal Areas for VIC",
                                  "2016Census_G01_VIC_POA.csv") %>%
    readr::read_csv()
#> Parsed with column specification:
#> cols(
#>   .default = col_double(),
#>   POA_CODE_2016 = col_character()
#> )
#> See spec(...) for full column specifications.
```

Clean up the demographics to only those columns that we’re interested
in. Presume just for illustrative purposes here that those are only the
basic “Age” classes. There are also columns about the ages of persons
attending educational institutions which need to be removed.

``` r
library(magrittr)
#> 
#> Attaching package: 'magrittr'
#> The following object is masked from 'package:purrr':
#> 
#>     set_names
#> The following object is masked from 'package:tidyr':
#> 
#>     extract
nms <- names (basicDemographicsVIC)
keep_cols <- c ("POA_CODE_2016", "Age")
remove_cols <- c ("Age_psns")
keep_index <- sapply (keep_cols, function (i) grep (i, nms)) %>%
    unlist() %>%
    unique()
remove_index <- sapply (remove_cols, function (i) grep (i, nms)) %>%
    unlist () %>%
    unique ()
keep_index <- keep_index [!keep_index %in% remove_index]
basicDemographicsVIC %<>%
    select(keep_index) %>%
    select(-remove_index)
```

## —- JoinCensusAndBoundaries —-

Join the demographics and shape tables, retaining victoria only use
postcode boundaries as the reference data frame so that coordinate
reference system is retained.

``` r
basicDemographicsVIC <- right_join(postcodeboundariesAUS,
                                   basicDemographicsVIC, 
                                   by=c("POA_CODE" = "POA_CODE_2016"))
```

## —- GeocodeRehabNetwork —-

To be clean
up

``` r
rehab_addresses <- c(DandenongHospital = "Dandenong Hospital, Dandenong VIC 3175, Australia",
                     CaseyHospital = "62-70 Kangan Dr, Berwick VIC 3806, Australia",
                     KingstonHospital = "The Kingston Centre, Heatherton VIC 3202, Australia")
RehabLocations <- tmaptools::geocode_OSM(rehab_addresses, as.sf=TRUE)
```

transform rehab locations to the same reference system

``` r
RehabLocations <- sf::st_transform(RehabLocations,
                                   sf::st_crs(basicDemographicsVIC))
```

## Check geocoding

With `tmap`:

``` r
library(tmap)
tmap_mode("view")

tm_shape(RehabLocations) + tm_markers() + 
  tm_basemap("OpenStreetMap")
```

Or with `mapdeck`:

``` r
library(mapdeck)
set_token(Sys.getenv("MAPBOX_TOKEN"))
mapdeck(location = c(145.2, -38),
        zoom = 12) %>%
    add_pointcloud (RehabLocations,
        layer_id = "rehab-locations")
```

## —- Postcodes surrounding rehab locations

There are 699 postcodes which we now want to reduce to only those within
a specified distance of the rehab locations, chosen here as 10km. Note
that we just use straight line distances here, because we only need to
roughly determine which postcodes surround our rehab centres. The
subsequent calculations will then use more accurate distances along
street networks.

``` r
dist_to_loc <- function (geometry, location){
    units::set_units(st_distance(geometry, location)[,1], km)
}
dist_range <- units::set_units(10, km)

#basicDemographicsVIC <- basicDemographicsVIC_old
basicDemographicsVIC_old <- basicDemographicsVIC
basicDemographicsVIC <- mutate(basicDemographicsVIC,
       DirectDistanceToDandenong = dist_to_loc(geometry,RehabLocations["DandenongHospital", ]),
       DirectDistanceToCasey     = dist_to_loc(geometry,RehabLocations["CaseyHospital", ]),
       DirectDistanceToKingston  = dist_to_loc(geometry,RehabLocations["KingstonHospital", ]),
       DirectDistanceToNearest   = pmin(DirectDistanceToDandenong,
                                        DirectDistanceToCasey,
                                        DirectDistanceToKingston)
)
basicDemographicsRehab <- filter(basicDemographicsVIC,
                                 DirectDistanceToNearest < dist_range) %>%
        mutate(Postcode = as.numeric(POA_CODE16)) %>%
        select(-grep("POA_", names(basicDemographicsVIC)))
```

That reduces the data down to 47 nearby postcodes, with the last 2 lines
converting all prior postcode columns (of which there were several all
beginning with “POA”) to a single numeric column named “Postcode”.

## —- SamplePostCodes —-

Select random addresses using a geocoded database

``` r
devtools::install_github("HughParsonage/PSMA")
```

Will increase this for the real example

``` r
addressesPerPostcode <- 500
```

A special function so we can sample the postcodes as we go. Sampling
syntax is due to the use of data.table inside PSMA. The last
`st_as_sf()` command converts the points labelled “LONGITUDE” and
“LATITUDE” into `sf::POINT` objects. (This function takes a few
seconds because of the `fetch_postcodes` call.)

``` r
library(PSMA)
samplePCode <- function(pcode, number) {
  d <- fetch_postcodes(pcode)
  return(d[, .SD[sample(.N, min(number, .N))], by=.(POSTCODE)])
}

randomaddresses <- map(basicDemographicsRehab$Postcode,
                       samplePCode,
                       number=addressesPerPostcode) %>%
            bind_rows() %>%
            sf::st_as_sf(coords = c("LONGITUDE", "LATITUDE"),
                         crs=st_crs(basicDemographicsRehab),
                         agr = "constant")
```

## —- PlotSampleLocations —-

With `tmap`:

``` r
library(tmap)
tmap_mode("view")
tm_shape(randomaddresses) + tm_markers(clustering=FALSE) + 
    tm_basemap("OpenStreetMap")
head(randomaddresses)
```

or with `mapdeck`

``` r
library(mapdeck)
set_token(Sys.getenv("MAPBOX_TOKEN"))
mapdeck(location = c(145.2, -38),
        zoom = 12) %>%
    add_pointcloud (randomaddresses,
                    radius = 2,
                    layer_id = "randomaddresses")
```

## —- AddressesToRehab —-

Compute the road distance and travel time from each address to each
hospital. This first requires a local copy of the street network within
the bounding polygon defined by `basicDemographicsRehab`. This is
easiest done with the `dodgr` package, which directly calls the
`osmdata` package to do the downloading.

### Street Network

It is instructive to examine the `mapdeck` view of the postcode
polygons:

``` r
library(mapdeck)
set_token(Sys.getenv("MAPBOX_TOKEN"))
mapdeck(location = c(145.2, -38),
        zoom = 12) %>%
    add_polygon (basicDemographicsRehab,
        layer_id = "randomaddresses")
```

The basic way to download the street network is within a defined,
implicitly rectangular, bounding box, but in this case that extends from
Mornington to St Kilda, and out to the Dandenongs, and even Koo Wee
Rup\! It is much better to extract the street network only within the
polygon defining our nearby postcode areas, which first needs to be
re-projected onto the CRS of OpenStreetMap data, which is epsg4326.
`st_union` merges all of the polygons to form the single enclosing
polygon, and the final command simply extracts the longitudinal and
latitudinal coordinates of that polygon (rather than leaving them in
`sf` format).

``` r
bounding_polygon <- sf::st_transform(basicDemographicsRehab,
                                     sf::st_crs(4326)) %>%
    sf::st_union () %>%
    sf::st_coordinates ()
bounding_polygon <- bounding_polygon [, 1:2]
```

We can now download the street network enclosed within that polygon.
Note that this is still a rather large network - over 40MB of data
representing over 60,000 street sections - that might take a minute or
two to process. It is therefore easier to save the result to discuss for
quicker re-usage.

``` r
library(dodgr)
system.time (
dandenong_streets <- dodgr_streetnet (bounding_polygon, expand = 0, quiet = FALSE)
)
saveRDS (dandenong_streets, file = "dandenong-streets.Rds")
format (file.size ("dandenong-streets.Rds"), big.mark = ",")
nrow (dandenong_streets)
```

The network can then be re-loaded with

``` r
dandenong_streets <- readRDS ("dandenong-streets.Rds")
```

### Distances to Hospitals

The `dodgr` package needs to de-compose the `sf`-formatted street
network, which consists of long, connected road segments, into
individual edges. This is done with the `weight_streetnet()` function,
which modifies the distance of each edge to reflect typical travel
conditions for a nominated mode of transport.

``` r
library (dodgr)
dandenong_streets <- readRDS ("dandenong-streets.Rds")
net <- weight_streetnet (dandenong_streets, wt_profile = "motorcar")
#> The following highway types are present in data yet lack corresponding weight_profile values: raceway, construction, corridor, road, proposed, living_street, bus_stop, NA,
nrow (dandenong_streets); nrow (net)
#> [1] 62624
#> [1] 626391
```

The 62,624 streets have been converted to 626,391 distinct edges. We can
now use the `net` object to calculate the distances, along with simple
numeric coordinates of our routing points, projected on to the same CRS
as OpenStreetMap (OSM), which is 4326:

``` r
from <- st_coordinates (st_transform (randomaddresses, crs = 4326))
to <- st_coordinates (st_transform (RehabLocations, crs = 4326))
```

Although not necessary, distance calculation is quicker if we map these
`from` and `to` points precisely on to the network itself. OSM assigns
unique identifiers to every single object, and so our routing
coordinates can be converted to OSM identifiers of the nearest street
nodes. The nodes themselves are obtained with the `dodgr_vertices()`
function.

``` r
nodes <- dodgr_vertices (net)
from <- nodes$id [match_pts_to_graph (nodes, from, connected = TRUE)]
to <- nodes$id [match_pts_to_graph (nodes, to, connected = TRUE)]
```

The matrices of `from` and `to` coordinates have now been converted to
simple vectors of OSM identifiers.

``` r
system.time (
             d <- dodgr_dists (net, from = from, to = to)
)
#>    user  system elapsed 
#>   1.126   0.010   1.032
```

And that takes only around 1 second to calulate distances between (3
rehab centres times 20,000 random addresses = ) 60,000 pairs of points.

## —- CatchmentBasins —-

I did this by creating a Voronoi tesselation, of the random addresses,
and combining the regions according to the nearest hospital. Hopefully
there are sf tools for this.

## —- CasesPerCentre —-

Also need a per postcode breakdown of proportion of addresses going to
each centre, so that we can compute the number of cases going to each
centre