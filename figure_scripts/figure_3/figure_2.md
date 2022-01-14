Figure 2
================

``` r
library(reticulate)
use_python("/home/nealpsmith/.conda/envs/sc_analysis/bin/python")
```

**For figure 2, we wanted to focus in on the T cells. We subsetted our main data object to just the T cell clusters. From there we had a few goals : Find heterogeneity among the T cells, determine which T cell clusters are enriched in AA and ANA and look for DEGs among the T cells.**

First, we isolated just the T cells from our original data object. NK cells were segregated and removed. Next, we wanted to determine the optimal number of clusters for our T cells.

``` python
import pegasus as pg
import numpy as np
import pandas as pd
import concurrent.futures
from sklearn.metrics.cluster import adjusted_rand_score
import random
import time
import leidenalg
import concurrent.futures
import os
from pegasus.tools import construct_graph
from scipy.sparse import csr_matrix

# Use Rand index to determine leiden resolution to use
def rand_index_plot(
        W,  # adata.uns['W_' + rep] or adata.uns['neighbors']
        resamp_perc=0.9,
        resolutions=(0.3, 0.5, 0.7, 0.9, 1.1, 1.3, 1.5, 1.7, 1.9),
        max_workers=25,
        n_samples=25,
        random_state=0
    ):
    assert isinstance(W, csr_matrix)
    rand_indx_dict = {}
    n_cells = W.shape[0]
    resamp_size = round(n_cells * resamp_perc)

    for resolution in resolutions:

        true_class = leiden(W, resolution, random_state)

        with concurrent.futures.ProcessPoolExecutor(max_workers=max_workers) as executor:
            futures = [executor.submit(_collect_samples, W, resolution, n_cells, resamp_size, true_class, random_state)
                       for i in range(n_samples)]
            rand_list = [f.result() for f in futures]

        rand_indx_dict[str(resolution)] = rand_list
        print("Finished {res}".format(res=resolution))
    return rand_indx_dict
def leiden(W, resolution, random_state=0):

    start = time.perf_counter()

    G = construct_graph(W)
    partition_type = leidenalg.RBConfigurationVertexPartition
    partition = leidenalg.find_partition(
        G,
        partition_type,
        seed=random_state,
        weights="weight",
        resolution_parameter=resolution,
        n_iterations=-1,
    )

    labels = np.array([str(x + 1) for x in partition.membership])

    end = time.perf_counter()
    n_clusters = len(set(labels))
    return pd.Series(labels)

def _collect_samples(W, resolution, n_cells, resamp_size, true_class, random_state=0):
    samp_indx = random.sample(range(n_cells), resamp_size)
    samp_data = W[samp_indx][:, samp_indx]
    true_class = true_class[samp_indx]
    new_class = leiden(samp_data, resolution, random_state)
    return adjusted_rand_score(true_class, new_class)
```

<img src="figure_2_files/figure-markdown_github/rand_func-1.png" width="1152" />

``` python
import matplotlib.pyplot as plt
import seaborn as sns

t_cell_harmonized = pg.read_input("/home/nealpsmith/projects/medoff/data/t_cell_harmonized.h5ad")

# rand_indx_dict = rand_index_plot(W = t_cell_harmonized.uns["W_pca_harmony"],
#                                       resolutions  = [0.3, 0.5, 0.7, 0.9, 1.1, 1.3, 1.5, 1.7, 1.9],
#                                       n_samples = 2)
#
# plot_df = pd.DataFrame(rand_indx_dict).T
# plot_df = plot_df.reset_index()
# plot_df = plot_df.melt(id_vars="index")
# plot_df.to_csv(os.path.join(file_path(), "data", "ari_plots", "t_cell_harmonized_ARI.csv"))
plot_df = pd.read_csv("/home/nealpsmith/projects/medoff/data/ari_plots/t_cell_harmonized_ARI.csv")
fig, ax = plt.subplots(1)
_ = sns.boxplot(x="index", y="value", data=plot_df, ax = ax)
for box in ax.artists:
    box.set_facecolor("grey")
ax.artists[5].set_facecolor("red") # The one we chose!
ax.spines['right'].set_visible(False)
ax.spines['top'].set_visible(False)
ax.tick_params(axis='both', which='major', labelsize=15)
_ = ax.set_ylabel("Adjusted Rand index", size = 20)
_ = ax.set_xlabel("leiden resolution", size = 20)
_ = plt.axhline(y = 0.9, color = "black", linestyle = "--")
fig.tight_layout()
fig
```

<img src="figure_2_files/figure-markdown_github/rand_plot-3.png" width="672" />

Based on this rand index approach, we can see that a leiden resolution of 1.3 is the highest resolution where the soultion remains quite stable. Given this, we went forward with clustering at this resolution. We can immediately appreciate this seperation of CD4 and CD8 clusters

``` python
import scanpy as sc
import matplotlib.colors as clr
colormap = clr.LinearSegmentedColormap.from_list('gene_cmap', ["#d3d3d3" ,'#482cc7'], N=200)

# pg.leiden(t_cell_harmonized, resolution = 1.3, rep = "pca_harmony")

figure = sc.pl.umap(t_cell_harmonized, color = ["leiden_labels", "CD4", "CD8A"],
                    return_fig = True, show = False, legend_loc = "on data", ncols = 3,
                    wspace = 0.3, cmap = colormap)
figure.set_size_inches(12, 4)
figure
```

<img src="figure_2_files/figure-markdown_github/clustering-5.png" width="1152" />

# Marker genes

First we can look at marker genes by AUROC. The motivation here is to determine for each cluster which specific genes are good classifiers for cluster membership. These stats were calculated using the Pegasus `de_analysis` function.

``` python
# pg.de_analysis(t_cell_harmonized, cluster = "leiden_labels", auc = True,
#                n_jobs = len(set(t_cell_harmonized.obs["leiden_labels"])))

top_auc = {}
top_genes = {}
for clust in sorted(set(t_cell_harmonized.obs["leiden_labels"]), key = int) :
    df_dict = {"auc": t_cell_harmonized.varm["de_res"]["auroc:{clust}".format(clust=clust)]}
    df = pd.DataFrame(df_dict, index=t_cell_harmonized.var.index)
    genes = df[df["auc"] >= 0.75].index.values
    top_genes[clust] = genes

top_gene_df = pd.DataFrame(dict([(k,pd.Series(v)) for k,v in top_genes.items() ]))
top_gene_df = top_gene_df.rename(columns = {clust : "cluster_{clust}".format(clust=clust) for clust in top_genes.keys()})
top_gene_df = top_gene_df.replace(np.nan, "")
```

<img src="figure_2_files/figure-markdown_github/DE_analysis-7.png" width="1152" />

``` r
library(knitr)
kable(reticulate::py$top_gene_df, caption = "genes with AUC > 0.75")
```

