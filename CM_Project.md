# Project goal: Compare human and mouse genes that change significantly with doxycyline exposure

# DESEQ2 Analysis for human and mouse genes

## Preparing mouse data for DESEQ2 analysis

``` r
#Raw counts file
MS_counts <- read.table("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/DATA/MOUSE/salmon.merged.gene_counts.tsv", header=TRUE, row.names=1)

#make g2s
g2s <- data.frame(
  gene_id = rownames(MS_counts),
  gene_name = MS_counts[, 1]
)
#Remove gene_name
MS_counts <- MS_counts[, -1]

# Round counts to integer mode required for DESEQ2
MS_integer <- round(MS_counts)

col_data <- data.frame(
  sample_id = colnames(MS_counts))

split_values <- strsplit(col_data$sample_id, "_")

# So here we will go through each row of split_values and run a "generic function(x)" 
# We will then retain the second item which is the time point value in sample_id
condition_values <- sapply(split_values, function(x) x[[1]])
time_values <- sapply(split_values, function(x) x[[2]])

# Similar to above we are using apply to grab the third fragment in split_values (replicate value)
replicate_values <- sapply(split_values, function(x) x[[3]])

# Adding condition and time point into samplesheet for DESEQ2
col_data$time <- time_values
col_data$condition <- condition_values
# Now let's add another column for replicate
col_data$replicate <- replicate_values

#facotring timepoint column
col_data$time <- factor(col_data$time, levels = c("0", "12", "24", "48", "96"))
levels(col_data$time)
```

    ## [1] "0"  "12" "24" "48" "96"

## Running DESEQ2 analysis on mouse data

``` r
stopifnot(all(colnames(MS_counts) == col_data$sample_id))

# Create the DESeqDataSet
MS_dds <- DESeqDataSetFromMatrix(countData = MS_integer, colData = col_data, design = ~ time)
```

    ## converting counts to integer mode

``` r
# Perform the LRT to find genes changing due to MS
MS_dds <- DESeq(MS_dds, test = "LRT", reduced = ~ 1)
```

    ## estimating size factors

    ## estimating dispersions

    ## gene-wise dispersion estimates

    ## mean-dispersion relationship

    ## final dispersion estimates

    ## fitting model and testing

``` r
MS_dds <- MS_dds [  rowSums ( counts (MS_dds) ) >  1 , ] 
nrow (MS_dds)
```

    ## [1] 30021

``` r
#Curating all results into data frame
resultsNames(MS_dds)
```

    ## [1] "Intercept"    "time_12_vs_0" "time_24_vs_0" "time_48_vs_0" "time_96_vs_0"

``` r
result_names <- resultsNames(MS_dds)

results_names <- result_names[-1]

res_df <- data.frame("gene_id" = character(), 
                     "baseMean" = numeric(), 
                     "log2FoldChange" = numeric(), 
                     "lfcSE" = numeric(),
                     "stat" = numeric(),
                     "pvalue" = numeric(),
                     "padj" = numeric(),
                     "gene_name" = character(),
                     "result_name" = character())

# For loop to get all results per time point  

for(i in 1:length(results_names)) {
  results_name <- results_names[i]
  res <- results(MS_dds, name = results_name)
  tmp_res_df <- res %>% as.data.frame() %>%
    rownames_to_column("gene_id") %>%
    merge(g2s) %>%
    mutate(result_name = results_name)
  res_df <- dplyr::bind_rows(res_df, tmp_res_df)
  
}

MS_res_df <- res_df

save(MS_dds, MS_res_df, file = "/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/RESULTS/MOUSEVSHUMAN/ML/MS_LRT_res_df.RData")
```

## Finding mouse genes that change signficicantly

``` r
# Calculate the maximum fold-change in any one timepoint
  MS_maxfc <- MS_res_df %>%
    group_by(gene_id) %>%
    summarize(max_fc = max(abs(log2FoldChange))) 
  
  # merge max shrnklfc into dataframe
  MS_res_df <- MS_res_df %>%
    left_join(MS_maxfc)
```

    ## Joining with `by = join_by(gene_id)`

``` r
  MS_res_df_padj0.05 <- MS_res_df %>% 
  filter(padj <= 0.05)
  print(length(unique(MS_res_df_padj0.05$gene_id)))#7665
```

    ## [1] 7665

``` r
  MS_sig <- MS_res_df_padj0.05 %>%
  filter(max_fc >= 1)
  print(length(unique(MS_sig$gene_id)))#609
```

    ## [1] 609

``` r
  #compare with non LRT results
  #load("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/RESULTS/MOUSE/DESEQ_results.rdata"), this file not exist anymore
  #filtered_res_df <- filtered_res_df %>%
  #filter(!(gene_name %in% MS_sig$gene_name))
  #print(length(unique(filtered_res_df$gene_id))) #135
  
 # MS_filtered_res_df <- filtered_res_df
  save(MS_sig, file = "/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/RESULTS/MOUSEVSHUMAN/ML/MS_sig.RData")
```

## Preparing human data for DESEQ2 analysis

``` r
#Raw counts file
human_counts <- read.table("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/DATA/HUMAN/salmon.merged.gene_counts.tsv", header=TRUE, row.names=1)

human_counts <- human_counts %>%
  select(gene_name, gfp_0_1, gfp_0_2, gfp_0_3, gfp_12_1, gfp_12_2, gfp_12_3, gfp_24_1, gfp_24_2, gfp_24_3, gfp_48_1, gfp_48_2, gfp_48_3, gfp_96_1, gfp_96_2, gfp_96_3)

#make g2s
g2s <- data.frame(
  gene_id = rownames(human_counts),
  gene_name = human_counts[, 1]
)
#Remove gene_name
human_counts <- human_counts[, -1]

# Round counts to integer mode required for DESEQ2
human_integer <- round(human_counts)

col_data <- data.frame(
  sample_id = colnames(human_counts))

split_values <- strsplit(col_data$sample_id, "_")

# So here we will go through each row of split_values and run a "generic function(x)" 
# We will then retain the second item which is the time point value in sample_id
condition_values <- sapply(split_values, function(x) x[[1]])
time_values <- sapply(split_values, function(x) x[[2]])

# Similar to above we are using apply to grab the third fragment in split_values (replicate value)
replicate_values <- sapply(split_values, function(x) x[[3]])

# Adding condition and time point into samplesheet for DESEQ2
col_data$time <- time_values
col_data$condition <- condition_values
# Now let's add another column for replicate
col_data$replicate <- replicate_values

#facotring timepoint column
col_data$time <- factor(col_data$time, levels = c("0", "12", "24", "48", "96"))
levels(col_data$time)
```

    ## [1] "0"  "12" "24" "48" "96"

