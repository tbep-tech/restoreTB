  
# Evaluation of full model {.tabset}

Matched to nearest site in each category, +/- five years. 


```r
knitr::opts_chunk$set(message = F, warning = F)

library(tidyverse)
library(ggmap)
library(lubridate)
library(geosphere)
library(stringi)
library(tibble)
library(raster)
library(sp)
library(rgdal)
library(foreach)
library(doParallel)
library(forcats)

# source R files
source('R/get_chg.R')
source('R/get_clo.R')
source('R/get_cdt.R')
source('R/get_brk.R')
source('R/get_fin.R')
source('R/get_all.R')
source('R/rnd_dat.R')

## globals
wqvar1 <- 'sal'
wqvar2 <- 'tn'
wqvar3 <- 'chla'
mtch <- 1
yrdf <- 5
resgrp <- 'type' 
qts <- c(0.33, 0.66)
lbs1 <- c('sal_lo', 'sal_md', 'sal_hi')
lbs2 <- c('tn_lo', 'tn_md', 'tn_hi')

data(restdat)
data(reststat)
data(wqdat)
data(wqstat)

# restoration data types
restdat <- restdat %>% 
  mutate(
    type = factor(type, 
                  levels = c('HABITAT_ENHANCEMENT', 'HABITAT_ESTABLISHMENT', 'HABITAT_PROTECTION', 'NONPOINT_SOURCE', 'POINT_SOURCE'),
                  labels = c('hab_enh', 'hab_est', 'hab_pro', 'non_src', 'pnt_src')
    )
  )
```

## Pre-1994 data


```r
# get sub data, restoration sites
restdat_sub <- restdat %>% 
  filter(date < 1994) %>% 
  mutate(type = fct_drop(type))
reststat_sub <- reststat %>% 
  filter(id %in% restdat_sub$id)

## Distance to restoration sites
wqmtch <- get_clo(restdat_sub, reststat_sub, wqstat, resgrp = resgrp, mtch = mtch)

## Summarizing effects of restoration projects on salinity
salchg <- get_chg(wqdat, wqmtch, statdat, restdat_sub, wqvar = wqvar1, yrdf = yrdf)

## Summarizing effects of restoration projects on tn
tnchg <- get_chg(wqdat, wqmtch, statdat, restdat_sub, wqvar = wqvar2, yrdf = yrdf)

# Get conditional probability distributions for the restoration type on salinity 
salcdt <- get_cdt(salchg)

# Discretization of salinity conditional probability distributions: 
salbrk <- get_brk(salcdt, qts = qts)

# get final conditional probability for last child node:
tnallchg <- get_fin(tnchg, salbrk, salchg, lbs = lbs1)

## Summarizing effects of restoration projects on chlorophyll
chlchg <- get_chg(wqdat, wqmtch, statdat, restdat_sub, wqvar = wqvar3, yrdf = yrdf)

# Get conditional probability distributions for the restoration type on tn
# Get conditional probability distributions for the restoration type on tn
tncdt<- tnallchg %>% 
  dplyr::select(-salev, -saval) %>% 
  dplyr::select(stat, everything()) %>% 
  get_cdt

# Discretization of tn conditional probability distributions: 
tnbrk <- get_brk(tncdt, qts = qts)

# get final conditional probability for last child node:
chlallchg <- get_fin(chlchg, tnbrk, tnchg, lbs = lbs2)
```

### Habitat and water projects by type {.tabset}

#### salinity and nitrogen


```r
toplo <- tnallchg %>% 
  group_by_(.dots = names(.)[!names(.) %in% c('stat', 'cva', 'saval')]) %>% 
  summarize(
    chvalmd = mean(cval, na.rm = T)
    ) %>% 
  na.omit %>% 
  unite('rest', names(.)[!names(.) %in% c('cval', 'salev', 'chvalmd')], sep = ', ') %>% 
  mutate(
    salev = factor(salev, levels = c('sal_lo', 'sal_md', 'sal_hi')) 
  )

# plot
ggplot(toplo, aes(x = rest, y = chvalmd)) + 
  theme_bw() + 
  theme(
    axis.title.y = element_blank()
  ) +
  geom_bar(stat = 'identity') +
  facet_wrap(~ salev, ncol = 1) + 
  coord_flip() +
  scale_y_continuous('TN')
```

![](suball_eval_files/figure-html/unnamed-chunk-2-1.png)<!-- -->

#### nitrogen and chlorophyll