<table>
<caption>genes with AUC &gt; 0.75</caption>
<colgroup>
<col width="9%" />
<col width="9%" />
<col width="9%" />
<col width="9%" />
<col width="9%" />
<col width="9%" />
<col width="9%" />
<col width="9%" />
<col width="9%" />
<col width="10%" />
</colgroup>
<thead>
<tr class="header">
<th align="left">cluster_1</th>
<th align="left">cluster_2</th>
<th align="left">cluster_3</th>
<th align="left">cluster_4</th>
<th align="left">cluster_5</th>
<th align="left">cluster_6</th>
<th align="left">cluster_7</th>
<th align="left">cluster_8</th>
<th align="left">cluster_9</th>
<th align="left">cluster_10</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">CD8A</td>
<td align="left">RPS8</td>
<td align="left">MALAT1</td>
<td align="left">IL7R</td>
<td align="left">GNLY</td>
<td align="left">TNFRSF4</td>
<td align="left">GZMK</td>
<td align="left">TNFRSF18</td>
<td align="left">SRGN</td>
<td align="left">ISG15</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPL32</td>
<td align="left"></td>
<td align="left">KLRB1</td>
<td align="left">HOPX</td>
<td align="left">TMSB10</td>
<td align="left"></td>
<td align="left">TNFRSF4</td>
<td align="left">CD69</td>
<td align="left">IFI6</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">RPL34</td>
<td align="left"></td>
<td align="left">RPLP1</td>
<td align="left">KLRD1</td>
<td align="left">LTB</td>
<td align="left"></td>
<td align="left">CHDH</td>
<td align="left">CCL4</td>
<td align="left">IFI44L</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPS23</td>
<td align="left"></td>
<td align="left"></td>
<td align="left">KLRC1</td>
<td align="left">IL2RA</td>
<td align="left"></td>
<td align="left">IL17RB</td>
<td align="left">CCL4L2</td>
<td align="left">RSAD2</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">EEF1A1</td>
<td align="left"></td>
<td align="left"></td>
<td align="left">CD63</td>
<td align="left"></td>
<td align="left"></td>
<td align="left">IL13</td>
<td align="left"></td>
<td align="left">EIF2AK2</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPS12</td>
<td align="left"></td>
<td align="left"></td>
<td align="left">CCL5</td>
<td align="left"></td>
<td align="left"></td>
<td align="left">HSP90AB1</td>
<td align="left"></td>
<td align="left">HERC5</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">RPL39</td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left">GATA3</td>
<td align="left"></td>
<td align="left">LY6E</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPL10</td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left">RPLP0</td>
<td align="left"></td>
<td align="left">IFIT1</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">RPL30</td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left">ISG20</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPS6</td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left">XAF1</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">TPT1</td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left">MX1</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPL13</td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
<td align="left"></td>
</tr>
</tbody>
</table>

We can see from the above AUROC genes, that we don't have a strong enough signal from some clusters to get a good sense of their phenotype solely on that. So we can also find markers using an OVA pseudobulk approach. To do this, we first created a psedudobulk matrix by summing the UMI counts across cells for each unique cluster/sample combination, creating a matrix of n genes x (n samples \* n clusters). Using this matrix with DESeq2, For each cluster, we used an input model gene ~ in\_clust where in\_clust is a factor with two levels indicating if the sample was in or not in the cluster being tested. Genes with an FDR &lt; 5% were considered marker genes.

``` python
# import neals_python_functions as nealsucks
# # Read in the raw count data
# raw_data = pg.read_input("/home/nealpsmith/projects/medoff/data/all_data.h5sc")
# raw_data = raw_data[t_cell_harmonized.obs_names]
# raw_data = raw_data[:, t_cell_harmonized.var_names]
# raw_data.obs = t_cell_harmonized.obs[["leiden_labels", "Channel"]]
#
# # Create the matrix
# raw_sum_dict = {}
# cell_num_dict = {}
# for samp in set(raw_data.obs["Channel"]):
#     for clust in set(raw_data.obs["leiden_labels"]):
#         dat = raw_data[(raw_data.obs["Channel"] == samp) & (raw_data.obs["leiden_labels"] == clust)]
#         if len(dat) == 0:
#             continue
#         cell_num_dict["samp_{samp}_{clust}".format(samp=samp, clust=clust)] = len(dat)
#         count_sum = np.array(dat.X.sum(axis=0)).flatten()
#         raw_sum_dict["samp_{samp}_{clust}".format(samp=samp, clust=clust)] = count_sum
#
# count_mtx = pd.DataFrame(raw_sum_dict, index=raw_data.var.index.values)
#
# meta_df = pd.DataFrame(cell_num_dict, index=["n_cells"]).T
# meta_df["cluster"] = [name.split("_")[-1] for name in meta_df.index.values]
# meta_df["sample"] = [name.split("_")[-2] for name in meta_df.index.values]
# meta_df["phenotype"] = [name.split("_")[-3] for name in meta_df.index.values]
# meta_df["id"] = ["_".join(name.split("_")[0:2]) for name in meta_df.index.values]
#
# clust_df = pd.DataFrame(index=count_mtx.index)
# # Lets run pseudobulk on clusters
# for clust in set(t_cell_harmonized.obs["leiden_labels"]):
#     print(clust)
#     meta_temp = meta_df.copy()
#     meta_temp["isclust"] = ["yes" if cluster == clust else "no" for cluster in meta_temp["cluster"]]
#
#     assert all(meta_temp.index.values == count_mtx.columns)
#     # Run DESeq2
#     deseq = nealsucks.analysis.deseq2.py_DESeq2(count_matrix=count_mtx, design_matrix=meta_temp,
#                                                 design_formula="~ isclust")
#     deseq.run_deseq()
#     res = deseq.get_deseq_result()
#     clust_df = clust_df.join(res[["pvalue"]].rename(
#         columns={"pvalue": "pseudobulk_p_val:{clust}".format(clust=clust)}))

de_res = t_cell_harmonized.varm["de_res"]
# de_res = pd.DataFrame(de_res, index=res.index)
# de_res = de_res.join(clust_df)
de_res = pd.DataFrame(de_res, index = t_cell_harmonized.var_names)
de_res = de_res.fillna(0)
names = [name for name in de_res.columns if name.startswith("pseudobulk_p_val")]

import statsmodels.stats.multitest as stats
for name in names :
    clust = name.split(":")[1]
    de_res["pseudobulk_q_val:{clust}".format(clust = clust)] = stats.fdrcorrection(de_res[name])[1]

de_res = de_res.to_records(index=False)
t_cell_harmonized.varm["de_res"] = de_res

top_genes = {}
for clust in sorted(set(t_cell_harmonized.obs["leiden_labels"]), key = int) :
    df_dict = {"auc": t_cell_harmonized.varm["de_res"]["auroc:{clust}".format(clust=clust)],
               "pseudo_q" : t_cell_harmonized.varm["de_res"]["pseudobulk_q_val:{clust}".format(clust = clust)],
               "pseudo_p" : t_cell_harmonized.varm["de_res"]["pseudobulk_p_val:{clust}".format(clust = clust)],
               "pseudo_log_fc" : t_cell_harmonized.varm["de_res"]["pseudobulk_log_fold_change:{clust}".format(clust = clust)],
               "percent" : t_cell_harmonized.varm["de_res"]["percentage:{clust}".format(clust = clust)]}
    df = pd.DataFrame(df_dict, index=t_cell_harmonized.var.index)
    # Lets limit to genes where at least 20% cells express it
    df = df[df["percent"] > 20]
    # df = df.sort_values(by=["auc"], ascending=False)
    # df = df.iloc[0:15]
    # genes = df.index.values
    # Get top 50 genes (first by AUC, then by pseudobulk)
    genes = df[df["auc"] >= 0.75].index.values

    n_from_pseudo = 50 - len(genes)
    if n_from_pseudo > 0 :
        # Dont want to repeat genes
        pseudobulk = df.drop(genes)
        pseudobulk = pseudobulk[(pseudobulk["pseudo_q"] < 0.05)]
        pseudobulk = pseudobulk.sort_values(by = "pseudo_log_fc", ascending = False).iloc[0:n_from_pseudo,:].index.values
        pseudobulk = [name for name in pseudobulk if name not in genes]
        genes = np.concatenate((genes, pseudobulk))

    print("Cluster {clust}: {length}".format(clust = clust, length = len(genes)))
    top_genes[clust] = genes

top_gene_df = pd.DataFrame(dict([(k,pd.Series(v)) for k,v in top_genes.items() ]))
top_gene_df = top_gene_df.rename(columns = {clust : "cluster_{clust}".format(clust=clust) for clust in top_genes.keys()})
top_gene_df = top_gene_df.replace(np.nan, "")
```

