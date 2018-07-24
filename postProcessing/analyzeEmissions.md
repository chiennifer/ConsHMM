Post-processing of ConsHMM state parameters
================

The ConsHMM pipeline learns a conservation state model from a multiple-sequence alignment of different genomes. This notebook was used to visualize the parameters of a 100-state ConsHMM model learned from the hg19 Multiz 100-way alignment. It can be used to visualize any ConsHMM model, by editing the input files.

### Required libraries.

``` r
library(cba)
library(yaml)
library(tidyr)
library(dplyr)
library(pheatmap)
library(heatmaply) #You may also need to install pandoc (https://pandoc.org/installing.html)
```

Input parameters
----------------

This notebook requires an emissions parameter file created by running ChromHMM on a multiple-sequence alignment processed through ConsHMM. Specify the name of this file using the `emissionsFileName` variable below:

``` r
emissionsFileName = "../models/hg19_multiz100way/emissions_100.txt"
```

To help interpret the emission parameters, provide a `.yaml` file that annotates the genome names used in the multiple sequence alignment with a common name (used for readability), a distance to human (used for arranging columns in the order of a phylogenetic tree), and a group (used to group columns into clades). Each species should have an entry in the `.yaml` file such as:

``` yaml
panTro4: 
 commonName: Chimp
 group: Primate
 distanceToHuman: 1
```

Specify the name of the `.yaml` file using the `yamlFileName` variable below:

``` r
yamlFileName = "speciesNames.yaml"
```

The states of the model are clustered hierarchically using optimal leaf ordering. To cut the dendrogram into a set number of clusters specify the number of clusters using the `numClusters` variable below. The resulting heatmap will insert white space between the clusters. Set to 0 for no set number of clusters. The full dendrogram is always displayed.

``` r
numClusters = 6
```

The notebook creates a large heatmap that is best viewed by saving into `.pdf` format. Specify the name of the output file below.

``` r
outputFileName = "heatmap_hg19_multiz100way.pdf"
```

Processing emissions file
-------------------------

Reading in and renaming data.

``` r
emissions = read.table(emissionsFileName, header=TRUE, sep="\t")
species_names_yaml = yaml.load_file(yamlFileName)
species_names = as.data.frame(cbind(names(species_names_yaml), matrix(unlist(species_names_yaml), ncol=length(unlist(species_names_yaml[1])), byrow = TRUE)))
rownames(species_names) = NULL
colnames(species_names) = cbind("genomeName", "commonName", "group", "distanceToHuman")

numSpecies = (ncol(emissions) - 1) / 2 
```

Due to internal ChromHMM representation, the matching probabilities will not be meaningful when the corresponding aligning probability is low. To correct for this we replace the matching probabilites are replaced with the corresponding align \* match probability.

``` r
emissions_matched_replaced = emissions
columns = c()
for (i in c(2:ncol(emissions))) {
  split_column_name = unlist(strsplit(colnames(emissions)[i], split="_", fixed=TRUE))
  species = split_column_name[1]
  emission_type = split_column_name[2]
  columns = rbind(columns, c(species, emission_type))
  
  if (emission_type == "matched") {
    emissions_matched_replaced[i] = emissions[,i] * emissions[[paste(species, "aligned", sep="_")]]
  }
}
columns = as.data.frame(columns)
colnames(columns) = c("genomeName", "emission_type")
columns$index = c(2:199)
columns = columns %>% mutate(fullName = paste(columns[,1], columns[,2], sep="_"))
```

Reordering columns according to distance to human.

``` r
emissions_columns_reordered = emissions_matched_replaced
columns = merge(columns, species_names, by="genomeName")
columns$distanceToHuman = as.numeric(as.character(columns$distanceToHuman))
newcolnames = colnames(emissions_matched_replaced)
for (i in c(1:nrow(columns))) {
  if (columns$emission_type[i] == "aligned") {
    emissions_columns_reordered[,(columns$distanceToHuman[i] + 1)] = emissions_matched_replaced[,columns$index[i]]
    newcolnames[columns$distanceToHuman[i] + 1] = as.character(columns$commonName[i])
  } else {
    emissions_columns_reordered[,(columns$distanceToHuman[i] + 100)] = emissions_matched_replaced[,columns$index[i]]
    newcolnames[columns$distanceToHuman[i] + 100] = as.character(columns$commonName[i])
  }
}
```

Clustering function used for clutering rows hierarchially according to optimal leaf ordering.

``` r
OLO_clustering = function(hc, mat) {
  d <- dist(mat)
  hc <- hclust(d)
  co <- order.optimal(d, hc$merge)
  ho <- hc
  ho$merge <- co$merge
  ho$order <- co$order
  hc <- ho
}
```

Basic formatting for column names at bottom and within the groupings at the top to be common names. (this needs to be changed for different MSAs)

``` r
for (i in 101:199){
newcolnames[i] = paste(" ", newcolnames[i])
}
rownames(annotation_col) = newcolnames[-1]
```

Heatmap plotting
----------------

The plotting below was optimized for the 100 state ConsHMM model learned from the hg19 Multiz 100-way alignment. Use the code below to specify plotting parameters to the pheatmap package that best fit the model you are analyzing.

``` r
columns = columns %>% arrange(emission_type, distanceToHuman)
colnames(emissions_columns_reordered) = c("state", columns$fullName)
annotation_col = columns %>% select(group)
rownames(annotation_col) = columns$fullName

#This is specific to the heatmaply interactive mapping
heatmaply(emissions_columns_reordered[,-1], xlab="Species Names\nAligned(left), Matched(right)", ylab="Conservation States", main="Heatmap of Conservation State by Species",  key.title="Emission\nProbability", Colv = FALSE, labCol = newcolnames[-1], file = outputFileName, k_row=6, scale_fill_gradient_fun = ggplot2::scale_fill_gradient2(low = "blue", high = "red", midpoint=0.5),   col_side_colors = annotation_col)

#This is specific to using pheatmap to create the static map
# Section below allows for diagonal column names in the pheatmap package (from StackOverflow)
# Edit body of pheatmap:::draw_colnames, customizing it to your liking
draw_colnames_45 <- function (coln, ...) {
    m = length(coln)
    x = (1:m)/m - 1/2/m
    grid.text(coln, x = x, y = unit(0.96, "npc"), vjust = .5, 
        hjust = 1, rot = 45, gp = gpar(...)) ## Was 'hjust=0' and 'rot=270'
}

# 'Overwrite' default draw_colnames with your own version 
assignInNamespace(x="draw_colnames", value="draw_colnames_45",
ns=asNamespace("pheatmap"))

pdf(outputFileName, width = 33, height = 16, onefile = FALSE)
pheatmap(emissions_columns_reordered[,-1], cluster_rows = TRUE, cluster_cols = FALSE, clustering_callback = OLO_clustering, cellwidth = 10, cellheight = 9, border_color = NA, labels_col = newcolnames[-1], gaps_col = numSpecies, fontsize_col = 7, annotation_col = annotation_col, cutree_rows = 6, display_numbers = TRUE, fontsize_number = 3)
dev.off()
```

    ## quartz_off_screen 
    ##                 2