```r
toplo <- chlallchg %>% 
  group_by_(.dots = names(.)[!names(.) %in% c('stat', 'cva', 'saval')]) %>% 
  summarize(
    chvalmd = mean(cval, na.rm = T)
    ) %>% 
  na.omit %>% 
  unite('rest', names(.)[!names(.) %in% c('cval', 'salev', 'chvalmd')], sep = ', ') %>% 
  mutate(
    salev = factor(salev, levels = c('tn_lo', 'tn_md', 'tn_hi')) 
  )

# plot
ggplot(toplo, aes(x = rest, y = chvalmd)) + 
  theme_bw() + 
  theme(
    axis.title.y = element_blank()
  ) +
  geom_bar(stat = 'identity') +
  facet_wrap(~ salev, ncol = 1) + 
  coord_flip() +
  scale_y_continuous('Chlorophyll')
```

![](suball_eval_files/figure-html/unnamed-chunk-3-1.png)<!-- -->

### Distance to restoration sites {.tabset}


```r
wqmtch <- get_clo(restdat_sub, reststat_sub, wqstat, resgrp = 'type', mtch = mtch)
```

#### Closest 

```r
## 
# plots

# combine lat/lon for the plot
toplo <- wqmtch %>% 
  left_join(wqstat, by = 'stat') %>% 
  left_join(reststat_sub, by = 'id') %>% 
  rename(
    `Restoration\ntype` = resgrp,
    `Distance (dd)` = dist
  )
    
# restoration project grouping column
resgrp <- 'type'
restall <- left_join(restdat_sub, reststat_sub, by = 'id')
names(restall)[names(restall) %in% resgrp] <- 'Restoration\ntype'

# extent
ext <- make_bbox(wqstat$lon, wqstat$lat, f = 0.1)
map <- get_stamenmap(ext, zoom = 12, maptype = "toner-lite")

# base map
pbase <- ggmap(map) +
  theme_bw() +
  theme(
    axis.title.x = element_blank(),
    axis.title.y = element_blank()
  ) +
  geom_point(data = restall, aes(x = lon, y = lat, fill = `Restoration\ntype`), size = 4, pch = 21) +
  geom_point(data = wqstat, aes(x = lon, y = lat), size = 2)

# closest
toplo1 <- filter(toplo, rnk %in% 1)

pbase + 
  geom_segment(data = toplo1, aes(x = lon.x, y = lat.x, xend = lon.y, yend = lat.y, alpha = -`Distance (dd)`, linetype = `Restoration\ntype`), size = 1)
```

![](suball_eval_files/figure-html/unnamed-chunk-5-1.png)<!-- -->

#### Closest three

```r
# closest five percent
# fvper <- max(toplo$rnk) %>% 
#   `*`(0.2) %>% 
#   ceiling
toplo2 <- filter(toplo, rnk %in% c(1:3))

pbase + 
  geom_segment(data = toplo2, aes(x = lon.x, y = lat.x, xend = lon.y, yend = lat.y, alpha = -`Distance (dd)`, linetype = `Restoration\ntype`), size = 1)
```

![](suball_eval_files/figure-html/unnamed-chunk-6-1.png)<!-- -->

#### Closest all

```r
# closest all combo
toplo3 <- toplo

pbase + 
  geom_segment(data = toplo3, aes(x = lon.x, y = lat.x, xend = lon.y, yend = lat.y, alpha = -`Distance (dd)`, linetype = `Restoration\ntype`), size = 1)
```

![](suball_eval_files/figure-html/unnamed-chunk-7-1.png)<!-- -->

## Post-1994 data


```r
# get sub data, restoration sites
restdat_sub <- restdat %>% 
  filter(date >= 1994) %>%  
  mutate(type = fct_drop(type))
reststat_sub <- reststat %>% 
  filter(id %in% restdat_sub$id)

## Distance to restoration sites
wqmtch <- get_clo(restdat_sub, reststat_sub, wqstat, resgrp = resgrp, mtch = mtch)

## Summarizing effects of restoration projects on salinity
salchg <- get_chg(wqdat, wqmtch, statdat, restdat_sub, wqvar = wqvar1, yrdf = yrdf)

## Summarizing effects of restoration projects on tn
tnchg <- get_chg(wqdat, wqmtch, statdat, restdat_sub, wqvar = wqvar2, yrdf = yrdf)

# Get conditional probability distributions for the restoration type on salinity 
salcdt <- get_cdt(salchg)

# Discretization of salinity conditional probability distributions: 
salbrk <- get_brk(salcdt, qts = qts)

# get final conditional probability for last child node:
tnallchg <- get_fin(tnchg, salbrk, salchg, lbs = lbs1)

## Summarizing effects of restoration projects on chlorophyll
chlchg <- get_chg(wqdat, wqmtch, statdat, restdat_sub, wqvar = wqvar3, yrdf = yrdf)

# Get conditional probability distributions for the restoration type on tn
# Get conditional probability distributions for the restoration type on tn
tncdt<- tnallchg %>% 
  dplyr::select(-salev, -saval) %>% 
  dplyr::select(stat, everything()) %>% 
  get_cdt

# Discretization of tn conditional probability distributions: 
tnbrk <- get_brk(tncdt, qts = qts)

# get final conditional probability for last child node:
chlallchg <- get_fin(chlchg, tnbrk, tnchg, lbs = lbs2)
```

