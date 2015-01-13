

# Which individuals to genotype?

  * outbred parents with large sibships
  * multigenerational links

**load pedigree**


```r
ped_2014 <- read.csv(file = "pedigree_2014.csv", header = TRUE, stringsAsFactors = FALSE)
ped_2014 <- tbl_dt(ped_2014)
```

```
## Loading required namespace: data.table
```

**Process pedigree function**
This function does some cleanup and calculates some variables.

  * Set id == 0 to NA
  * Cleanup sex variable 
  * Run fixPedigree (orders pedigree so that all parents exist as individuals before their offspring appear.)
  * DNA availability.
  * Parental relatedness and parental DNA availability.
  * Calculate fullsibs and sibships (only including individuals with DNA)
  * Generational depth (again only individuals with DNA)



**Process Pedigree**


```r
ped_to_clean = ped_2014 %>% filter(grid == "KS") %>% mutate(year = dna_year)

processed_pedigree <- processPedigree(ped_to_clean, dna_year_cutoff = 2006)
```

```
## Joining by: "id"
## Joining by: "id"
## Joining by: "parents"
## Joining by: "id"
## Joining by: c("id", "dam", "sire", "sex", "dna", "sire_dna", "dam_dna", "year", "has_dna", "par_rel", "par_dna", "parents", "fullsibs", "sibships")
## Joining by: "id"
```

**Winnow pedigree function**

  * Prune to individuals with DNA.
  * Prune pedigree to prioritize individuals with n or more fullsibs. Fullsibs calculated among individuals with DNA and only for individuals whose parents both have DNA.
  * Prune to indivdiauls with n generational depth.
  * Prune to largest family(ies). e.g. 1 = largest family, 1:5 = largest 5 families.
  * Results returned as "kinship2::pedigree" for plotting. Affected is individuals with related parents. Color is red for individuals without DNA.










**For Jaime**

Sent to Jaime 1/13/15.


```r
source("plot_fix.R")
kl_ped <- processed_pedigree[processed_pedigree$id %in% c(6381, 6417) | grepl("KL", processed_pedigree$dna), ]

request_01_15 <- winnowPedigree(kl_ped, fullsib_cutoff = 2, generation_cutoff = 3, include_families = c(1:2,4))
```

```
##   Var1 Freq
## 1    1   94
## 4    4   18
## 2    2   13
## 5    5   12
## 3    3    6
## 6    6    6
```

```r
pdf(file = "request_01_15.pdf", width = 11, height = 8.5)
  plot_fix(request_01_15[[1]], width = 10000, cex = 0.3, col = request_01_15$color)
dev.off()
```

```
## pdf 
##   2
```

```r
print(request_01_15[[1]])
```

```
## Pedigree object with 124 subjects
## Bit size= 125
```

```r
write.csv(request_01_15$pedigree %>% arrange(year, depth), row.names = FALSE, file = "request_01_15.csv")

plot_fix(request_01_15[[1]], width = 1000, cex = 0.3, col = request_01_15$color)
```

![plot of chunk unnamed-chunk-8](figure/unnamed-chunk-8-1.png) 