<img src="figure_2_files/figure-markdown_github/pseudobulk-1.png" width="1152" />

``` r
kable(reticulate::py$top_gene_df, caption = "genes with AUC> 0.75 or pseudo q < 0.05")
```

<table>
<caption>genes with AUC&gt; 0.75 or pseudo q &lt; 0.05</caption>
<colgroup>
<col width="9%" />
<col width="9%" />
<col width="11%" />
<col width="9%" />
<col width="9%" />
<col width="11%" />
<col width="9%" />
<col width="9%" />
<col width="9%" />
<col width="10%" />
</colgroup>
<thead>
<tr class="header">
<th align="left">cluster_1</th>
<th align="left">cluster_2</th>
<th align="left">cluster_3</th>
<th align="left">cluster_4</th>
<th align="left">cluster_5</th>
<th align="left">cluster_6</th>
<th align="left">cluster_7</th>
<th align="left">cluster_8</th>
<th align="left">cluster_9</th>
<th align="left">cluster_10</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">CD8A</td>
<td align="left">RPS8</td>
<td align="left">MALAT1</td>
<td align="left">IL7R</td>
<td align="left">GNLY</td>
<td align="left">TNFRSF4</td>
<td align="left">GZMK</td>
<td align="left">TNFRSF18</td>
<td align="left">SRGN</td>
<td align="left">ISG15</td>
</tr>
<tr class="even">
<td align="left">CLIC3</td>
<td align="left">RPL32</td>
<td align="left">GPR155</td>
<td align="left">KLRB1</td>
<td align="left">HOPX</td>
<td align="left">TMSB10</td>
<td align="left">EOMES</td>
<td align="left">TNFRSF4</td>
<td align="left">CD69</td>
<td align="left">IFI6</td>
</tr>
<tr class="odd">
<td align="left">THEMIS</td>
<td align="left">RPL34</td>
<td align="left">ITGA1</td>
<td align="left">RPLP1</td>
<td align="left">KLRD1</td>
<td align="left">LTB</td>
<td align="left">HLA-DQA1</td>
<td align="left">CHDH</td>
<td align="left">CCL4</td>
<td align="left">IFI44L</td>
</tr>
<tr class="even">
<td align="left">FKBP11</td>
<td align="left">RPS23</td>
<td align="left">SMURF2</td>
<td align="left">SLC4A10</td>
<td align="left">KLRC1</td>
<td align="left">IL2RA</td>
<td align="left">HLA-DRA</td>
<td align="left">IL17RB</td>
<td align="left">CCL4L2</td>
<td align="left">RSAD2</td>
</tr>
<tr class="odd">
<td align="left">RFLNB</td>
<td align="left">EEF1A1</td>
<td align="left">CD226</td>
<td align="left">IL4I1</td>
<td align="left">CD63</td>
<td align="left">FOXP3</td>
<td align="left">DTHD1</td>
<td align="left">IL13</td>
<td align="left">EGR2</td>
<td align="left">EIF2AK2</td>
</tr>
<tr class="even">
<td align="left">RGL4</td>
<td align="left">RPS12</td>
<td align="left">THEMIS</td>
<td align="left">TNFSF13B</td>
<td align="left">CCL5</td>
<td align="left">CTLA4</td>
<td align="left">KLRG1</td>
<td align="left">HSP90AB1</td>
<td align="left">EGR3</td>
<td align="left">HERC5</td>
</tr>
<tr class="odd">
<td align="left">LGALS1</td>
<td align="left">RPL39</td>
<td align="left">THUMPD3-AS1</td>
<td align="left">LST1</td>
<td align="left">KIR2DL4</td>
<td align="left">RTKN2</td>
<td align="left">HLA-DQB1</td>
<td align="left">GATA3</td>
<td align="left">CCL3</td>
<td align="left">LY6E</td>
</tr>
<tr class="even">
<td align="left">CD2</td>
<td align="left">RPL10</td>
<td align="left">AC245297.3</td>
<td align="left">NCR3</td>
<td align="left">TRDC</td>
<td align="left">TBC1D4</td>
<td align="left">GZMH</td>
<td align="left">RPLP0</td>
<td align="left">CCL3L1</td>
<td align="left">IFIT1</td>
</tr>
<tr class="odd">
<td align="left">RNF167</td>
<td align="left">RPL30</td>
<td align="left">ITGAE</td>
<td align="left">PRR5</td>
<td align="left">CLNK</td>
<td align="left">TNFRSF18</td>
<td align="left">HLA-DRB1</td>
<td align="left">IL5</td>
<td align="left">NR4A1</td>
<td align="left">ISG20</td>
</tr>
<tr class="even">
<td align="left">CD3G</td>
<td align="left">RPS6</td>
<td align="left">GRAP2</td>
<td align="left">CCR6</td>
<td align="left">CXXC5</td>
<td align="left">IL1R1</td>
<td align="left">HLA-DRB5</td>
<td align="left">HPGDS</td>
<td align="left">XCL1</td>
<td align="left">XAF1</td>
</tr>
<tr class="odd">
<td align="left">COMMD8</td>
<td align="left">TPT1</td>
<td align="left">CD8A</td>
<td align="left">TMIGD2</td>
<td align="left">PIK3AP1</td>
<td align="left">LINC01943</td>
<td align="left">SAMD3</td>
<td align="left">IL4</td>
<td align="left">XCL2</td>
<td align="left">MX1</td>
</tr>
<tr class="even">
<td align="left">UBTF</td>
<td align="left">RPL13</td>
<td align="left">SUN2</td>
<td align="left">NRIP1</td>
<td align="left">LINC02446</td>
<td align="left">LTA</td>
<td align="left">HLA-DPB1</td>
<td align="left">PTGDR2</td>
<td align="left">NFKBID</td>
<td align="left">IFIT3</td>
</tr>
<tr class="odd">
<td align="left">S100A4</td>
<td align="left">MAL</td>
<td align="left">AC004687.1</td>
<td align="left">CEBPD</td>
<td align="left">KLRC2</td>
<td align="left">HPGD</td>
<td align="left">CMC1</td>
<td align="left">KRT1</td>
<td align="left">EGR1</td>
<td align="left">CMPK2</td>
</tr>
<tr class="even">
<td align="left">S100A10</td>
<td align="left">CD40LG</td>
<td align="left">TTC14</td>
<td align="left">CTSH</td>
<td align="left">IKZF2</td>
<td align="left">HAPLN3</td>
<td align="left">CST7</td>
<td align="left">CACNA1D</td>
<td align="left">DUSP2</td>
<td align="left">OAS1</td>
</tr>
<tr class="odd">
<td align="left">HMOX2</td>
<td align="left">CCR7</td>
<td align="left">SPN</td>
<td align="left">DPP4</td>
<td align="left">CAPN12</td>
<td align="left">ZC3H12D</td>
<td align="left">SH2D1A</td>
<td align="left">GATA3-AS1</td>
<td align="left">TNF</td>
<td align="left">OAS3</td>
</tr>
<tr class="even">
<td align="left">RAB8A</td>
<td align="left">C1orf162</td>
<td align="left">PARP8</td>
<td align="left">IFI44</td>
<td align="left">SLC12A6</td>
<td align="left">CSGALNACT1</td>
<td align="left">CD27</td>
<td align="left">LIF</td>
<td align="left">CD160</td>
<td align="left">IFIT2</td>
</tr>
<tr class="odd">
<td align="left">MYL12A</td>
<td align="left">CD4</td>
<td align="left">LENG8</td>
<td align="left">PHACTR2</td>
<td align="left">TRGC1</td>
<td align="left">KLF2</td>
<td align="left">CD74</td>
<td align="left">PPARG</td>
<td align="left">IFNG</td>
<td align="left">EPSTI1</td>
</tr>
<tr class="even">
<td align="left">SH3BGRL3</td>
<td align="left">PLAC8</td>
<td align="left">TRG-AS1</td>
<td align="left">ABCB1</td>
<td align="left">TXK</td>
<td align="left">CD79B</td>
<td align="left">HLA-DPA1</td>
<td align="left">GADD45G</td>
<td align="left">NR4A2</td>
<td align="left">MX2</td>
</tr>
<tr class="odd">
<td align="left">SAP18</td>
<td align="left">LTB</td>
<td align="left">NKTR</td>
<td align="left">HIPK2</td>
<td align="left">CD160</td>
<td align="left">ZC2HC1A</td>
<td align="left">LYST</td>
<td align="left">CYSLTR1</td>
<td align="left">BTG2</td>
<td align="left">IFIH1</td>
</tr>
<tr class="even">
<td align="left">VAMP8</td>
<td align="left">TCF7</td>
<td align="left">ZBTB20</td>
<td align="left">TBC1D31</td>
<td align="left">BCAS4</td>
<td align="left">IL6R</td>
<td align="left">APOBEC3G</td>
<td align="left">ALAS1</td>
<td align="left">CRTAM</td>
<td align="left">IFI44</td>
</tr>
<tr class="odd">
<td align="left">RPL18A</td>
<td align="left">AP3M2</td>
<td align="left">CHST12</td>
<td align="left">RUNX2</td>
<td align="left">DGKD</td>
<td align="left">SOCS3</td>
<td align="left">LINC00861</td>
<td align="left">ZC2HC1A</td>
<td align="left">RGCC</td>
<td align="left">HERC6</td>
</tr>
<tr class="even">
<td align="left">RPL29</td>
<td align="left">NOSIP</td>
<td align="left">PDE7A</td>
<td align="left">IFNGR1</td>
<td align="left">ZNF683</td>
<td align="left">MAF</td>
<td align="left">CLDND1</td>
<td align="left">TMEM273</td>
<td align="left">SPRY1</td>
<td align="left">STAT1</td>
</tr>
<tr class="odd">
<td align="left">ZFAS1</td>
<td align="left">KDSR</td>
<td align="left">PCNX1</td>
<td align="left">RORA</td>
<td align="left">VAV3</td>
<td align="left">ICOS</td>
<td align="left">NKG7</td>
<td align="left">C1orf162</td>
<td align="left">GZMB</td>
<td align="left">IFI35</td>
</tr>
<tr class="even">
<td align="left">RPS8</td>
<td align="left">RIPOR2</td>
<td align="left">TMX4</td>
<td align="left">MAN1A1</td>
<td align="left">GZMA</td>
<td align="left">NAMPT</td>
<td align="left">F2R</td>
<td align="left">TESPA1</td>
<td align="left">GFOD1</td>
<td align="left">OAS2</td>
</tr>
<tr class="odd">
<td align="left">NEAT1</td>
<td align="left">INPP4B</td>
<td align="left">POLR2J3.#~2</td>
<td align="left">ELK3</td>
<td align="left">CTSW</td>
<td align="left">CCR4</td>
<td align="left">ITGB2</td>
<td align="left">GNAQ</td>
<td align="left">HIC1</td>
<td align="left">DDX60L</td>
</tr>
<tr class="even">
<td align="left">LTB</td>
<td align="left">LDHB</td>
<td align="left">PAG1</td>
<td align="left">AQP3</td>
<td align="left">SRGAP3</td>
<td align="left">ARID5B</td>
<td align="left">LYAR</td>
<td align="left">ACSL4</td>
<td align="left">FASLG</td>
<td align="left">IRF7</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">EEF1B2</td>
<td align="left">AHNAK</td>
<td align="left">NR1D2</td>
<td align="left">PTGDR</td>
<td align="left">MAST4</td>
<td align="left">PYHIN1</td>
<td align="left">LIMA1</td>
<td align="left">GPR18</td>
<td align="left">RTP4</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPS3A</td>
<td align="left">MTRNR2L12</td>
<td align="left">PNP</td>
<td align="left">SERPINB6</td>
<td align="left">CD4</td>
<td align="left">LITAF</td>
<td align="left">OSM</td>
<td align="left">EVI2A</td>
<td align="left">SAMD9L</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">RPL9</td>
<td align="left">OXNAD1</td>
<td align="left">SPOCK2</td>
<td align="left">TRGC2</td>
<td align="left">CREM</td>
<td align="left">APOBEC3C</td>
<td align="left">PLIN2</td>
<td align="left">TAGAP</td>
<td align="left">DDX58</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPL5</td>
<td align="left">CTSD</td>
<td align="left">ERN1</td>
<td align="left">TRG-AS1</td>
<td align="left">LMNA</td>
<td align="left">GZMM</td>
<td align="left">IFNGR2</td>
<td align="left">ZFP36L1</td>
<td align="left">HELZ2</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">RPL3</td>
<td align="left">MT-ND3</td>
<td align="left">PDE4D</td>
<td align="left">CRTAM</td>
<td align="left">CCR7</td>
<td align="left">APMAP</td>
<td align="left">PKIA</td>
<td align="left">PGGHG</td>
<td align="left">IFIT5</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPL11</td>
<td align="left">DYNC1H1</td>
<td align="left">CERK</td>
<td align="left">ENTPD1</td>
<td align="left">STAM</td>
<td align="left">PTMS</td>
<td align="left">PHLDA1</td>
<td align="left">REL</td>
<td align="left">HSH2D</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">RPL29</td>
<td align="left">GABPB1-AS1</td>
<td align="left">TTC39C</td>
<td align="left">FAM3C</td>
<td align="left">GLRX</td>
<td align="left">CEMIP2</td>
<td align="left">TNFSF10</td>
<td align="left">TNFSF14</td>
<td align="left">DDX60</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPL22</td>
<td align="left">MT-ND4L</td>
<td align="left">NDRG1</td>
<td align="left">DAPK2</td>
<td align="left">CD28</td>
<td align="left">CD81</td>
<td align="left">CCDC71L</td>
<td align="left">SDCBP</td>
<td align="left">PARP9</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">RPL35A</td>
<td align="left">NSD1</td>
<td align="left">JAML</td>
<td align="left">RAB3GAP1</td>
<td align="left">TNFRSF1B</td>
<td align="left">GIMAP4</td>
<td align="left">CENPV</td>
<td align="left">ID2</td>
<td align="left">OASL</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPS13</td>
<td align="left">AKNA</td>
<td align="left">GPR65</td>
<td align="left">CD7</td>
<td align="left">RELB</td>
<td align="left">CCSER2</td>
<td align="left">DUSP6</td>
<td align="left">PRF1</td>
<td align="left">PLSCR1</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">RPL18A</td>
<td align="left">MT-ND1</td>
<td align="left">YWHAQ</td>
<td align="left">CSF1</td>
<td align="left">PIM2</td>
<td align="left">PRKCB</td>
<td align="left">AHR</td>
<td align="left">PTGER4</td>
<td align="left">PARP12</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPL10A</td>
<td align="left">MDM4</td>
<td align="left">LMO4</td>
<td align="left">TNIP3</td>
<td align="left">FRMD4B</td>
<td align="left">SYNE1</td>
<td align="left">METTL8</td>
<td align="left">FABP5</td>
<td align="left">TNFSF10</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">RPS25</td>
<td align="left">MT-ND2</td>
<td align="left">MIS18BP1</td>
<td align="left">AOAH</td>
<td align="left">GBP2</td>
<td align="left">ARAP2</td>
<td align="left">NME1</td>
<td align="left">PHLDA1</td>
<td align="left">PARP14</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPS5</td>
<td align="left">CLEC2D</td>
<td align="left">PMVK</td>
<td align="left">LINC01871</td>
<td align="left">CMTM6</td>
<td align="left">PPP1R14B</td>
<td align="left">NDFIP2</td>
<td align="left">KLRD1</td>
<td align="left">GBP4</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">RPL14</td>
<td align="left">ASH1L</td>
<td align="left">ZNRF1</td>
<td align="left">SCML4</td>
<td align="left">DNPH1</td>
<td align="left">DGKZ</td>
<td align="left">LINC01943</td>
<td align="left">PTPN7</td>
<td align="left">STAT2</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">PFDN5</td>
<td align="left">ANKRD13D</td>
<td align="left">DDIT4</td>
<td align="left">GSTP1</td>
<td align="left">RAB11FIP1</td>
<td align="left">TC2N</td>
<td align="left">ICOS</td>
<td align="left">SH2D2A</td>
<td align="left">TRIM22</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">UQCRB</td>
<td align="left">N4BP2L2</td>
<td align="left">OSTF1</td>
<td align="left">IL2RB</td>
<td align="left">FAS</td>
<td align="left">ITGB1</td>
<td align="left">P2RX5</td>
<td align="left">IER2</td>
<td align="left">TRIM25</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">BTF3</td>
<td align="left">CD2</td>
<td align="left">S100A4</td>
<td align="left">GNPTAB</td>
<td align="left">PIM3</td>
<td align="left">HMGB2</td>
<td align="left">PDLIM5</td>
<td align="left">NUCB2</td>
<td align="left">GBP1</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">EIF3G</td>
<td align="left">MYO1F</td>
<td align="left">RGS14</td>
<td align="left">PTPN22</td>
<td align="left">TFRC</td>
<td align="left">MXD4</td>
<td align="left">BATF</td>
<td align="left">NKG7</td>
<td align="left">SAMD9</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">RPL36AL</td>
<td align="left">PCSK7</td>
<td align="left">RPSA</td>
<td align="left">MATK</td>
<td align="left">FURIN</td>
<td align="left">YPEL3</td>
<td align="left">THADA</td>
<td align="left">NFKBIA</td>
<td align="left">IFITM1</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">EIF3K</td>
<td align="left">MTRNR2L1</td>
<td align="left">RPL41</td>
<td align="left">PLA2G16</td>
<td align="left">BATF</td>
<td align="left">RIPOR2</td>
<td align="left">CREM</td>
<td align="left">RILPL2</td>
<td align="left">NT5C3A</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">COX4I1</td>
<td align="left">ACAP1</td>
<td align="left">RPS24</td>
<td align="left">SEMA4D</td>
<td align="left">MIR4435-2HG</td>
<td align="left">MT2A</td>
<td align="left">NR3C1</td>
<td align="left">TIGIT</td>
<td align="left">C19orf66</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">COMMD6</td>
<td align="left">TRIM56</td>
<td align="left">SOD1</td>
<td align="left">CCDC69</td>
<td align="left">AQP3</td>
<td align="left">LIMD2</td>
<td align="left">SLAMF1</td>
<td align="left">IPCEF1</td>
<td align="left">LAP3</td>
</tr>
<tr class="even">
<td align="left"></td>
<td align="left">GTF3A</td>
<td align="left">ITM2C</td>
<td align="left">EMP3</td>
<td align="left">PRKACB</td>
<td align="left">LIMS1</td>
<td align="left">CXCR4</td>
<td align="left">TIMP1</td>
<td align="left">ZFP36</td>
<td align="left">DTX3L</td>
</tr>
</tbody>
</table>

