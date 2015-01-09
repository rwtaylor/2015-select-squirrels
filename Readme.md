
```r
library(dplyr)
```

```
## 
## Attaching package: 'dplyr'
## 
## The following object is masked from 'package:stats':
## 
##     filter
## 
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

# Which individuals to genotype?

  * outbred parents with large sibships
  * multigenerational links

**load pedigree**


```r
ped_2011 <- read.csv(file = "../data/Pedigree2011b.csv", header = TRUE, stringsAsFactors = FALSE)
```

```
## Warning in file(file, "rt"): cannot open file '../data/Pedigree2011b.csv':
## No such file or directory
```

```
## Error in file(file, "rt"): cannot open the connection
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


```r
processPedigree <- function(ped, dna_year_cutoff = 2006){
  # pedigree format
  #dataframe with these columns in this order:
  #id, dam, sire, sex, dna, dam dna, sire dna, year_collected

  require(pedantics)
  require(kinship2)
  require(dplyr)
  require(foreach)
  require(doMC)
  registerDoMC()
  
  names(ped) <- c("id", "dam", "sire", "sex", "dna", "dam_dna", "sire_dna", "year")

  # Set zeros to NA
  ped$id[ped$id == 0] <- NA
  ped$dam[ped$dam == 0] <- NA
  ped$sire[ped$sire == 0] <- NA
  ped$dna[ped$dna == 0] <- NA
  ped$dam_dna[ped$dam_dna == 0] <- NA
  ped$sire_dna[ped$sire_dna == 0] <- NA

  # Sex Cleanup
  if(is.factor(ped$sex)){
    ped$sex <- as.character(ped$ex)
  }

  sexes <- c("male", "female", NA)

  if(is.character(ped$sex)){
    ped$sex <- charmatch(casefold(ped$sex, upper = FALSE), sexes, nomatch = 3)
    ped$sex <- sexes[ped$sex]
  }

  # Run fixPedigree
  data_to_save <- ped %>% arrange(id, dam, sire) %>% dplyr::select(id, sex, dna, sire_dna, dam_dna, year)

  ped <- fixPedigree(ped %>% dplyr::select(id, dam, sire))

  ped$id <- as.integer(as.character(ped$id))
  ped$sire <- as.integer(as.character(ped$sire))
  ped$dam <- as.integer(as.character(ped$dam))

  ped <- left_join(ped, data_to_save)

  # DNA available
  ped$has_dna <- 1
  ped$has_dna[is.na(ped$dna) | ped$dna == "DS"] <- 0
  # DNA year cutoff
  ped$has_dna[ped$year < dna_year_cutoff | is.na(ped$year)] <- 0

  # Parental relatedness
  # Also, record whether we have DNA for both parents

  # First generate relatedness matrix
  Rmatrix <- kinship(ped$id, ped$sire, ped$dam, ped$sex)

  parental_relatedness <- foreach(id.i = ped$id, .combine = rbind) %dopar% {
      sire_id <- ped$sire[ped$id == id.i]
      dam_id <- ped$dam[ped$id == id.i]
      if(!is.na(sire_id) & !is.na(dam_id)) {
        sire_dna <- ped$has_dna[ped$id == sire_id]
        dam_dna <- ped$has_dna[ped$id == dam_id]
        parental_dna <- sire_dna == 1 & dam_dna == 1
      } else {
        parental_dna <- FALSE
      }
      if(is.na(sire_id) | is.na(dam_id)){
        out <- data.frame(id = id.i, par_rel = NA, par_dna = parental_dna)
      } else {
        out <- data.frame(
          id = id.i,
          par_rel = Rmatrix[as.character(sire_id), as.character(dam_id)], par_dna = parental_dna)
      }
      out
  }

  ped <- left_join(ped, parental_relatedness, all.x = TRUE)
  
  # Prune pedigree to only informative for those with DNA and parents with DNA
  pped <- prunePed(ped, keep = ped$id[ped$has_dna == 1 & ped$par_dna], make.base = TRUE)
  
  # We want outbred parents with large sibships / offspring with many fullsibs. Need to only count parents for which we have DNA for both parents and offspring.
  pped$parents <- paste(pped$sire, pped$dam, sep = "_")

  fullsibs <- data.frame(
    table(pped$parents[pped$par_rel == 0 & pped$par_dna == "TRUE"]),
    stringsAsFactors = FALSE
  )
  names(fullsibs) <- c("parents", "fullsibs")
  fullsibs$parents <- as.character(fullsibs$parents)
  
  pped <- left_join(pped, fullsibs)
  
  pped$fullsibs[is.na(pped$fullsibs)] <- 0

  pped$fullsibs[is.na(pped$dam) | is.na(pped$sire)] <- NA

  parents <- pped %>% dplyr::select(dam, sire, parents, par_rel, sibships = fullsibs) %>% distinct()

  table(parents$sibships, parents$par_rel == 0)
  parents$sibships[parents$par_rel > 0] <- NA

  dams <- parents %>%
    dplyr::select(id = dam, sibships) %>%
    group_by(id) %>%
    summarize(sibships = max(sibships))

  sires <- parents %>%
    dplyr::select(id = sire, sibships) %>%
    group_by(id) %>%
    summarize(sibships = max(sibships))

  pped <- left_join(pped, rbind(dams, sires))

  # Record generational depth
  temp_ped <- pped
  temp_ped$sire[is.na(temp_ped$dam)] <- NA
  temp_ped$dam[is.na(temp_ped$sire)] <- NA
  temp_ped$depth <- kindepth(id = temp_ped$id, dad.id = temp_ped$sire, mom.id = temp_ped$dam)

  temp_depth <- temp_ped %>% dplyr::select(id, depth)
  pped <- left_join(pped, temp_ped)
  pped$id <- as.integer(pped$id)
  ped <- left_join(ped, pped %>% dplyr::select(id, fullsibs, sibships, depth))
  return(ped)
}

ped_to_clean = ped_2011 %>% filter(Grid == "KS") %>% dplyr::select(ID, DAM.ID, SIRE.ID, Sex, DNA, DAM.DNA, SIRE.DNA, BYEAR)
```