### Habitat and water projects by type {.tabset}

#### salinity and nitrogen


```r
toplo <- tnallchg %>% 
  group_by_(.dots = names(.)[!names(.) %in% c('stat', 'cva', 'saval')]) %>% 
  summarize(
    chvalmd = mean(cval, na.rm = T)
    ) %>% 
  na.omit %>% 
  unite('rest', names(.)[!names(.) %in% c('cval', 'salev', 'chvalmd')], sep = ', ') %>% 
  mutate(
    salev = factor(salev, levels = c('sal_lo', 'sal_md', 'sal_hi')) 
  )

# plot
ggplot(toplo, aes(x = rest, y = chvalmd)) + 
  theme_bw() + 
  theme(
    axis.title.y = element_blank()
  ) +
  geom_bar(stat = 'identity') +
  facet_wrap(~ salev, ncol = 1) + 
  coord_flip() +
  scale_y_continuous('TN')
```

![](suball_eval_files/figure-html/unnamed-chunk-9-1.png)<!-- -->

#### nitrogen and chlorophyll


```r
toplo <- chlallchg %>% 
  group_by_(.dots = names(.)[!names(.) %in% c('stat', 'cva', 'saval')]) %>% 
  summarize(
    chvalmd = mean(cval, na.rm = T)
    ) %>% 
  na.omit %>% 
  unite('rest', names(.)[!names(.) %in% c('cval', 'salev', 'chvalmd')], sep = ', ') %>% 
  mutate(
    salev = factor(salev, levels = c('tn_lo', 'tn_md', 'tn_hi')) 
  )

# plot
ggplot(toplo, aes(x = rest, y = chvalmd)) + 
  theme_bw() + 
  theme(
    axis.title.y = element_blank()
  ) +
  geom_bar(stat = 'identity') +
  facet_wrap(~ salev, ncol = 1) + 
  coord_flip() +
  scale_y_continuous('Chlorophyll')
```

![](suball_eval_files/figure-html/unnamed-chunk-10-1.png)<!-- -->

### Distance to restoration sites


```r
wqmtch <- get_clo(restdat_sub, reststat_sub, wqstat, resgrp = 'type', mtch = mtch)
```


```r
## 
# plots

# combine lat/lon for the plot
toplo <- wqmtch %>% 
  left_join(wqstat, by = 'stat') %>% 
  left_join(reststat_sub, by = 'id') %>% 
  rename(
    `Restoration\ntype` = resgrp,
    `Distance (dd)` = dist
  )
    
# restoration project grouping column
resgrp <- 'type'
restall <- left_join(restdat_sub, reststat_sub, by = 'id')
names(restall)[names(restall) %in% resgrp] <- 'Restoration\ntype'

# extent
ext <- make_bbox(wqstat$lon, wqstat$lat, f = 0.1)
map <- get_stamenmap(ext, zoom = 12, maptype = "toner-lite")

# base map
pbase <- ggmap(map) +
  theme_bw() +
  theme(
    axis.title.x = element_blank(),
    axis.title.y = element_blank()
  ) +
  geom_point(data = restall, aes(x = lon, y = lat, fill = `Restoration\ntype`), size = 4, pch = 21) +
  geom_point(data = wqstat, aes(x = lon, y = lat), size = 2)

# closest
toplo1 <- filter(toplo, rnk %in% 1)

pbase + 
  geom_segment(data = toplo1, aes(x = lon.x, y = lat.x, xend = lon.y, yend = lat.y, alpha = -`Distance (dd)`, linetype = `Restoration\ntype`), size = 1)
```

![](suball_eval_files/figure-html/unnamed-chunk-12-1.png)<!-- -->