## Running DESEQ2 analysis on human data

``` r
stopifnot(all(colnames(human_counts) == col_data$sample_id))

# Create the DESeqDataSet
human_dds <- DESeqDataSetFromMatrix(countData = human_integer, colData = col_data, design = ~ time)
```

    ## converting counts to integer mode

``` r
# Perform the LRT to find genes changing due to human
human_dds <- DESeq(human_dds, test = "LRT", reduced = ~ 1)
```

    ## estimating size factors

    ## estimating dispersions

    ## gene-wise dispersion estimates

    ## mean-dispersion relationship

    ## final dispersion estimates

    ## fitting model and testing

``` r
human_dds <- human_dds [  rowSums ( counts (human_dds) ) >  1 , ] 
nrow (human_dds)
```

    ## [1] 38745

``` r
#Curating all results into data frame
resultsNames(human_dds)
```

    ## [1] "Intercept"    "time_12_vs_0" "time_24_vs_0" "time_48_vs_0" "time_96_vs_0"

``` r
result_names <- resultsNames(human_dds)

results_names <- result_names[-1]

res_df <- data.frame("gene_id" = character(), 
                     "baseMean" = numeric(), 
                     "log2FoldChange" = numeric(), 
                     "lfcSE" = numeric(),
                     "stat" = numeric(),
                     "pvalue" = numeric(),
                     "padj" = numeric(),
                     "gene_name" = character(),
                     "result_name" = character())

# For loop to get all results per time point  

for(i in 1:length(results_names)) {
  results_name <- results_names[i]
  res <- results(human_dds, name = results_name)
  tmp_res_df <- res %>% as.data.frame() %>%
    rownames_to_column("gene_id") %>%
    merge(g2s) %>%
    mutate(result_name = results_name)
  res_df <- dplyr::bind_rows(res_df, tmp_res_df)
  
}

human_res_df <- res_df

save(human_dds, human_res_df, file = "/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/RESULTS/MOUSEVSHUMAN/ML/human_LRT_res_df.RData")
```

## Finding human genes that change significantly

``` r
# Calculate the maximum fold-change in any one timepoint
  human_maxfc <- human_res_df %>%
    group_by(gene_id) %>%
    summarize(max_fc = max(abs(log2FoldChange))) 
  
  # merge max shrnklfc into dataframe
  human_res_df <- human_res_df %>%
    left_join(human_maxfc)
```

    ## Joining with `by = join_by(gene_id)`

``` r
  human_res_df_padj0.05 <- human_res_df %>% 
  filter(padj <= 0.05)
  print(length(unique(human_res_df_padj0.05$gene_id)))#11116
```

    ## [1] 11116

``` r
  human_sig <- human_res_df_padj0.05 %>%
  filter(max_fc >= 1)
  print(length(unique(human_sig$gene_id)))#1937
```

    ## [1] 1937

``` r
  #compare with non LRT results
  load("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/RESULTS/HUMAN/time_point_res_df.RData")
  
  filtered_res_df <- res_df %>%
  filter(padj < 0.05, abs(log2FoldChange) > 1)
  print(length(unique(filtered_res_df$gene_id)))#2078
```

    ## [1] 2078

``` r
  filtered_res_df <- filtered_res_df %>%
  filter(!(gene_name %in% human_sig$gene_name))
  print(length(unique(filtered_res_df$gene_id)))#330
```

    ## [1] 330

``` r
  human_filtered_res_df <- filtered_res_df
  save(human_sig, human_filtered_res_df, file = "/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/RESULTS/MOUSEVSHUMAN/ML/human_sig.RData")
```

# Comparing significant human and mouse genes

## Using [2015 Dataset](https://doi.org/10.1038/s41588-023-01620-7)