Now with the AUROC and OVA marker genes, we can visualize the markers with a heatmap. First, we looked at the data with a heatmap where both the rows and columns were clustered.

``` python
from mpl_toolkits.axes_grid1 import make_axes_locatable
import matplotlib as mpl
heatmap_genes = []
repeated_genes = [] # Get genes that are not unique, do not want to annotate them
for key in top_genes.keys() :
    for gene in top_genes[key] :
        if gene not in heatmap_genes :
            heatmap_genes.append(gene)
        else :
            repeated_genes.append(gene)

# Get the genes for annotation: top markers that are not in repeated genes
annot_genes = {}
for clust in top_genes.keys() :
    non_rep_genes = [gene for gene in top_genes[clust] if gene not in repeated_genes and not gene.startswith("RP")]
    annot_genes[clust] = non_rep_genes

# Write out the annotation genes for the heatmap (making with ComplexHeatmap)
annot_genes = pd.DataFrame(dict([ (k,pd.Series(v)) for k,v in annot_genes.items() ]))
annot_genes = annot_genes.rename(columns = {clust : "cluster_{clust}".format(clust=clust) for clust in annot_genes.columns})
# Lets add the colors for each cluster from the UMAP
clust_cols = dict(zip(sorted(set(t_cell_harmonized.obs["leiden_labels"]), key = int),
                      t_cell_harmonized.uns["leiden_labels_colors"]))
clust_cols = pd.DataFrame(clust_cols,
                          index = ["col"]).rename(columns = dict(zip(clust_cols.keys(),
                                                                     ["cluster_{clust}".format(clust = clust) for clust
                                                                      in clust_cols.keys()])))

annot_genes = annot_genes.append(clust_cols)

# Also need to add mean gene counts
# Get the mean gene counts for sidebar
gene_val_list = []
gene_val_dict = {}
for clust in sorted(set(t_cell_harmonized.obs["leiden_labels"]), key = int) :
    gene_vals = t_cell_harmonized.obs["n_genes"][t_cell_harmonized.obs["leiden_labels"] == clust]

    mean = np.mean(gene_vals)
    gene_val_list.append(mean)
    gene_val_dict[clust] = mean

# Append these mean gene counts to the dataframe
annot_genes = annot_genes.append(pd.DataFrame(gene_val_dict,
                          index = ["mean_genes"]).rename(columns = dict(zip(gene_val_dict.keys(),
                                                                     ["cluster_{clust}".format(clust = clust) for clust
                                                                      in gene_val_dict.keys()]))))


# Get the mean expression of the top genes from each cluster
de_df = {"mean_log_{clust}".format(clust = clust) : t_cell_harmonized.varm["de_res"]["mean_logExpr:{clust}".format(clust = clust)] for clust in sorted(set(t_cell_harmonized.obs["leiden_labels"]), key = int)}
de_df = pd.DataFrame(de_df, index = t_cell_harmonized.var.index)

heatmap_df = de_df.loc[heatmap_genes]


colors = sns.color_palette("ch:2.5,-.2,dark=.2", n_colors = len(gene_val_list)).as_hex()
# Put the gene values in order lowest to highest
sorted_cols = sorted(gene_val_list)

fig, ax = plt.subplots(1, 1, figsize = (10, 10))
divider = make_axes_locatable(ax)
axDivY = divider.append_axes( 'right', size=0.2, pad= 0.1)
axDivY2 = divider.append_axes( 'right', size=0.2, pad= 0.2)
axDivY3 = divider.append_axes( 'right', size=0.2, pad= 0.2)
axDivY4 = divider.append_axes( 'top', size=0.2, pad= 0.2)

ax1 = sns.clustermap(heatmap_df, method = "ward", row_cluster =True, col_cluster =True, z_score = 0, cmap = "vlag")
col_order = np.array([name.split("_")[-1] for name in ax1.data2d.columns])
index = [sorted_cols.index(gene_val_dict[clust]) for clust in col_order]
plt.close()
ax1 = sns.heatmap(ax1.data2d, cmap = "vlag", ax = ax, cbar_ax = axDivY)
ax2 = axDivY2.imshow(np.array([[min(gene_val_list), max(gene_val_list)]]), cmap = mpl.colors.ListedColormap(list(colors)),
                     interpolation = "nearest", aspect = "auto")
axDivY2.set_axis_off()
axDivY2.set_visible(False)
_ = plt.colorbar(ax2, cax = axDivY3)
_ = axDivY3.set_title("n_genes")
ax3 = axDivY4.imshow(np.array([index]),cmap=mpl.colors.ListedColormap(list(colors)),
              interpolation="nearest", aspect="auto")
axDivY4.set_axis_off()
_ = plt.title("top genes for every cluster")
plt.show()
```