```
## Error in eval(expr, envir, enclos): object 'ped_2011' not found
```

```r
processed_pedigree <- processPedigree(ped_to_clean, dna_year_cutoff = 2006)
```

```
## Loading required package: pedantics
## Loading required package: MasterBayes
## Loading required package: coda
## Loading required package: lattice
## Loading required package: genetics
## Loading required package: combinat
## 
## Attaching package: 'combinat'
## 
## The following object is masked from 'package:utils':
## 
##     combn
## 
## Loading required package: gdata
## gdata: read.xls support for 'XLS' (Excel 97-2004) files ENABLED.
## 
## gdata: read.xls support for 'XLSX' (Excel 2007+) files ENABLED.
## 
## Attaching package: 'gdata'
## 
## The following object is masked from 'package:dplyr':
## 
##     combine
## 
## The following object is masked from 'package:stats':
## 
##     nobs
## 
## The following object is masked from 'package:utils':
## 
##     object.size
## 
## Loading required package: gtools
## Loading required package: MASS
## 
## Attaching package: 'MASS'
## 
## The following object is masked from 'package:dplyr':
## 
##     select
## 
## Loading required package: mvtnorm
## 
## 
## NOTE: THIS PACKAGE IS NOW OBSOLETE.
## 
## 
## 
##   The R-Genetics project has developed an set of enhanced genetics
## 
##   packages to replace 'genetics'. Please visit the project homepage
## 
##   at http://rgenetics.org for informtion.
## 
## 
## 
## 
## Attaching package: 'genetics'
## 
## The following objects are masked from 'package:base':
## 
##     %in%, as.factor, order
## 
## Loading required package: kinship2
## Loading required package: Matrix
## Loading required package: quadprog
## Loading required package: MCMCglmm
## Loading required package: ape
## Loading required package: grid
## Loading required package: foreach
## foreach: simple, scalable parallel programming from Revolution Analytics
## Use Revolution R for scalability, fault tolerance and more.
## http://www.revolutionanalytics.com
## Loading required package: doMC
## Loading required package: iterators
## Loading required package: parallel
```

```
## Error in names(ped) <- c("id", "dam", "sire", "sex", "dna", "dam_dna", : object 'ped_to_clean' not found
```

**Winnow pedigree**

  * Prune to individuals with DNA.
  * Prune pedigree to prioritize individuals with n or more fullsibs. Fullsibs calculated among individuals with DNA and only for individuals whose parents both have DNA.
  * Prune to indivdiauls with n generational depth.
  * Prune to largest family(ies). e.g. 1 = largest family, 1:5 = largest 5 families.
  * Results returned as "kinship2::pedigree" for plotting. Affected is individuals with related parents. Color is red for individuals without DNA.


