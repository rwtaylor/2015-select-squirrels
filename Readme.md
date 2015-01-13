

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
ped_to_clean = ped_2014 %>% filter(grid == "KS") %>% rename(year = byear)

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



**Plot some pedigrees**


```r
source("plot_fix.R")
plot_ped_1 <- winnowPedigree(processed_pedigree, fullsib_cutoff = 2, generation_cutoff = 2, include_families = 1:5)
```

```
##   Var1 Freq
## 1    1  248
## 5    5   15
## 7    7   14
## 6    6   10
## 2    2    6
## 8    8    6
## 3    3    4
## 4    4    4
## 9    9    4
```

```r
pdf(file = "target_pedigree_1.pdf", width = 11, height = 8.5)
  plot_fix(plot_ped_1[[1]], width = 1000, cex = 0.25, col = plot_ped_1$color)
dev.off()
```

```
## pdf 
##   2
```

```r
print(plot_ped_1[[1]])
```

```
## Pedigree object with 293 subjects
## Bit size= 355
```

```r
write.csv(plot_ped_1$pedigree, file = "plot_ped_1.csv")
plot_fix(plot_ped_1[[1]], width = 1000, cex = 0.25, col = plot_ped_1$color)
```

![plot of chunk unnamed-chunk-5](figure/unnamed-chunk-5-1.png) 


```r
source("plot_fix.R")
plot_ped_2 <- winnowPedigree(processed_pedigree, fullsib_cutoff = 2, generation_cutoff = 1, include_families = 1)
```

```
##    Var1 Freq
## 1     1  385
## 11   11   58
## 10   10   23
## 6     6    8
## 14   14    8
## 4     4    7
## 8     8    7
## 13   13    7
## 2     2    6
## 3     3    5
## 5     5    5
## 16   16    5
## 18   18    5
## 7     7    4
## 9     9    4
## 12   12    4
## 15   15    4
## 17   17    4
```

```r
pdf(file = "target_pedigree_2.pdf", width = 11, height = 8.5)
  plot_fix(plot_ped_2[[1]], width = 1000, cex = 0.4, col = plot_ped_2$color)
```

```
## Did not plot the following people: 9426 9427
```

```r
dev.off()
```

```
## pdf 
##   2
```

```r
print(plot_ped_2[[1]])
```

```
## Pedigree object with 385 subjects
## Bit size= 503
```

```r
write.csv(plot_ped_2$pedigree, file = "plot_ped_2.csv")

plot_fix(plot_ped_2[[1]], width = 1000, cex = 0.4, col = plot_ped_2$color)
```

![plot of chunk unnamed-chunk-6](figure/unnamed-chunk-6-1.png) 

```
## Did not plot the following people: 9426 9427
```