<img src="figure_2_files/figure-markdown_github/heatmap1-1.png" width="960" /><img src="figure_2_files/figure-markdown_github/heatmap1-2.png" width="960" />

To make things more readable, we also made a heatmap where we kept the columns clustered such that phenotypically similar clusters were grouped together, but manually ordered the rows.

``` python
n_heatmap_genes = {}
heatmap_genes = []
for key in col_order :
    cnt = 0
    for gene in top_genes[key] :
        if gene not in heatmap_genes :
            heatmap_genes.append(gene)
            cnt+=1
    n_heatmap_genes[key] = cnt

n_heatmap_genes = pd.DataFrame(n_heatmap_genes, index = ["n_genes"]).rename(columns = dict(zip(n_heatmap_genes.keys(),
                                                                                               ["cluster_{clust}".format(clust = clust) for
                                                                                                clust in n_heatmap_genes.keys()])))
# Add number of genes in the heatmap for each clusters
annot_genes = annot_genes.append(n_heatmap_genes)
annot_genes = annot_genes.reset_index()
annot_genes = annot_genes.fillna('')


# Get the mean expression of the top genes from each cluster
de_df = {"mean_log_{clust}".format(clust = clust) : t_cell_harmonized.varm["de_res"]["mean_logExpr:{clust}".format(clust = clust)] for clust in sorted(set(t_cell_harmonized.obs["leiden_labels"]), key = int)}
de_df = pd.DataFrame(de_df, index = t_cell_harmonized.var.index)

heatmap_df = de_df.loc[heatmap_genes]

# Get the mean gene counts for sidebar
gene_val_list = []
gene_val_dict = {}
for clust in sorted(set(t_cell_harmonized.obs["leiden_labels"]), key = int) :
    gene_vals = t_cell_harmonized.obs["n_genes"][t_cell_harmonized.obs["leiden_labels"] == clust]

    mean = np.mean(gene_vals)
    gene_val_list.append(mean)
    gene_val_dict[clust] = mean

colors = sns.color_palette("ch:2.5,-.2,dark=.2", n_colors = len(gene_val_list)).as_hex()
# Put the gene values in order lowest to highest
sorted_cols = sorted(gene_val_list)

fig, ax = plt.subplots(1, 1, figsize = (10, 10))
divider = make_axes_locatable(ax)
axDivY = divider.append_axes( 'right', size=0.2, pad= 0.1)
axDivY2 = divider.append_axes( 'right', size=0.2, pad= 0.2)
axDivY3 = divider.append_axes( 'right', size=0.2, pad= 0.2)
axDivY4 = divider.append_axes( 'top', size=0.2, pad= 0.2)

# color_label_list =[random.randint(0,14) for i in range(14)]
ax1 = sns.clustermap(heatmap_df, method = "ward", row_cluster =False, col_cluster =True, z_score = 0, cmap = "vlag")
col_order = np.array([name.split("_")[-1] for name in ax1.data2d.columns])
index = [sorted_cols.index(gene_val_dict[clust]) for clust in col_order]
plt.close()

heatmap_carpet = ax1.data2d

ax1 = sns.heatmap(ax1.data2d, cmap = "vlag", ax = ax, cbar_ax = axDivY)
ax2 = axDivY2.imshow(np.array([[min(gene_val_list), max(gene_val_list)]]), cmap = mpl.colors.ListedColormap(list(colors)),
                     interpolation = "nearest", aspect = "auto")
axDivY2.set_axis_off()
axDivY2.set_visible(False)
_ = plt.colorbar(ax2, cax = axDivY3)
_ = axDivY3.set_title("n_genes")
ax3 = axDivY4.imshow(np.array([index]),cmap=mpl.colors.ListedColormap(list(colors)),
              interpolation="nearest", aspect="auto")
axDivY4.set_axis_off()
_ = plt.title("top genes for every cluster")
plt.show()
```