```r
winnowPedigree <- function(ped, fullsib_cutoff = 3, generation_cutoff = 1, include_families = 1:100, no_dna_ok = FALSE){
  require(MCMCglmm)
  require(pedantics)
  require(kinship2)
  require(dplyr)
  pped <- ped
  if(no_dna_ok == FALSE){
    no_id <- pped %>% filter(has_dna == 0)
    pped$sire[ped$sire %in% no_id$id] <- NA
    pped$dam[ped$dam %in% no_id$id] <- NA
    pped <- pped %>% filter(has_dna == 1)
  }

  target_individuals <- pped %>% filter(fullsibs >= fullsib_cutoff & has_dna == 1 & par_rel == 0 & par_dna & depth >= generation_cutoff)
  target_ped <- prunePed(pped, keep = target_individuals$id)
  target_ped$sire[is.na(target_ped$dam)] <- NA
  target_ped$dam[is.na(target_ped$sire)] <- NA
  target_ped <- prunePed(target_ped, target_individuals$id)
  target_ped$color <- ifelse(target_ped$has_dna == 0, 2, 1)
  target_ped$affected <- ifelse(target_ped$par_rel > 0, 1, 0)
  target_ped$family <- makefamid(id = target_ped$id, father.id = target_ped$sire, mother.id = target_ped$dam)

  families <- data.frame(table(target_ped$family))

  families <- families[order(-families$Freq), ]

  target_ped <- target_ped %>%
    filter(family %in% families$Var1[include_families])
  target_ped$color <- ifelse(target_ped$has_dna == 0, 2, 1)

  t_ped <- pedigree(
    id = target_ped$id,
    dadid = target_ped$sire,
    momid = target_ped$dam,
    sex = target_ped$sex,
    affected = target_ped$affected
  )
  print(families)

  return(list(t_ped, color = target_ped$color, pedigree = target_ped))
}
```

**Some Plots**
Be sure to load the plot_fix function below.


```r
source("plot_fix.R")
plot_ped_1 <- winnowPedigree(processed_pedigree, fullsib_cutoff = 2, generation_cutoff = 2, include_families = 1:5)
```

```
## Error in winnowPedigree(processed_pedigree, fullsib_cutoff = 2, generation_cutoff = 2, : object 'processed_pedigree' not found
```

```r
pdf(file = "target_pedigree_1.pdf", width = 11, height = 8.5)
  plot_fix(plot_ped_1[[1]], width = 1000, cex = 0.25, col = plot_ped_1$color)
```

```
## Error in plot_fix(plot_ped_1[[1]], width = 1000, cex = 0.25, col = plot_ped_1$color): object 'plot_ped_1' not found
```

```r
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
## Error in print(plot_ped_1[[1]]): error in evaluating the argument 'x' in selecting a method for function 'print': Error: object 'plot_ped_1' not found
```

```r
write.csv(plot_ped_1$pedigree, file = "plot_ped_1.csv")
```

```
## Error in is.data.frame(x): object 'plot_ped_1' not found
```

```r
plot_fix(plot_ped_1[[1]], width = 1000, cex = 0.25, col = plot_ped_1$color)
```

```
## Error in plot_fix(plot_ped_1[[1]], width = 1000, cex = 0.25, col = plot_ped_1$color): object 'plot_ped_1' not found
```


```r
source("plot_fix.R")
plot_ped_2 <- winnowPedigree(processed_pedigree, fullsib_cutoff = 2, generation_cutoff = 1, include_families = 1)
```

```
## Error in winnowPedigree(processed_pedigree, fullsib_cutoff = 2, generation_cutoff = 1, : object 'processed_pedigree' not found
```

```r
pdf(file = "target_pedigree_2.pdf", width = 11, height = 8.5)
  plot_fix(plot_ped_2[[1]], width = 1000, cex = 0.4, col = plot_ped_2$color)
```

```
## Error in plot_fix(plot_ped_2[[1]], width = 1000, cex = 0.4, col = plot_ped_2$color): object 'plot_ped_2' not found
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
## Error in print(plot_ped_2[[1]]): error in evaluating the argument 'x' in selecting a method for function 'print': Error: object 'plot_ped_2' not found
```

```r
write.csv(plot_ped_2$pedigree, file = "plot_ped_2.csv")
```

```
## Error in is.data.frame(x): object 'plot_ped_2' not found
```

```r
plot_fix(plot_ped_2[[1]], width = 1000, cex = 0.4, col = plot_ped_2$color)
```

```
## Error in plot_fix(plot_ped_2[[1]], width = 1000, cex = 0.4, col = plot_ped_2$color): object 'plot_ped_2' not found
```