Here, we use the comparison from [Hezroni et
al.](https://doi.org/10.1038/s41588-023-01620-7) to determine which
genes are conserved between the human and mouse genes.

``` r
gencode_synteny <- read.delim("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/DATA/GENCODEvGENCODE_orthologs_v3.txt", header = TRUE, stringsAsFactors = FALSE, sep = "\t")

gencode_synteny_filtered <- gencode_synteny %>% 
  filter(SpeciesWithSynteny == "[Mouse]")

gencode_synteny_filtered <- gencode_synteny_filtered %>% 
  mutate(across(everything(), ~ str_replace_all(.x, "\\[|\\]", "")))

gencode_synteny_filtered <- gencode_synteny_filtered %>%
  separate_rows(Mouse.Synteny, sep = ", ")

gencode_synteny_filtered <- gencode_synteny_filtered %>%
  mutate(across(c(GeneId, Mouse.Synteny), ~ str_replace(.x, "\\..*", "")))

#In gencode_synteny_filtered, "Mouse.Synteny" is transcript ID, need annotate with gene ID
#library(biomaRt)

# Connect to Ensembl database
#mart <- useMart("ensembl", dataset = "mmusculus_gene_ensembl")

# Retrieve transcript-to-gene mapping
#tx2gene <- getBM(attributes = c("ensembl_transcript_id", "ensembl_gene_id"),
#                 mart = mart)
#write_csv(tx2gene, file = "/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/RESULTS/MOUSEVSHUMAN/ML/tx2gene.csv")
#have probelm during knit, save tx2gene then load
tx2gene <- read_csv("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/RESULTS/MOUSEVSHUMAN/ML/tx2gene.csv")
```

    ## Rows: 278375 Columns: 2
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (2): ensembl_transcript_id, ensembl_gene_id
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
gencode_synteny_filtered <- gencode_synteny_filtered %>%
  left_join(tx2gene, by = c("Mouse.Synteny" = "ensembl_transcript_id"))

#transform to stable gene id
MS_sig <- MS_sig %>%
  mutate(across(gene_id, ~ str_replace(.x, "\\..*", "")))
human_sig <- human_sig %>%
  mutate(across(gene_id, ~ str_replace(.x, "\\..*", "")))

common_sig <- gencode_synteny_filtered %>%
  filter(GeneId %in% human_sig$gene_id,
         ensembl_gene_id %in% MS_sig$gene_id)

common_sig_filtered <- common_sig %>%
  dplyr::select(GeneId, ensembl_gene_id)

human_common_sig <- human_sig %>%
  filter(gene_id %in% common_sig$GeneId)
human_common_sig <- human_common_sig %>%
  left_join(common_sig_filtered, by = c("gene_id" = "GeneId"))

MS_common_sig <- MS_sig %>%
  filter(gene_id %in% common_sig$ensembl_gene_id)
MS_common_sig <- MS_common_sig %>%
  left_join(common_sig_filtered, by = c("gene_id" = "ensembl_gene_id"))
MS_common_sig <- MS_common_sig %>%
  dplyr::rename(ensembl_gene_id = GeneId)
#5

save(tx2gene, gencode_synteny_filtered, human_common_sig, MS_common_sig, file = "/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/RESULTS/MOUSEVSHUMAN/ML/gencode_synteny_filtered.RData")
```

## Using [2024 Dataset](https://doi.org/10.1016/j.celrep.2015.04.023)

Now, we use the comparison from [Huang et
al.](https://doi.org/10.1016/j.celrep.2015.04.023) to determine which
genes are conserved between the human and mouse genes.

``` r
synteny <- read.delim("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/DATA/human_mouse_syntenic_lncRNA.txt", header = TRUE, stringsAsFactors = FALSE, sep = "\t")
#data already tidy

common_sig <- synteny %>%
  filter(human.gene %in% human_sig$gene_id,
         mouse.gene %in% MS_sig$gene_id)

common_sig_filtered <- common_sig %>%
  dplyr::select(human.gene, mouse.gene)

human_common_sig_lncrna <- human_sig %>%
  filter(gene_id %in% common_sig$human.gene)
human_common_sig_lncrna <- human_common_sig_lncrna %>%
  left_join(common_sig_filtered, by = c("gene_id" = "human.gene"))
human_common_sig_lncrna <- human_common_sig_lncrna %>%
  dplyr::rename(ensembl_gene_id = mouse.gene)

MS_common_sig_lncrna <- MS_sig %>%
  filter(gene_id %in% common_sig$mouse.gene)
MS_common_sig_lncrna <- MS_common_sig_lncrna %>%
  left_join(common_sig_filtered, by = c("gene_id" = "mouse.gene"))
MS_common_sig_lncrna <- MS_common_sig_lncrna %>%
  dplyr::rename(ensembl_gene_id = human.gene)
#5

save(synteny, common_sig, human_common_sig_lncrna, MS_common_sig_lncrna, file = "/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/RESULTS/MOUSEVSHUMAN/ML/common_lncRNA.RData")
```

## Using bioMart

Finally, we used the bioMart function in the [Ensemble gene
browser](http://www.ensembl.org/biomart/martview/7a6e3f7aaa7334709b8e81ada35342e4)
to retrieve orthologs between the human and mouse genes.

``` r
orthologs <- read.csv("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/DATA/mart_export.txt", header = TRUE, stringsAsFactors = FALSE, sep = ",")

orthologs_filtered <- orthologs[orthologs$Mouse.homology.type == "ortholog_one2one", ]

common_orthologs <- orthologs_filtered %>%
  filter(Gene.stable.ID %in% human_sig$gene_id,
         Mouse.gene.stable.ID %in% MS_sig$gene_id)

common_orthologs_filtered <- common_orthologs %>%
  dplyr::select(Gene.stable.ID, Mouse.gene.stable.ID) %>%
  dplyr::distinct()


human_common_sig_PCG <- human_sig %>%
  filter(gene_id %in% common_orthologs$Gene.stable.ID)
print(length(unique(human_common_sig_PCG$gene_id)))
```

    ## [1] 35

``` r
human_common_sig_PCG <- human_common_sig_PCG %>%
  left_join(common_orthologs_filtered, by = c("gene_id" = "Gene.stable.ID"))
human_common_sig_PCG <- human_common_sig_PCG %>%
  dplyr::rename(ensembl_gene_id = Mouse.gene.stable.ID)

MS_common_sig_PCG <- MS_sig %>%
  filter(gene_id %in% common_orthologs$Mouse.gene.stable.ID)
print(length(unique(MS_common_sig_PCG$gene_id)))
```

    ## [1] 35

``` r
MS_common_sig_PCG <- MS_common_sig_PCG %>%
  left_join(common_orthologs_filtered, by = c("gene_id" = "Mouse.gene.stable.ID"))
MS_common_sig_PCG <- MS_common_sig_PCG %>%
  dplyr::rename(ensembl_gene_id = Gene.stable.ID)

save(orthologs_filtered, human_common_sig_PCG, MS_common_sig_PCG, file = "/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/RESULTS/MOUSEVSHUMAN/ML/common_PCG.RData")
```

## Combining the significant genes from the 2015 dataset, 2024 dataset, and the bioMart dataset so that we can generate plots with their data.

``` r
human_combined_sig <- bind_rows(human_common_sig_lncrna, human_common_sig, human_common_sig_PCG) %>%
  distinct()

MS_combined_sig <- bind_rows(MS_common_sig_lncrna, MS_common_sig, MS_common_sig_PCG) %>%
  distinct()
```

# Visualizing the data

``` r
# Load TPM data for mouse
tpm_MS <- read.table("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/DATA/MOUSE/salmon.merged.gene_tpm.tsv", header=TRUE, row.names=1)

# Load TPM data for human
tpm_human <- read.table("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/DATA/HUMAN/salmon.merged.gene_tpm.tsv", header=TRUE, row.names=1)
tpm_human <- tpm_human %>%
  dplyr::select(gene_name, gfp_0_1, gfp_0_2, gfp_0_3, gfp_12_1, gfp_12_2, gfp_12_3, gfp_24_1, gfp_24_2, gfp_24_3, gfp_48_1, gfp_48_2, gfp_48_3, gfp_96_1, gfp_96_2, gfp_96_3)

# Filter the TPM data for genes in MS/human_combined_sig
filtered_tpm_MS <- tpm_MS[tpm_MS$gene_name %in% MS_combined_sig$gene_name, ]
filtered_tpm_human <- tpm_human[tpm_human$gene_name %in% human_combined_sig$gene_name, ]

# Reshape mouse TPM data to long format
filtered_tpm_MS_long <- filtered_tpm_MS %>%
  pivot_longer(
    cols = starts_with("WT"),
    names_to = c("time_point", "replicate"),
    names_pattern = "^WT_(\\d+)_(\\d+)$",
    values_to = "tpm"
  ) %>%
  mutate(
    time_point = as.numeric(time_point),
    replicate = as.numeric(replicate),
    condition = "mouse"  # Add a column to label as mouse
  )

# Reshape human TPM data to long format
filtered_tpm_human_long <- filtered_tpm_human %>%
  pivot_longer(
    cols = starts_with("gfp"),
    names_to = c("time_point", "replicate"),
    names_pattern = "^gfp_(\\d+)_(\\d+)$",
    values_to = "tpm"
  ) %>%
  mutate(
    time_point = as.numeric(time_point),
    replicate = as.numeric(replicate),
    condition = "human"  # Add a column to label as human
  )

# Combine human and mouse data into one data frame
tpm_combined_long <- bind_rows(filtered_tpm_MS_long, filtered_tpm_human_long)

# Calculate mean and standard error (SE) for TPM at each time point and condition
tpm_mean_combined <- tpm_combined_long %>%
  group_by(gene_name, time_point, condition) %>%
  summarise(
    mean_tpm = mean(tpm, na.rm = TRUE),
    se_tpm = sd(tpm, na.rm = TRUE) / sqrt(n()),  # Standard error
    .groups = 'drop'
  )

# Plot the combined TPM values for human and mouse with error bars
ggplot(tpm_mean_combined, aes(x = time_point, y = mean_tpm)) +
  geom_line(aes(color = condition, linetype = condition), alpha = 0.7) +  # Line colored by condition
  geom_point(aes(color = condition), alpha = 0.5) +  # Points for each mean TPM value
  geom_errorbar(
    aes(ymin = mean_tpm - se_tpm, ymax = mean_tpm + se_tpm, color = condition),
    width = 0.2, # Adjust the width of the error bars
    alpha = 0.7
  ) +
  facet_wrap(~ gene_name, scales = "free_y") +  # Separate plots for each gene
  labs(
    x = "Time Point (h)", 
    y = "Mean TPM", 
    color = "Condition"
  ) +
  theme_minimal() +  # Minimal theme
  scale_color_manual(values = c("human" = "darkred", "mouse" = "blue")) +  # Manual color scale
  scale_linetype_manual(values = c("human" = "solid", "mouse" = "dashed")) + 
  scale_x_continuous(breaks = unique(tpm_mean_combined$time_point)) +  # Set x-axis breaks
  theme(
    aspect.ratio = 0.6,  # Aspect ratio of 3:2 
    strip.text = element_text(size = 12),  # Adjust facet labels font size if necessary
  )
```

<img src="CM_Project_files/figure-markdown_strict/plotting tpm-1.png" style="display: block; margin: auto;" />

## Plotting figures

``` r
# Prepare mouse data with source and timepoint column
MS_combined_sig <- MS_combined_sig %>%
  mutate(source = "mouse") %>%
  mutate(timepoint = as.numeric(sub("time_([0-9]+)_vs_0", "\\1", result_name)))
# Add a 0 time_point for mouse data
MS_combined_sig_zero <- MS_combined_sig %>%
  dplyr::select(gene_id, gene_name, baseMean) %>%
  distinct() %>%
  mutate(log2FoldChange = 0,
         timepoint = 0,
         source = "mouse")

MS_combined_sig <- MS_combined_sig %>%
  bind_rows(MS_combined_sig_zero)


# Prepare human data with source and timepoint column
human_combined_sig <- human_combined_sig %>%
  mutate(source = "human") %>%
  mutate(timepoint = as.numeric(sub("time_([0-9]+)_vs_0", "\\1", result_name)))
# Add a 0 time_point for human data
human_combined_sig_zero <- human_combined_sig %>%
  dplyr::select(gene_id, gene_name, baseMean) %>%
  distinct() %>%
  mutate(log2FoldChange = 0,
         timepoint = 0,
         source = "human")

human_combined_sig <- human_combined_sig %>%
  bind_rows(human_combined_sig_zero)

# Combine human and mouse data
combined_data <- bind_rows(MS_combined_sig, human_combined_sig)

# Generate the plot
ggplot(combined_data, aes(x = timepoint, y = log2FoldChange, group = interaction(gene_id, source), color = source)) +
  geom_hline(yintercept = 0, linetype = "dashed") +
  geom_line(alpha = 0.7, aes(linetype = source)) +
  geom_point(alpha = 0.8) +
  facet_wrap(~gene_name, scales = "free_y") +
  scale_color_manual(values = c("human" = "darkred", "mouse" = "blue")) +
  scale_linetype_manual(values = c("human" = "solid", "mouse" = "dashed")) + 
  scale_x_continuous(breaks = c(0, 12, 24, 48, 96), labels = c("0", "12", "24", "48", "96")) +
  theme_minimal() +
  labs(title = "Gene LFC Trends for human and mouse",
       x = "Timepoint (hours)",
       y = "Log2 Fold Change",
       color = "Source",
       linetype = "Source") +
   theme(
    aspect.ratio = 0.6,  # Aspect ratio of 3:2 
    strip.text = element_text(size = 12),  # Adjust facet labels font size if necessary
  )
```

<img src="CM_Project_files/figure-markdown_strict/plotting log2FC-1.png" style="display: block; margin: auto;" />

## Plotting counts

``` r
#counts data for mouse
MS_counts <- read.table("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/DATA/MOUSE/salmon.merged.gene_counts.tsv", header=TRUE, row.names=1)
#Load counts data for human
human_counts <- read.table("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/DATA/HUMAN/salmon.merged.gene_counts.tsv", header=TRUE, row.names=1)
human_counts <- human_counts %>%
  dplyr::select(gene_name, gfp_0_1, gfp_0_2, gfp_0_3, gfp_12_1, gfp_12_2, gfp_12_3, gfp_24_1, gfp_24_2, gfp_24_3, gfp_48_1, gfp_48_2, gfp_48_3, gfp_96_1, gfp_96_2, gfp_96_3)

# Filter the counts data for genes in MS/human_combined_sig
MS_counts_filtered <- MS_counts[MS_counts$gene_name %in% MS_combined_sig$gene_name, ]
human_counts_filtered <- human_counts[human_counts$gene_name %in% human_combined_sig$gene_name, ]

# Reshape mouse counts data to long format
MS_counts_filtered_long <- MS_counts_filtered %>%
  pivot_longer(
    cols = starts_with("WT"),
    names_to = c("time_point", "replicate"),
    names_pattern = "^WT_(\\d+)_(\\d+)$",
    values_to = "counts"
  ) %>%
  mutate(
    time_point = as.numeric(time_point),
    replicate = as.numeric(replicate),
    condition = "mouse"  # Add a column to label as mouse
  )

# Reshape human counts data to long format
human_counts_filtered_long <- human_counts_filtered %>%
  pivot_longer(
    cols = starts_with("gfp"),
    names_to = c("time_point", "replicate"),
    names_pattern = "^gfp_(\\d+)_(\\d+)$",
    values_to = "counts"
  ) %>%
  mutate(
    time_point = as.numeric(time_point),
    replicate = as.numeric(replicate),
    condition = "human"  # Add a column to label as human
  )

# Combine LINC00667 and GFP data into one data frame
counts_combined_long <- bind_rows(MS_counts_filtered_long, human_counts_filtered_long)

# Calculate mean and standard error (SE) for counts at each time point and condition
counts_mean_combined <- counts_combined_long %>%
  group_by(gene_name, time_point, condition) %>%
  summarise(
    mean_counts = mean(counts, na.rm = TRUE),
    se_counts = sd(counts, na.rm = TRUE) / sqrt(n()),  # Standard error
    .groups = 'drop'
  )

# Plot the combined counts values for LINC00667 and GFP with error bars
ggplot(counts_mean_combined, aes(x = time_point, y = mean_counts, group = interaction(gene_name, condition))) +
  geom_line(aes(color = condition), alpha = 0.7) +  # Line colored by condition
  geom_point(aes(color = condition), alpha = 0.5) +  # Points for each mean counts value
  geom_errorbar(
    aes(ymin = mean_counts - se_counts, ymax = mean_counts + se_counts, color = condition),
    width = 0.2, # Adjust the width of the error bars
    alpha = 0.7
  ) +
  facet_wrap(~ gene_name, scales = "free_y") +  # Separate plots for each gene
  labs(
    x = "Time Point (h)", 
    y = "Mean counts", 
    color = "Condition"
  ) +
  theme_minimal() +  # Minimal theme
  scale_color_manual(values = c("human" = "darkred", "mouse" = "blue")) +
  scale_linetype_manual(values = c("human" = "solid", "mouse" = "dashed")) +# Manual color scale
  scale_x_continuous(breaks = unique(counts_mean_combined$time_point))  # Set x-axis breaks
```

    ## Warning: No shared levels found between `names(values)` of the manual scale and the
    ## data's linetype values.

<img src="CM_Project_files/figure-markdown_strict/plotting counts-1.png" style="display: block; margin: auto;" />

# Time analysis of RNASeq Data

I want to look into how Doxycycline effects both human and mouse stem
cells throughout the duration of the experiment. There are a few methods
to accomplish this. [Oh et
al. (2021)](https://doi.org/10.3390/genes12030352) presents some of
these time analysis methods.

## Fourier Analysis

One of the most famous methods for analyzing the periodicity of time
series data is the Fourier transform. This method transforms time series
data into frequency space so that we can determine which frequency a
signal might be oscillating with. In our context, this would tell us if
doxycycline’s effects display periodic behavior over the duration of the
experiment.

Unfortunately, this analysis technique is not valid with our data. For
the Fourier transform to work, the data has to be sampled at evenly
spaced intervals. Our data was sampled at 12, 24, 48, and 96 hours.
Thankfully, a method to detect peridic signals for unevenly spaced
samples exists. This method is called the Lomb-Scargle Periodogram.

## Lomb-Scargle Periodogram

The Lomb-Scargle Periodogram is a temporal-analysis method that can be
used to estimate periodicity of unevenly spaced data. The method is
closely related to Fourier analysis and it is commonly known as
least-squares spectral analysis (LSSA).

The RNA-seq data is unevenly spaced which makes the Lomb-Scargle
Periodgram a great candidate for our analysis. A major drawback with
this technique come from

### Applying Lomb-Scargle to our counts data

``` r
# We will be using the Lomb-Scargle Periodogram instead of a Fourier Transform because the data is unevenly spaced
?lomb
```

#### First, we will calculate the Lomb-Scargle Periodogram using the TPM data

``` r
# Group by gene and condition
grouped_data <- tpm_mean_combined %>%
  group_by(gene_name, condition) %>%
  group_split()

# Compute periodograms and store frequency, power, and labels
periodogram_data <- map_dfr(grouped_data, function(group) {
  time <- group$time_point
  counts <- group$mean_tpm
  
  res <- lsp(x = counts, times = time,
             from = 1, to = 96,
             type = "period", ofac = 30, plot = FALSE)
  
  tibble(
    frequency = res$scanned,
    power = res$power,
    gene_name = unique(group$gene_name),
    condition = unique(group$condition)
  )
})


ggplot(periodogram_data, aes(x = frequency, y = power)) +
  geom_line() +
  facet_wrap(~ gene_name + condition, scales = "free_y") +
  geom_hline(yintercept = 1, linetype = "dashed", color = "blue") +  # Optional: significance line
  geom_vline(xintercept = c(8, 12, 24, 48), color = "red") + 
  labs(title = "Lomb-Scargle Periodograms", 
       x = "Period (hour)", 
       y = "Normalized Power") +
  theme_minimal() +
  theme(strip.text = element_text(size = 10, face = "bold"))
```

<img src="CM_Project_files/figure-markdown_strict/Computing for TPMs-1.png" style="display: block; margin: auto;" />

``` r
ggplot(periodogram_data, aes(x = frequency, y = power)) +
  geom_line() +
  facet_wrap(~ gene_name + condition, scales = "free_y") +
  geom_hline(yintercept = 1, linetype = "dashed", color = "blue") +  # Optional: significance line
  geom_vline(xintercept = c(8, 12, 24, 48), color = "red") + 
  labs(title = "Lomb-Scargle Periodograms Zoomed In", 
       x = "Period (hour)", 
       y = "Normalized Power") +
  xlim(0, 12) + 
  theme_minimal() +
  theme(strip.text = element_text(size = 10, face = "bold"))
```

    ## Warning: Removed 210 rows containing missing values or values outside the scale range
    ## (`geom_line()`).

    ## Warning: Removed 168 rows containing missing values or values outside the scale range
    ## (`geom_vline()`).

<img src="CM_Project_files/figure-markdown_strict/Computing for TPMs-2.png" style="display: block; margin: auto;" />

``` r
# Specify the gene name you're interested in
target_gene <- "GJB2"

# Filter the data for that gene
filtered_data <- periodogram_data %>%
  filter(gene_name == target_gene)

# Plot only for that gene using ggplot2
ggplot(filtered_data, aes(x = frequency, y = power, scales = "free_y")) +
  geom_line() +
  geom_hline(yintercept = 1, linetype = "dashed", color = "blue") +  # significance threshold + 
  geom_vline(xintercept = c(8, 12, 24, 48), color = "red") + 
  scale_x_reverse() +  # Optional: reverse the x-axis so shorter periods are on the right
  labs(title = paste("Lomb-Scargle Periodogram for", target_gene),
       x = "Period (hours)",
       y = "Normalized Power")
```

<img src="CM_Project_files/figure-markdown_strict/Case gene: GJB2 with TPMs-1.png" style="display: block; margin: auto;" />

``` r
  theme(strip.text = element_text(size = 10, face = "bold"))
```

    ## List of 1
    ##  $ strip.text:List of 11
    ##   ..$ family       : NULL
    ##   ..$ face         : chr "bold"
    ##   ..$ colour       : NULL
    ##   ..$ size         : num 10
    ##   ..$ hjust        : NULL
    ##   ..$ vjust        : NULL
    ##   ..$ angle        : NULL
    ##   ..$ lineheight   : NULL
    ##   ..$ margin       : NULL
    ##   ..$ debug        : NULL
    ##   ..$ inherit.blank: logi FALSE
    ##   ..- attr(*, "class")= chr [1:2] "element_text" "element"
    ##  - attr(*, "class")= chr [1:2] "theme" "gg"
    ##  - attr(*, "complete")= logi FALSE
    ##  - attr(*, "validate")= logi TRUE

#### Now using the counts data

``` r
# Group by gene and condition
grouped_data <- counts_mean_combined %>%
  group_by(gene_name, condition) %>%
  group_split()

# Compute periodograms and store frequency, power, and labels
periodogram_data <- map_dfr(grouped_data, function(group) {
  time <- group$time_point
  counts <- group$mean_counts
  
  res <- lsp(x = counts, times = time,
             from = 1, to = 96,
             type = "period", ofac = 30, plot = FALSE)
  
  tibble(
    frequency = res$scanned,
    power = res$power,
    gene_name = unique(group$gene_name),
    condition = unique(group$condition)
  )
})


ggplot(periodogram_data, aes(x = frequency, y = power)) +
  geom_line() +
  facet_wrap(~ gene_name + condition, scales = "free_y") +
  geom_hline(yintercept = 1, linetype = "dashed", color = "blue") +  # Optional: significance line
  geom_vline(xintercept = c(8, 12, 24, 48), color = "red") + 
  labs(title = "Lomb-Scargle Periodograms", 
       x = "Period (hour)", 
       y = "Normalized Power") +
  theme_minimal() +
  theme(strip.text = element_text(size = 10, face = "bold"))
```

<img src="CM_Project_files/figure-markdown_strict/Computing for counts-1.png" style="display: block; margin: auto;" />

``` r
ggplot(periodogram_data, aes(x = frequency, y = power)) +
  geom_line() +
  facet_wrap(~ gene_name + condition, scales = "free_y") +
  geom_hline(yintercept = 1, linetype = "dashed", color = "blue") +  # Optional: significance line
  geom_vline(xintercept = c(8, 12, 24, 48), color = "red") + 
  labs(title = "Lomb-Scargle Periodograms Zoomed In", 
       x = "Period (hour)", 
       y = "Normalized Power") +
  xlim(0, 12) + 
  theme_minimal() +
  theme(strip.text = element_text(size = 10, face = "bold"))
```

    ## Warning: Removed 210 rows containing missing values or values outside the scale range
    ## (`geom_line()`).

    ## Warning: Removed 168 rows containing missing values or values outside the scale range
    ## (`geom_vline()`).

<img src="CM_Project_files/figure-markdown_strict/Computing for counts-2.png" style="display: block; margin: auto;" />

``` r
# Specify the gene name you're interested in
target_gene <- "GJB2"

# Filter the data for that gene
filtered_data <- periodogram_data %>%
  filter(gene_name == target_gene)

# Plot only for that gene using ggplot2
ggplot(filtered_data, aes(x = frequency, y = power, scales = "free_y")) +
  geom_line() +
  geom_hline(yintercept = 1, linetype = "dashed", color = "blue") +  # significance threshold + 
  geom_vline(xintercept = c(8, 12, 24, 48), color = "red") + 
  scale_x_reverse() +  # Optional: reverse the x-axis so shorter periods are on the right
  labs(title = paste("Lomb-Scargle Periodogram for", target_gene),
       x = "Period (hours)",
       y = "Normalized Power")
```

<img src="CM_Project_files/figure-markdown_strict/Case gene: GJB2 with counts-1.png" style="display: block; margin: auto;" />

``` r
  theme(strip.text = element_text(size = 10, face = "bold"))
```

    ## List of 1
    ##  $ strip.text:List of 11
    ##   ..$ family       : NULL
    ##   ..$ face         : chr "bold"
    ##   ..$ colour       : NULL
    ##   ..$ size         : num 10
    ##   ..$ hjust        : NULL
    ##   ..$ vjust        : NULL
    ##   ..$ angle        : NULL
    ##   ..$ lineheight   : NULL
    ##   ..$ margin       : NULL
    ##   ..$ debug        : NULL
    ##   ..$ inherit.blank: logi FALSE
    ##   ..- attr(*, "class")= chr [1:2] "element_text" "element"
    ##  - attr(*, "class")= chr [1:2] "theme" "gg"
    ##  - attr(*, "complete")= logi FALSE
    ##  - attr(*, "validate")= logi TRUE

### Fail

These were lame. Th The results are not good enough to make any
conclusions about the data. The combination of irregularly spaced and
too few points gives pretty bad results. The human data has a lot more
time points than what was shown above. They still aren’t evenly spaced
so we will still be using the Lomb-Scargle Periodogram. We’ll add these
time points and see if that helps our results.

#### Adding in all time points from the human data

``` r
#counts data for mouse
MS_counts <- read.table("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/DATA/MOUSE/salmon.merged.gene_counts.tsv", header=TRUE, row.names=1)
#Load counts data for human
all_human_tp_counts <- read.table("/scratch/Shares/rinnclass/MASTER_CLASS/STUDENTS/genehomies/DATA/HUMAN/salmon.merged.gene_counts.tsv", header=TRUE, row.names=1)
all_human_tp_counts <- all_human_tp_counts %>%
  dplyr::select(gene_name, gfp_0_1, gfp_0_2, gfp_0_3, gfp_0.5_1, gfp_0.5_2, gfp_1_1, gfp_1_2,  gfp_1.5_1, gfp_1.5_2, gfp_2_1, gfp_2_2, gfp_2.5_1, gfp_2.5_2, gfp_3_1, gfp_3_2, gfp_3.5_1, gfp_3.5_2, gfp_4_1, gfp_4_2, gfp_4.5_1, gfp_4.5_2, gfp_5_1, gfp_5_2, gfp_5.5_1, gfp_5.5_2, gfp_12_1, gfp_12_2, gfp_12_3, gfp_24_1, gfp_24_2, gfp_24_3, gfp_48_1, gfp_48_2, gfp_48_3, gfp_96_1, gfp_96_2, gfp_96_3)

# Filter the counts data for genes in MS/human_combined_sig
MS_counts_filtered <- MS_counts[MS_counts$gene_name %in% MS_combined_sig$gene_name, ]
all_human_tp_counts_filtered <- all_human_tp_counts[human_counts$gene_name %in% human_combined_sig$gene_name, ]

# Reshape mouse counts data to long format
MS_counts_filtered_long <- MS_counts_filtered %>%
  pivot_longer(
    cols = starts_with("WT"),
    names_to = c("time_point", "replicate"),
    names_pattern = "^WT_([0-9.]+)_([0-9]+)$",
    values_to = "counts"
  ) %>%
  mutate(
    time_point = as.numeric(time_point),
    replicate = as.numeric(replicate),
    condition = "mouse"  # Add a column to label as mouse
  )

# Reshape human counts data to long format
ahtp_counts_filtered_long <- all_human_tp_counts_filtered %>%
  pivot_longer(
    cols = starts_with("gfp"),
    names_to = c("time_point", "replicate"),
    names_pattern = "^gfp_([0-9.]+)_([0-9]+)$",
    values_to = "counts"
  ) %>%
  mutate(
    time_point = as.numeric(time_point),
    replicate = as.numeric(replicate),
    condition = "human"  # Add a column to label as human
  )

# Combine LINC00667 and GFP data into one data frame
counts_combined_long <- bind_rows(human_counts_filtered_long, ahtp_counts_filtered_long)

# Calculate mean and standard error (SE) for counts at each time point and condition
counts_mean_combined <- counts_combined_long %>%
  group_by(gene_name, time_point, condition) %>%
  summarise(
    mean_counts = mean(counts, na.rm = TRUE),
    se_counts = sd(counts, na.rm = TRUE) / sqrt(n()),  # Standard error
    .groups = 'drop'
  )

# Plot the combined counts values for LINC00667 and GFP with error bars
ggplot(counts_mean_combined, aes(x = time_point, y = mean_counts, group = interaction(gene_name, condition))) +
  geom_line(aes(color = condition), alpha = 0.7) +  # Line colored by condition
  geom_point(aes(color = condition), alpha = 0.5) +  # Points for each mean counts value
  geom_errorbar(
    aes(ymin = mean_counts - se_counts, ymax = mean_counts + se_counts, color = condition),
    width = 0.2, # Adjust the width of the error bars
    alpha = 0.7
  ) +
  facet_wrap(~ gene_name, scales = "free_y") +  # Separate plots for each gene
  labs(
    x = "Time Point (h)", 
    y = "Mean counts", 
    color = "Condition"
  ) +
  theme_minimal() +  # Minimal theme
  scale_color_manual(values = c("human" = "darkred", "human" = "blue")) +
  scale_linetype_manual(values = c("human" = "solid", "human" = "dashed")) +# Manual color scale
  scale_x_continuous(breaks = unique(counts_mean_combined$time_point))  # Set x-axis breaks
```

    ## Warning: No shared levels found between `names(values)` of the manual scale and the
    ## data's linetype values.

<img src="CM_Project_files/figure-markdown_strict/Plotting counts for all time points in the human data-1.png" style="display: block; margin: auto;" />

``` r
#### Now using the counts data
# Group by gene and condition
grouped_data <- counts_mean_combined %>%
  group_by(gene_name, condition) %>%
  group_split()

# Compute periodograms and store frequency, power, and labels
periodogram_data <- map_dfr(grouped_data, function(group) {
  time <- group$time_point
  counts <- group$mean_counts
  
  res <- lsp(x = counts, times = time,
             from = 1, to = 96,
             type = "period", ofac = 30, plot = FALSE)
  
  tibble(
    frequency = res$scanned,
    power = res$power,
    gene_name = unique(group$gene_name),
    condition = unique(group$condition)
  )
})


ggplot(periodogram_data, aes(x = frequency, y = power)) +
  geom_line() +
  facet_wrap(~ gene_name + condition, scales = "free_y") +
  geom_hline(yintercept = 1, linetype = "dashed", color = "blue") +  # Optional: significance line
  geom_vline(xintercept = c(8, 12, 24, 48), color = "red") + 
  labs(title = "Lomb-Scargle Periodograms", 
       x = "Period (hour)", 
       y = "Normalized Power") +
  theme_minimal() +
  theme(strip.text = element_text(size = 10, face = "bold"))
```

<img src="CM_Project_files/figure-markdown_strict/Plotting counts for all time points in the human data-2.png" style="display: block; margin: auto;" />

``` r
ggplot(periodogram_data, aes(x = frequency, y = power)) +
  geom_line() +
  facet_wrap(~ gene_name + condition, scales = "free_y") +
  geom_hline(yintercept = 1, linetype = "dashed", color = "blue") +  # Optional: significance line
  geom_vline(xintercept = c(8, 12, 24, 48), color = "red") + 
  labs(title = "Lomb-Scargle Periodograms Zoomed In", 
       x = "Period (hour)", 
       y = "Normalized Power") +
  xlim(0, 12) + 
  theme_minimal() +
  theme(strip.text = element_text(size = 10, face = "bold"))
```

    ## Warning: Removed 210 rows containing missing values or values outside the scale range
    ## (`geom_line()`).

    ## Warning: Removed 84 rows containing missing values or values outside the scale range
    ## (`geom_vline()`).

<img src="CM_Project_files/figure-markdown_strict/Plotting counts for all time points in the human data-3.png" style="display: block; margin: auto;" />

``` r
# Specify the gene name you're interested in
target_gene <- "GJB2"

# Filter the data for that gene
filtered_data <- periodogram_data %>%
  filter(gene_name == target_gene)

# Plot only for that gene using ggplot2
ggplot(filtered_data, aes(x = frequency, y = power, scales = "free_y")) +
  geom_line() +
  geom_hline(yintercept = 1, linetype = "dashed", color = "blue") +  # significance threshold + 
  geom_vline(xintercept = c(8, 12, 24, 48), color = "red") + 
  scale_x_reverse() +  # Optional: reverse the x-axis so shorter periods are on the right
  labs(title = paste("Lomb-Scargle Periodogram for", target_gene),
       x = "Period (hours)",
       y = "Normalized Power")
```

<img src="CM_Project_files/figure-markdown_strict/Case gene: GJB2 with new time points-1.png" style="display: block; margin: auto;" />

``` r
  theme(strip.text = element_text(size = 10, face = "bold"))
```

    ## List of 1
    ##  $ strip.text:List of 11
    ##   ..$ family       : NULL
    ##   ..$ face         : chr "bold"
    ##   ..$ colour       : NULL
    ##   ..$ size         : num 10
    ##   ..$ hjust        : NULL
    ##   ..$ vjust        : NULL
    ##   ..$ angle        : NULL
    ##   ..$ lineheight   : NULL
    ##   ..$ margin       : NULL
    ##   ..$ debug        : NULL
    ##   ..$ inherit.blank: logi FALSE
    ##   ..- attr(*, "class")= chr [1:2] "element_text" "element"
    ##  - attr(*, "class")= chr [1:2] "theme" "gg"
    ##  - attr(*, "complete")= logi FALSE
    ##  - attr(*, "validate")= logi TRUE

### Results

The data results look a lot better with the extra data points. The noise
at lower frequencies has significantly decreased and peaks later on are
seemingly more prominent. Some of the genes are clearly periodic in the
dataset. [RP11-478C19.2](https://doi.org/10.1016/j.tranon.2022.101525),
[SP5](https://www.sciencedirect.com/science/article/pii/S0006291X05027658),
and [ADGRL4](https://pmc.ncbi.nlm.nih.gov/articles/PMC6626334/) are
great examples of this behavior.

I would want to do this with more data points and see what it looks like
after that. It would also be interesting to examine the genes in more
depth and see if there is some kind of correlation between function and
periodicity.