<img src="figure_2_files/figure-markdown_github/ordered_heatmap-5.png" width="960" /><img src="figure_2_files/figure-markdown_github/ordered_heatmap-6.png" width="960" />

Finally, we wanted to make a publication-ready figure using the wonderful `ComplexHeatmap` package, where we can add some annotations for each cluster and add spaces between clusters to make it even more readable.

``` r
library(ComplexHeatmap)
library(tidyverse)
library(magrittr)
library(circlize)

heatmap_data <- reticulate::py$heatmap_carpet
annotation_info <- reticulate::py$annot_genes
rownames(annotation_info) <- annotation_info$index
annotation_info$index <- NULL


for (c in colnames(annotation_info)){
  annotation_info[[c]] <- unlist(annotation_info[[c]])
}
# Change the column names to be cleaner
colnames(heatmap_data) <- paste("Cluster", unlist(strsplit(colnames(heatmap_data), "_"))[3*(1:length(colnames(heatmap_data)))], sep = " ")

# Make column names consistent with heatmap data
colnames(annotation_info) <- sapply(str_replace(colnames(annotation_info), "_", " "), str_to_title)

# Lets just get the genes
annotation_genes <- unique(as.character(unlist(annotation_info[1:5,])))
annotation_genes <- annotation_genes[annotation_genes != ""]

# Now lets organize the color info that will be used for annotations
col_info = annotation_info %>%
  t() %>%
  as.data.frame() %>%
  dplyr::select(-mean_genes) %>%
  rownames_to_column(var = "cluster") %>%
  reshape2::melt(id.vars = c("cluster", "col")) %>%
  select(-variable)

# Get the gene colors
gene_cols = c()
for (gene in annotation_genes){
  color = as.character(filter(col_info, value == gene)["col"][[1]])
  gene_cols = c(gene_cols, color)
}

# Get the cluster colors
clust_cols <- c()
for (clust in colnames(heatmap_data)){
  color <- col_info %>%
    dplyr::select(cluster, col) %>%
    distinct() %>%
    filter(cluster == clust)
  clust_cols <- c(clust_cols, as.character(color$col))
}

mean_genes <- annotation_info["mean_genes",] %>%
  mutate_each(funs(as.numeric(as.character(.)))) %>%
  select(colnames(heatmap_data)) # To order them like they will be ordered in the heatmap (same as how GEX data was read in)

gene_col_fun <- colorRamp2(c(min(mean_genes), max(mean_genes)), c("#1d111d", "#bbe7c8"))
gene_bar <-  HeatmapAnnotation("mean # genes" = as.numeric(mean_genes), col = list("mean # genes" = gene_col_fun), show_legend = FALSE)
gene_lgd <- Legend(col_fun = gene_col_fun, title = "# genes", legend_height = unit(4, "cm"), title_position = "topcenter")


heatmap_col_fun = colorRamp2(c(min(heatmap_data), 0, max(heatmap_data)), c("purple", "black", "yellow"))
heatmap_lgd = Legend(col_fun = heatmap_col_fun, title = "z-score", legend_height = unit(4, "cm"), title_position = "topcenter")

lgd_list <- packLegend(heatmap_lgd, gene_lgd, column_gap = unit(1,"cm"), direction = "horizontal")

split <- c()
for (clust in colnames(heatmap_data)){
  n_split <- as.numeric(as.character(annotation_info["n_genes", clust]))
  split <- c(split, rep(gsub("Cluster ", "", clust), n_split))
}
split <- factor(split, levels = as.character(unique(split)))

# Make block annotation
left_annotation =   HeatmapAnnotation(blk = anno_block(gp = gpar(fill = clust_cols, col = clust_cols)), which = "row", width = unit(1.5, "mm"))

heatmap_list = Heatmap(heatmap_data, name = "z-score", col = heatmap_col_fun, cluster_rows = FALSE, cluster_columns = TRUE,
                       cluster_row_slices = FALSE, row_km = 1, cluster_column_slices = FALSE,
                       clustering_method_columns = "ward.D2", clustering_distance_columns = "euclidean",
                       column_dend_reorder = FALSE, top_annotation = gene_bar, show_heatmap_legend = FALSE,
                       column_names_gp = gpar(col = clust_cols, fontface = "bold"),
                       split = split, left_annotation = left_annotation, show_column_names = FALSE) +
  rowAnnotation(link = anno_mark(at = match(annotation_genes, rownames(heatmap_data)),labels = annotation_genes,
                                 labels_gp = gpar(col = gene_cols, fontsize = 8, fontface = "bold")))

draw(heatmap_list, heatmap_legend_list =lgd_list, padding = unit(c(0.5, 0.5, 2, 2), "cm"), cluster_rows = FALSE,
     cluster_row_slices = FALSE)
```

