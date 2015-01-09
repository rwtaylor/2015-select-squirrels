# Which individuals to genotype?

  * outbred parents with large sibships
  * multigenerational links

**load pedigree**


```r
ped_2011 <- read.csv(file = "Pedigree2011b.csv", header = TRUE, stringsAsFactors = FALSE)
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
ped_to_clean = ped_2011 %>% filter(Grid == "KS") %>% dplyr::select(ID, DAM.ID, SIRE.ID, Sex, DNA, DAM.DNA, SIRE.DNA, BYEAR)

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
##    Var1 Freq
## 5     5   97
## 8     8   14
## 2     2   13
## 7     7   12
## 4     4   10
## 1     1    6
## 11   11    6
## 3     3    4
## 6     6    4
## 9     9    4
## 10   10    4
## 12   12    4
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
## Pedigree object with 146 subjects
## Bit size= 160
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
## 7     7  111
## 15   15   24
## 2     2   23
## 6     6   20
## 12   12   19
## 14   14   19
## 17   17   16
## 9     9   12
## 10   10   12
## 13   13   12
## 20   20   10
## 5     5    8
## 3     3    7
## 11   11    7
## 19   19    7
## 22   22    7
## 23   23    7
## 1     1    6
## 4     4    5
## 8     8    5
## 24   24    5
## 16   16    4
## 18   18    4
## 21   21    4
```

```r
pdf(file = "target_pedigree_2.pdf", width = 11, height = 8.5)
  plot_fix(plot_ped_2[[1]], width = 1000, cex = 0.4, col = plot_ped_2$color)
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
## Pedigree object with 111 subjects
## Bit size= 135
```

```r
write.csv(plot_ped_2$pedigree, file = "plot_ped_2.csv")

plot_fix(plot_ped_2[[1]], width = 1000, cex = 0.4, col = plot_ped_2$color)
```

![plot of chunk unnamed-chunk-6](figure/unnamed-chunk-6-1.png) 