![](figure_2_files/figure-markdown_github/nice_heatmap-9.png)

# Differential abundance analysis

To determine which clusters were associated with particular group at a given condition, we used a mixed-effects association logistic regression model similar to that described by Fonseka et al. We fit a logistic regression model for each cell cluster. Each cluster was modelled independently as follows : `cluster ~ 1 + condition:group + condition + group + (1 | id)`

The least-squares means of the factors in the model were calculated and all pairwise contrasts between the means of the groups at each condition (e.g. AA vs AC within baseline, AA vs AC within allergen, etc.) were compared. The OR with confidence interval for each sample/condition combination were plotted.

``` python
cell_info = t_cell_harmonized.obs
```

<img src="figure_2_files/figure-markdown_github/cell_info-1.png" width="960" />

``` r
library(lme4)
library(ggplot2)
library(emmeans)

# A function to perform the mixed-effects logistic regression modelling, returning, p-values, odds-ratio and confidence intervals
get_abund_info <- function(dataset, cluster, contrast, random_effects = NULL, fixed_effects = NULL){
  # Generate design matrix from cluster assignments
  cluster <- as.character(cluster)
  designmat <- model.matrix(~ cluster + 0, data.frame(cluster = cluster))
  dataset <- cbind(designmat, dataset)
  # Create output list to hold results
  res <- vector(mode = "list", length = length(unique(cluster)))
  names(res) <- attributes(designmat)$dimnames[[2]]

  # Create model formulas
  model_rhs <- paste0(c(paste0(fixed_effects, collapse = " + "),
                        paste0("(1|", random_effects, ")", collapse = " + ")),
                      collapse = " + ")

  # Initialize list to store model objects for each cluster
  cluster_models <- vector(mode = "list",
                           length = length(attributes(designmat)$dimnames[[2]]))
  names(cluster_models) <- attributes(designmat)$dimnames[[2]]

  for (i in seq_along(attributes(designmat)$dimnames[[2]])) {
    test_cluster <- attributes(designmat)$dimnames[[2]][i]

    # Make it a non-intercept model to get odds for each variable
    full_fm <- as.formula(paste0(c(paste0(test_cluster, " ~ 1 + ", contrast, " + "),
                                   model_rhs), collapse = ""))

    full_model <- lme4::glmer(formula = full_fm, data = dataset,
                              family = binomial, nAGQ = 1, verbose = 0,
                              control = glmerControl(optimizer = "bobyqa"))

    pvals <-lsmeans(full_model, pairwise ~ "phenotype | sample")

    p_val_df <- summary(pvals$contrasts)
    p_val_df$cluster <- test_cluster

    ci <- eff_size(pvals, sigma = sigma(full_model), edf = df.residual(full_model))
    ci_df <- summary(ci) %>%
    dplyr::select(sample, asymp.LCL, asymp.UCL)
    ci_df$cluster <- test_cluster

    info_df <- left_join(p_val_df, ci_df, by = c("sample", "cluster"))

    cluster_models[[i]] <- info_df

  }
  return(cluster_models)
}

# A function to make a forest plot for the differential abundance analyses
plot_contrasts <- function(d, x_breaks_by = 1, wrap_ncol = 6, y_ord = FALSE) {
  # if (y_ord != FALSE){
  #   d$cluster <- factor(d$cluster, levels = y_ord)
  # }
  ggplot() +
    annotate(
      geom = "rect",
      xmin = -Inf,
      xmax = Inf,
      ymin = seq(from = 1, to = length(unique(d$cluster)), by = 2) - 0.5,
      ymax = seq(from = 1, to = length(unique(d$cluster)), by = 2) + 0.5,
      alpha = 0.2
    ) +
    geom_vline(xintercept = 0, size = 0.2) +
    geom_errorbarh(
      data = d,
      mapping = aes(
        xmin = asymp.LCL, xmax = asymp.UCL, y = cluster,
        color = sig
      ),
      height = 0
    ) +
    geom_point(
      data = d,
      mapping = aes(
        x = estimate, y = cluster,
        color = sig
      ),
      size = 3
    ) +
    scale_color_manual(
      name = "P < 0.05",
      values = c("#FF8000", "#40007F", "grey60"),
      breaks = c("AA", "ANA")
    ) +
    scale_x_continuous(
      breaks = log(c(0.125, 0.5, 1, 2, 4)),
      labels = function(x) exp(x)
    ) +
    scale_y_discrete(
      # expand = c(0, 0),
      # breaks = seq(1, length(unique(d$cluster))),
      labels = levels(plot_df$cluster),
    ) +
    # annotation_logticks(sides = "b") +
    expand_limits(y = c(0.5, length(unique(d$cluster)) + 0.5)) +
    # facet_grid(~ GeneName) +
    facet_wrap(~ sample, ncol = wrap_ncol) +
    theme_classic() +
    theme(
      strip.text = element_text(face = "italic"),
      text = element_text(size = 15)
    ) +
    labs(
      title = "contrasts by sample",
      x = "Odds Ratio",
      y = "cluster"
    )
}

clust_df <- reticulate::py$cell_info

abund_info <- get_abund_info(clust_df, cluster = clust_df$leiden_labels,
                            contrast = "sample:phenotype",
                            random_effects = "id",
                            fixed_effects = c("sample", "phenotype"))

plot_df <- do.call(rbind, abund_info)
plot_df$direction <- ifelse(plot_df$estimate > 0, "AA", "ANA")


plot_df$cluster <- as.numeric(gsub("cluster", "", plot_df$cluster))
plot_df$sig <- ifelse(plot_df$p.value < 0.05, plot_df$direction, "non_sig")
cl_order <- c(9, 7, 3, 1, 5, 6, 8, 10, 2, 4)
plot_df$cluster <- factor(plot_df$cluster, levels = rev(cl_order))
plot_df$sample <- factor(plot_df$sample, levels = c("Pre", "Dil", "Ag"))
plot_contrasts(plot_df)
```

![](figure_2_files/figure-markdown_github/diff_abund-3.png)

The most noticable thing in terms of differences between AA and AC are the Th2 cells. Even more interestingly, the IL9 expression in the Th2 cells is almost exclusively in the AA as shown below.

``` python

cmap_dict = {"AA" : clr.LinearSegmentedColormap.from_list('gene_cmap', ["#d3d3d3" ,'#FF8000'], N=200),
             "ANA" : clr.LinearSegmentedColormap.from_list('gene_cmap', ["#d3d3d3" ,'#40007F'], N=200)}

il9_fig, ax = plt.subplots(nrows = 1, ncols = 2, figsize = (10, 3))
ax = ax.ravel()
for num, pheno in enumerate(["AA", "ANA"]) :
    pheno_dat = t_cell_harmonized[t_cell_harmonized.obs["phenotype"] == pheno]
    _ = sc.pl.umap(pheno_dat, color = "IL9", cmap = cmap_dict[pheno], show = False, ax = ax[num],
               title = pheno)
_ = il9_fig.suptitle("IL9 expression")
il9_fig
```

<img src="figure_2_files/figure-markdown_github/il9_plot-1.png" width="960" />

We can look at the other Th2 genes in feature plots to show they really have a pathogenic effector phenotype

``` python

colormap = clr.LinearSegmentedColormap.from_list('gene_cmap', ["#e0e0e1", '#4576b8', '#02024a'], N=200)

feature_genes = ["IL4", "IL5", "IL9", "IL13", "IL17RB", "HPGDS",
                 "PTGDR2", "IL1RL1", "GATA3", "PPARG", "IRF4",
                 "IL1RL1"]

fig, ax = plt.subplots(nrows = 4, ncols = 3, figsize = (7, 7))
ax = ax.ravel()
for num, gene in enumerate(feature_genes) :
    _ = sc.pl.umap(t_cell_harmonized, color=gene, cmap=colormap, show=False, ax = ax[num], title = gene)
fig.tight_layout()
fig
```

<img src="figure_2_files/figure-markdown_github/th2_plot-3.png" width="672" />

# Differential expression analysis

Next, we wanted to know which genes were differentially expressed between AA and AC at Ag. For this, we used `DESeq2` on the pseudobulk count matrix. We used a simple model of `gene ~ phenotype` where phenotype was a factor with 2 levels indicating the phenotypical group the sample came from. There were not too many significant genes, however the ones that came out were interesting. For example, in the Th2 cells, we see IL2RA and CTLA4 were up-regulated in the AA. All significant genes we found were up-regulated in the AA.

``` r
library(DESeq2)
library(glue)

count_mtx <- as.matrix(read.csv("/home/nealpsmith/projects/medoff/data/pseudobulk_t_cell_harmonized_counts.csv", row.names = 1))
meta_data <- read.csv("/home/nealpsmith/projects/medoff/data/pseudobulk_t_cell_harmonized_meta.csv", row.names = 1)
meta_data$phenotype <- factor(meta_data$phenotype, levels = c("ANA", "AA"))

# Limit to just Ag samples
meta_data <- meta_data[meta_data$sample == "Ag",]
count_mtx <- count_mtx[,rownames(meta_data)]

de_list <- list()
for (clust in unique(meta_data$cluster)){
  clust_meta <-meta_data[meta_data$cluster == clust,]
  clust_count <- count_mtx[,rownames(clust_meta)]
    if (nrow(clust_meta) > 5){

      n_samp <- rowSums(clust_count != 0)
      clust_count <- clust_count[n_samp > round(nrow(clust_meta) / 2),]

      stopifnot(rownames(clust_meta) == colnames(clust_count))

      dds <- DESeqDataSetFromMatrix(countData = clust_count,
                                    colData = clust_meta,
                                    design = ~phenotype)
      dds <- DESeq(dds)
      res <- results(dds)
      plot_data <- as.data.frame(res)
      plot_data <- plot_data[!is.na(plot_data$padj),]
      plot_data$gene <- rownames(plot_data)
      de_list[[glue("clust_{clust}")]] <- plot_data
    }

}

aa_up <- lapply(names(de_list), function(x){
    data <- de_list[[x]]
    up_aa <- data[data$padj < 0.1 & data$log2FoldChange > 0,]$gene
    up_ana <- data[data$padj < 0.1 & data$log2FoldChange < 0,]$gene
    info_df <- data.frame("gene" = up_aa)
    if (nrow(info_df) > 0){
      info_df$cluster <- x
    }
  return(info_df)
  }) %>%
    do.call(rbind, .)

aa_up
```

    ##    gene cluster
    ## 1 CTLA4 clust_8
    ## 2 IL2RA clust_8
    ## 3   MAL clust_2
    ## 4  GZMB clust_5
    ## 5  CCR7 clust_3
    ## 6 SOCS3 clust_3

To visualize these, we used violin plots comparing the AA and AC expression distributions in the particular clusters

``` python
t_cell_ag = t_cell_harmonized[t_cell_harmonized.obs["sample"] == "Ag"]

plot_info = pd.DataFrame({"gene" : ["IL2RA", "CTLA4", "MAL", "GZMB", "CCR7", "SOCS3"],
                          "clust" : ["8", "8", "2", "5", "3", "3"]})

fig, ax = plt.subplots(ncols=plot_info.shape[0], figsize = (12, 2))
ax = ax.ravel()

for num, indx in enumerate(plot_info.index.values) :
    gene = plot_info.loc[indx, "gene"]
    in_clust = plot_info.loc[indx, "clust"]
    t_cell_gene_data = pd.DataFrame.sparse.from_spmatrix(t_cell_ag[:, gene].X, columns=[gene],
                                                      index=t_cell_ag.obs_names).sparse.to_dense()
    t_cell_gene_data = t_cell_gene_data.merge(t_cell_ag.obs[["id", "phenotype", "leiden_labels"]], how="left",
                                        left_index=True, right_index=True)
    t_cell_gene_data = t_cell_gene_data[t_cell_gene_data["leiden_labels"] == in_clust]

    # Get medians
    medians = [np.median(t_cell_gene_data[t_cell_gene_data["phenotype"] == "AA"][gene]),
               np.median(t_cell_gene_data[t_cell_gene_data["phenotype"] == "ANA"][gene])]

    # Make the plot
    sns.violinplot(x="phenotype", y=gene, hue="phenotype", data=t_cell_gene_data, inner=None,
                   palette={"AA": "#FF8000", "ANA": "#40007F"}, ax=ax[num], cut=0, alpha=0.5, dodge = False)
    for violin in ax[num].collections:
        violin.set_alpha(0.5)
    sns.stripplot(x="phenotype", y=gene, hue="phenotype", data=t_cell_gene_data,
                  palette={"AA": "grey", "ANA": "grey"},
                  dodge=False, size=2, ax=ax[num], zorder=0)
    # ax[num].scatter(x = [0, 1], y = medians, marker = "_", s = 600, c = "black")
    _ = ax[num].get_legend().remove()
    _ = ax[num].set_xlabel("")
    _ = ax[num].set_ylabel("log(CPM)")
    _ = ax[num].set_title(f"{gene} : t_cell {in_clust}")

fig.tight_layout()
fig
```

<img src="figure_2_files/figure-markdown_github/de_violins-1.png" width="1152" />