# 单细胞Seurat包升级之2,700 PBMCs分析

> 刘小泽写于19.7.18、28 官方总共有7个数据集，涵盖了新版本的不同亮点
> 这一次跟着教程https://satijalab.org/seurat/v3.0/pbmc3k_tutorial.html走一遍最简单的教程，记录自己的学习轨迹
> 隔了这么久，中间肯定发生了什么故事嘿嘿
> 过去的10天，学到了很多东西，技能树单细胞培训、第一届生信人才发展论坛、今天来到北京，准备明天为期5天的北大龙星计划助教，不知道有没有认识的小伙伴参加。

### 前言

这个数据集是10X放出的2700个外周血单核细胞(PBMC，Peripheral Blood Mononuclear Cells )公共数据，使用 Illumina NextSeq 500测序，原始数据在：https://s3-us-west-2.amazonaws.com/10x.files/samples/cell/pbmc3k/pbmc3k_filtered_gene_bc_matrices.tar.gz

这个教程更新于2019-6-24，它会做这么几件事：QC and data filtration, calculation of high-variance genes, dimensional reduction, graph-based clustering, and the identification of cluster markers. 

### 从读取数据开始

10X的数据可以使用`Read10X`这个函数，会返回一个UMI count矩阵，其中的每个值表示每个基因(行表示)在每个细胞(列表示)的分子数量

##### 加载PBMC数据

```R
pbmc.data <- Read10X(data.dir = "filtered_gene_bc_matrices/hg19/")
```

##### 这个矩阵长什么样？和我们平常见到的bulk转录组矩阵一样吗？

找几个基因，看看前30个细胞的情况

```R
> pbmc.data[c("CD3D", "TCL1A", "MS4A1"), 1:30]

3 x 30 sparse Matrix of class "dgCMatrix"
   [[ suppressing 30 column names ‘AAACATACAACCAC’, ‘AAACATTGAGCTAC’, ‘AAACATTGATCAGC’ ... ]]
                                                                   
CD3D  4 . 10 . . 1 2 3 1 . . 2 7 1 . . 1 3 . 2  3 . . . . . 3 4 1 5
TCL1A . .  . . . . . . 1 . . . . . . . . . . .  . 1 . . . . . . . .
MS4A1 . 6  . . . . . . 1 1 1 . . . . . . . . . 36 1 2 . . 2 . . . .
```

从这个结果可以得出两个结论：它是`sparse Matrix`；它很杂乱

但其实仔细看，这个`.`就表示空着，数值为0，也就是没有检测到该分子。因为单细胞数据和bulk转录组数据还不太一样，scRNA的大部分数值都是0，原因一是由于建库时细胞的起始反应模板浓度就很低；二是测序存在dropout现象(本来基因在这个细胞有表达量，但没有测到)。

这里Seurat采用了一种聪明的再现方式，比原来用0表示的矩阵**大大减小了空间占用**，可以对比一下：

```R
dense.size <- object.size(as.matrix(pbmc.data))
dense.size # 709548272 bytes

sparse.size <- object.size(pbmc.data)
sparse.size # 29861992 bytes

dense.size/sparse.size #空间缩小23.8倍
```

##### 然后利用这个矩阵构建`Seurat`对象

这个对象是一个容器，里面装了数据(比如表达矩阵)和分析结果(比如PCA、聚类)。另外关于这个对象的详细解释，见：https://github.com/satijalab/seurat/wiki/Seurat#slots，包装了许多的接口

![](https://upload-images.jianshu.io/upload_images/9376801-61a7097d1f95ed3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 

```R
# 用原始数据（非标准化）先构建一个对象
pbmc <- CreateSeuratObject(counts = pbmc.data, project = "pbmc3k", min.cells = 3, min.features = 200)

> pbmc
An object of class Seurat 
13714 features across 2700 samples within 1 assay 
Active assay: RNA (13714 features)
```

发现这里有个`active assay`，它其中存的就是表达矩阵，用`pbmc[["RNA"]]@counts`就可以访问

> 使用两个中括号是对assay的快速访问，例如访问metadata：`pbmc[[]]`就如同`object@meta.data` 

另外看上面的图，seurat对象还有很多信息：

```R
# 可以利用str来查看
> str(pbmc)
Formal class 'Seurat' [package "Seurat"] with 12 slots
  ..@ assays      :List of 1
  .. ..$ RNA:Formal class 'Assay' [package "Seurat"] with 7 slots
  .. .. .. ..@ counts       :Formal class 'dgCMatrix' [package "Matrix"] with 6 slots
  .. .. .. 
# 后面还有很多
# 如果要查看其他的信息，可以用@，例如
> pbmc@version
[1] ‘3.0.0.9000’
```

### 预处理阶段

标准流程包括了根据QC 结果进行的细胞选择和过滤，标准化过程，检测显著差异基因

#### 质控（QC）及筛选细胞

这篇文章：https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4758103/介绍了一些QC的标准

例如：

- 检测每个细胞中检测到的unique genes数量(这种情况，一般低质量的细胞或空的液滴"droplet"中基因数量也会较少；如果一个液滴中有两个细胞"doublets"或者存在多个细胞"multiplets"，这样会导致检测到的基因数量出奇的高)

- 利用细胞中检测到的molecules总量也可以(它的结果和unique genes方法高度相关)

- 比对到线粒体基因组的reads数量，因为一般低质量或死亡的细胞中会广泛存在线粒体基因组污染；可以利用`PercentageFeatureSet`函数计算，顾名思义这个函数会计算来自某一个feature的reads count数量，需要做的就是挑出来以`MT-`开头的feature对应的所有基因。

  ```R
  # 质控筛选, [[符号可以在pbmc这个对象中添加metadata，利用这个方法还可以挑出其他的feature
  pbmc[["percent.mt"]] <- PercentageFeatureSet(pbmc, pattern = "^MT-")
  # 检查下metadata
  > head(pbmc@meta.data, 3)
                 orig.ident nCount_RNA nFeature_RNA percent.mito percent.mt
  AAACATACAACCAC   10X_PBMC       2419          779  0.030177759  3.0177759
  AAACATTGAGCTAC   10X_PBMC       4903         1352  0.037935958  3.7935958
  AAACATTGATCAGC   10X_PBMC       3147         1129  0.008897363  0.8897363
  # 绘制多个feature指标
  VlnPlot(pbmc, features = c("nFeature_RNA", "nCount_RNA", "percent.mt"), ncol = 3)
  ```

  ![](https://upload-images.jianshu.io/upload_images/9376801-af4192fc9ef5733c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  这张图可以给出的提示就是**过滤信息**：过滤的时候可以过滤掉feature大于2500和小于200的(就是避免前面所说基因数过多/过少的情况)；然后线粒体占比图显示大部分都集中在5%以下，因此超过5%的可以过滤掉

  ```R
  # 另外可以用一个散点图（FeatureScatter）来绘制两组feature指标的相关性，最后组合在一起（CombinePlots）
  plot1 <- FeatureScatter(pbmc, feature1 = "nCount_RNA", feature2 = "percent.mt")
  plot2 <- FeatureScatter(pbmc, feature1 = "nCount_RNA", feature2 = "nFeature_RNA")
  CombinePlots(plots = list(plot1, plot2))
  ```

  ![](https://upload-images.jianshu.io/upload_images/9376801-2a97979ca4fcccd0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  这个图给出的意思是：(左边👈)有表达量的reads和在线粒体的reads数量一般成反比，线粒体reads占比很高的情况一般有表达量的很少；相反，如果是真正表达的基因reads，很少会来自线粒体基因组；(右边👉)就是刚才所说的比对结果中的基因数量(feature)一般会和测序得到的reads count值成正比

  一切就绪，最后进行一个**过滤操作**

  ```R
  # 最后进行一个过滤操作
  pbmc <- subset(pbmc, subset = nFeature_RNA > 200 & nFeature_RNA < 2500 & percent.mt < 5)
  ```

#### 数据归一化（normalize）

> **为何normalize要叫“归一化”？**
> 这个名称不必纠结，看它使用的方法应该就好理解：因为我们经常关心的就是看差异表达基因，也就是找**一个基因在不同样本的差异**，而这种差异，往往不能直接进行比较，因为不同样本的测序深度和文库大小可能不能，那么基因表达量差异就存在一部分因素来自这里。为了更真实反映基因差异的生物学意义，就要将数据进行一个转换（例如RPKM、TPM等），可以让同一基因在不同样本中具有可比性。
> **“归一”归的就是这个文库大小。**
> 另外呢，原始表达量是一个离散程度很高的值，有的高表达基因表达量可能成千上万，可有的却只有几十。我们即使采用了一些归一化的算法消除了一些技术性误差，但**真实存在的表达量极差往往会影响之后绘制热图、箱线图的美观性**，于是可以另外采用`log` 进行一个**区间缩放（却不会影响真实值）**，比如原来表达量为1的，用`log2(1+1)`转换后结果依然为1；原来表达量为1000的，`log2(1000+1)`转换后约为10。那么原来相差1000倍的变化，现在只差10倍，在不破坏原有数据差异的同时，使数据更加集中。

关于Seurat归一化原理，可以看这一篇：https://www.biorxiv.org/content/biorxiv/early/2019/03/18/576827.full.pdf

当过滤掉一部分细胞以后，就要会数据进行归一化。默认是进行一个全局的`LogNormalize`操作，具体方法是：

> normalizes the feature expression measurements for each cell by the **total expression**, multiplies this by a **scale factor** (10,000 by default), and **log-transforms** the result
> 翻译成公式就是：`log1p(value/colSums[cell-idx] *scale_factor)` ，其中`log1p指的是log(x + 1)`，(Tip:只看到`log`就表示`以e为底的log值`，用`log1p(exp(1)-1)`测试一下，看结果是`1` )
>
> 这里也有解释：https://github.com/satijalab/seurat/issues/833
>
> 当然，除了这一种默认的算法，还有：
>
> - **CLR:** Applies a centered log ratio transformation
> - **RC:** Relative counts. Feature counts for each cell are divided by the total counts for that cell and multiplied by the scale.factor. No log-transformation is applied. For counts per million (CPM) set `scale.factor = 1e6`

得到的结果存储在：`pbmc[["RNA"]]@data` 

```R
pbmc <- NormalizeData(pbmc, normalization.method = "LogNormalize", scale.factor = 10000)
# 当然其中都是默认参数，于是直接写这种也是可以的
pbmc <- NormalizeData(pbmc)
```

这一步要因人而异，可以看：https://bioinformatics.stackexchange.com/questions/5115/seurat-with-normalized-count-matrix 假入有一个TPM的count矩阵，那么就可以不需要使用`Seurat::NormalizeData()`操作了，因为TPM已经根据测序深度进行了归一化，只需要进行log降一下维度即可。如果后续进行`ScaleData`操作，程序会检测是否使用了Seurat提供的标准化方法，如果我们使用自己的的归一化数据，那么就可能出现一个warning提醒，不过到时候不想被提醒，可以设置`check.for.norm =F` 

#### 鉴定差异基因HVGs(highly variable features)

意思就是：在细胞与细胞间进行比较，选择表达量差别最大的(也即是同一个基因在有的细胞中表达量很高，同时在部分细胞中表达很少)，利用`FindVariableFeatures`函数，会计算一个`mean-variance `结果，也就是给出表达量均值和方差的关系并且得到`top variable features`。

那么关于如何计算这些`top variable features`，这个函数是有三种算法的，分别是：

- (默认) vst：对方差(variance)和均值(mean)都取log值+loess线性拟合
- `mean.var.plot`方法：average expression & dispersion (for each feature) + z-scores
- `dispersion`方法： the highest dispersion values

```R
pbmc <- FindVariableFeatures(pbmc, selection.method = "vst", nfeatures = 2000)
top10 <- head(VariableFeatures(pbmc), 10)

# 分别绘制带基因名和不带基因名的
plot1 <- VariableFeaturePlot(pbmc)
plot2 <- LabelPoints(plot = plot1, points = top10, repel = TRUE)
CombinePlots(plots = list(plot1, plot2))
```

![](https://upload-images.jianshu.io/upload_images/9376801-b2c84309894b6f74.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

> 刘小泽写于19.8.19
> 主要介绍官方教程PBMC 3K的下游分析

### 数据标准化（scale）

它的目的是实现数据的线性转换(scaling)，这也是降维处理(如PCA)之前的标准预处理。
主要利用了`ScaleData`函数，回忆一下，之前归一化（normalize）这里做的操作是log处理，它是对所有基因的表达量统一对待的，最后放在了一个起跑线上。但是为了真正找到差异，我们还要基于这个起跑线，考虑不同样本对表达量的影响

它做了：

- 将每个基因在所有细胞的表达量进行调整，使得每个基因在所有细胞的表达量**均值为0**

- 将每个基因的表达量进行标准化，让每个基因在所有细胞中的表达量**方差为1**

```R
# 先使用全部基因
all.genes <- rownames(pbmc)
> length(all.genes)
[1] 13714
pbmc <- ScaleData(pbmc, features = all.genes)
# 结果存储在pbmc[["RNA"]]@scale.data中
```

#### 感觉这一步太慢，能不能加速一点？

这一步主要目的就是下一步的降维，使用全部基因是比较慢。如果想提高速度，可以只对鉴定的HVGs（之前`FindVariableFeatures`设置的是2000个）进行操作，其实这个函数**默认就是对部分差异基因进行操作**，：`pbmc <- ScaleData(pbmc)` 。这样操作的结果中**降维和聚类不会被影响**，但是只对HVGs基因进行归一化，下面画的热图依然会受到全部基因中某些极值的影响。所以**为了热图的结果，还是对所有基因进行归一化比较好。**

#### `ScaleData`的操作过程是怎样的?

看一下帮助文档就能了解：
`ScaleData`这个函数有两个默认参数：`do.scale = TRUE`和`do.center = TRUE`，然后需要输入进行`scale/center`的基因名（默认是取前面`FindVariableFeatures`的结果）。
`scale`和`center`这两个默认参数应该不陌生了，`center`就是对每个基因在不同细胞的表达量都减去均值；
`scale`就是对每个进行center后的值再除以标准差（就是进行了一个**z-score的操作**）

#### 再看一个来自BioStar的讨论：https://www.biostars.org/p/349824/

##### **问题1：**为什么在`NormalizeData() `之后，还需要进行`ScaleData`操作？

前面`NormalizeData`进行的归一化主要考虑了测序深度/文库大小的影响（reads*10000 divide by total reads, and then log），但实际上归一化的结果还是存在基因表达量分布区间不同的问题；而`ScaleData`做的就是在归一化的基础上再添`zero-centres` （mean/sd =》 z-score），将各个表达量**放在了同一个范围中，真正实现了”表达量的同一起跑线“。**

另外，新版的`ScaleData`函数还支持设定回归参数，如（nUMI、percent.mito），来校正一些技术噪音

##### **问题2：** `FindVariableGenes() or RunPCA() or FindCluster()`这些参数是基于归一化`Normalized_Data`还是标准化`Scaled_Data` ？

所有操作都是基于标准化的数据`Scaled_Data` ，因为这个数据是针对基因间比较的。例如有两组基因表达量如下:

```shell
g1 10 20 30 40 50
g2 20 40 60 80 100
```

虽然它们看起来是g2的表达量是g1的两倍，但真要降维聚类时，就要看`scale`的结果：

```R
> scale(c(10,20,30,40,50))
           [,1]
[1,] -1.2649111
[2,] -0.6324555
[3,]  0.0000000
[4,]  0.6324555
[5,]  1.2649111
attr(,"scaled:center")
[1] 30
attr(,"scaled:scale")
[1] 15.81139
> scale(c(20,40,60,80,100))
           [,1]
[1,] -1.2649111
[2,] -0.6324555
[3,]  0.0000000
[4,]  0.6324555
[5,]  1.2649111
attr(,"scaled:center")
[1] 60
attr(,"scaled:scale")
[1] 31.62278

```

二者标准化后的分布形态是一样的（只是中位数不同），而我们后来聚类也是看数据的分布，分布相似聚在一起。所以归一化差两倍的两个基因，根据标准化的结果依然聚在了一起

##### **问题3：** `ScaleData() `在scRNA分析中一定要用吗？

实际上标准化这个过程不是单细胞分析固有的，在很多机器学习、降维算法中比较不同feature的距离时经常使用。当不进行标准化时，有时feature之间巨大的差异（就像👆问题2中差两倍的基因表达量，实际上可能差的倍数会更多）会影响分析结果，让我们误以为它们的数据分布相差很远。

很多时候，我们会混淆`normalization`和`scale`：那么看一句英文解释：

> **Normalization** "normalizes" within the cell for the difference in sequenicng depth / mRNA throughput. 主要着眼于样本的文库大小差异
> **Scaling** "normalizes" across the sample for differences in *range* of variation of expression of genes . 主要着眼于基因的表达分布差异

另外，看`scale()`函数的帮助文档就会发现，它实际上是**对列进行操作：**
![](https://jieandze1314-1255603621.cos.ap-guangzhou.myqcloud.com/blog/2019-08-19-033534.png)

那么具体操作中是要先对表达矩阵转置，然后scale，最后转回来

```R
t(scale(t(dat[genes,])))
```

##### **问题4：**`RunCCA() `函数需要归一化数据还是标准化数据？

通过看这个函数的帮助文档：

```R
RunCCA(object, object2, group1, group2, group.by, num.cc = 20, genes.use,
  scale.data = TRUE, rescale.groups = FALSE, ...)
```

显示`scale.data = TRUE`，那么就需要标准化后的数据

> 怎样移除不想要的差异来源？

在进行降维聚类时，主要就是考虑数据差异对分布产生的影响。由于技术误差导致的差异是我们不感兴趣的，排除这一部分其实也在上面的问题1中提到了，如果说不想线粒体污染导致的差异影响整体差异分布：

```R
pbmc <- ScaleData(pbmc, vars.to.regress = "percent.mt")
```

这个函数仅针对初级用户，如果想深入探索技术误差的排除，官方给出了另一个专业版函数：`SCTransform`，它也包含`vars.to.regress`这个选项。详细的说明在：https://satijalab.org/seurat/v3.0/sctransform_vignette.html

---

### 进行线性降维

最常用的线性降维就是PCA方法，**使用标准化的数据**

它默认选择之前鉴定的差异基因（2000个）作为input，但可以使用`features`进行设置；默认分析的主成分数量是50个

```R
# 这里就使用默认值
pbmc <- RunPCA(pbmc, features = VariableFeatures(object = pbmc))
# 结果保存在reductions 这个接口中
```

#### 检查下PCA结果：

```R
> print(pbmc[["pca"]], dims = 1:3, nfeatures = 5)
# 或者用 print(pbmc@reductions$pca, dims = 1:3, nfeatures = 5)
PC_ 1 
Positive:  CST3, TYROBP, LST1, AIF1, FTL 
Negative:  MALAT1, LTB, IL32, IL7R, CD2 
PC_ 2 
Positive:  CD79A, MS4A1, TCL1A, HLA-DQA1, HLA-DQB1 
Negative:  NKG7, PRF1, CST7, GZMB, GZMA 
PC_ 3 
Positive:  HLA-DQA1, CD79A, CD79B, HLA-DQB1, HLA-DPB1 
Negative:  PPBP, PF4, SDPR, SPARC, GNG11 
```

#### 用`VizDimLoadings`对print的结果可视化：

```R
VizDimLoadings(pbmc, dims = 1:3, reduction = "pca")
```

![对print的结果可视化](https://jieandze1314-1255603621.cos.ap-guangzhou.myqcloud.com/blog/2019-08-19-035810.png)

#### 用`DimPlot`进行降维后的可视化

提取两个主成分（默认前两个，当然可以修改`dims`选项）绘制散点图

```R
DimPlot(pbmc, reduction = "pca")
```

![](https://jieandze1314-1255603621.cos.ap-guangzhou.myqcloud.com/blog/2019-08-19-040130.png)

这个图中每个点就是一个样本，它根据PCA的结果坐标进行画图，这个坐标保存在了`cell.embeddings`中：

```R
> head(pbmc[['pca']]@cell.embeddings)[1:2,1:2]
                     PC_1       PC_2
AAACATACAACCAC -4.7296855 -0.5184265
AAACATTGAGCTAC -0.5174029  4.5918957
```

其实还支持定义分组：根据`pbmc@meta.data`中的`ident`分组：

```R
> table(pbmc@meta.data$orig.ident)

pbmc3k 
  2638 
# 当然这里只有一个分组
```

#### 探索异质性来源—`DimHeatmap` 

每个细胞和基因都根据PCA结果得分进行了排序，默认画前30个基因（`nfeatures`设置），1个主成分(`dims`设置)，细胞数没有默认值（`cells`设置）

```R
DimHeatmap(pbmc, dims = 1:15, cells = 500, balanced = TRUE)
# 其中balanced表示正负得分的基因各占一半
```

![](https://jieandze1314-1255603621.cos.ap-guangzhou.myqcloud.com/blog/2019-08-19-063348.png)

> 降维的真实目的是尽可能减少具有相关性的变量数目，例如原来有700个样本，也就是700个维度/变量，但实际上根据样本中的基因表达量来归类，它们或多或少都会有一些关联，这样有关联的一些样本就会聚成一小撮，表示它们的”特征距离“相近。最后直接分析这些”小撮“，就用最少的变量代表了实际最多的特征

既然每个主成分都表示具有相关性的一小撮变量（称之为”metafeature“，元特征集），那么问题来了：

#### **降维后怎么选择合适数量的主成分来代表整个数据集**？

选10个？20个？50个？最简单的方法就是：

```R
ElbowPlot(pbmc)
```

它是根据每个主成分对总体变异水平的贡献百分比排序得到的图，我们主要关注”肘部“的PC，它是一个转折点（也即是这里的PC9-10），说明取前10个主成分可以比较好地代表总体变化

![](https://jieandze1314-1255603621.cos.ap-guangzhou.myqcloud.com/blog/2019-08-19-071354.png)

但官方依然建议我们，下游分析时多用几个成分试试（10个、15个甚至50个），如果起初选择的合适，结果不会有太大变化；另外推荐**开始不确定时要多选一些主成分**，也不要直接就定下5个成分进行后续分析，那样很有可能会出错

---

### 细胞聚类

Seurat 3版本提供了`graph-based` 聚类算法，参考两篇文章：[SNN-Cliq, Xu and Su, Bioinformatics, 2015\]](http://bioinformatics.oxfordjournals.org/content/early/2015/02/10/bioinformatics.btv088.abstract)  和 [PhenoGraph, Levine *et al*., Cell, 2015\]](http://www.ncbi.nlm.nih.gov/pubmed/26095251). 简言之，这些`graph`的方法都是将细胞嵌入一个算法图层中，例如：`K-nearest neighbor (KNN) graph` 中，每个细胞之间的连线就表示相似的基因表达模式，然后按照这个相似性将图分隔成一个个的高度相关的小群体（名叫`‘quasi-cliques’ or ‘communities’`）；

![KNN解释（https://slideplayer.com/slide/7087250/）](https://jieandze1314-1255603621.cos.ap-guangzhou.myqcloud.com/blog/2019-08-19-073441.png)

又比如，在`PhenoGraph`中，首先根据PCA的欧氏距离结果，构建了KNN图，找到了各个近邻组成的小社区；然后根据近邻中两两住户（细胞）之间的交往次数（`shared overlap`）计算每条线的权重（术语叫`Jaccard similarity`） 。这些计算都是用包装好的函数`FindNeighbors()` 得到的，它的输入就是前面降维最终确定的主成分

```R
# 因为我们前面挑选了10个PCs，所以这里dims定义为10个
pbmc <- FindNeighbors(pbmc, dims = 1:10)
```

> 以上是凭个人理解发挥，大概就是这么个意思，不涉及算法层面推导

得到权重后，为了对细胞进行聚类，使用了计算模块的算法（默认使用`Louvain`，另外还有包括SLM的其他三种），使用`FindClusters()`  进行聚类。其中包含了一个`resolution`的选项，它会设置一个”间隔“值，该值越大，间隔越大，得到的cluster数量越多

> **怎么理解这个`resolution`参数？**
> 可以想象成配眼镜的时候，镜片度数越高，分辨率越高，看的越清楚，看到的细节越丰富（cluster越多）；反之，如果分辨率调的很低，结果就看的模模糊糊一大坨

一般来说，这个值在细胞数量为3000左右时设为`0.4-1.2` 会有比较好的结果

```R
pbmc <- FindClusters(pbmc, resolution = 0.5)
# 结果聚成几类，用Idents查看
> head(Idents(pbmc), 5)
AAACATACAACCAC AAACATTGAGCTAC AAACATTGATCAGC AAACCGTGCTTCCG 
             1              3              1              2 
AAACCGTGTATGCG 
             6 
Levels: 0 1 2 3 4 5 6 7 8
# 发现3000细胞，设resolution=0.5时，得到Number of communities: 9
```

---

### 另一种降维方法—非线性降维（UMAP/tSNE）

> **线性降维PCA（1933）**一个特点就是：它将高维中不相似的数据点放在较低维度时，会尽可能多地保留原始数据的差异，因此导致这些不相似的数据相距甚远，大多的数据重叠放置
> **tSNE（2008）兼顾了高维降维+低维**展示，降维后的那些不相似的数据点依然可以放得靠近一些，保证了点既在局部分散，又在全局聚集
> 后来开发的**UMAP（2018）**相比tSNE又能展示更多的维度(参考文献：https://sci-hub.tw/https://doi.org/10.1038/nbt.4314)

做个对比：

![https://www.nature.com/articles/nbt.4314](https://jieandze1314-1255603621.cos.ap-guangzhou.myqcloud.com/blog/2019-08-19-082039.png)

```R
# 使用UMAP
pbmc <- RunUMAP(pbmc, dims = 1:10)
DimPlot(pbmc, reduction = "umap") # 还可以设置label = TRUE让数字显示在每个cluster上

# 对计算过程很费时的结果保存一下
save(pbmc, file = 'pbmc_umap.Rdata')
```

#### 需要注意的点：

注意1：tSNE算法具有随机性，为了保证重复性，要固定一个随机种子

##### 注意2：如何在R中使用UMAP？

> 以下测试是在个人电脑上，需要管理员权限

因为UMAP是基于Python的，如果要用它，必须先保证有一个python的安装环境，先试验一下：

```R
> reticulate::py_install(packages ='umap-learn')
# 结果报错，说让我升级一下virtualenv(mac和linux默认使用virtualenv，而Windows只能使用conda)
Error: Prerequisites for installing Python packages not available.

Execute the following at a terminal to install the prerequisites:

$ sudo /usr/local/bin/pip install --upgrade virtualenv
```

然后直接在Rstudio中打开`Terminal`，输入：`sudo /usr/local/bin/pip install --upgrade virtualenv` 

![](https://jieandze1314-1255603621.cos.ap-guangzhou.myqcloud.com/blog/2019-08-19-082728.png)

再运行一遍上面的R代码（成功与否取决于个人的网络，因为它要通过外链去下载，wlan有限制的话就用个人热点吧）：

![](https://jieandze1314-1255603621.cos.ap-guangzhou.myqcloud.com/blog/2019-08-19-082834.png)

---

### 找差异表达基因（cluster biomarker）

> 目的是根据表达量不同这个特征而分出不同细胞群的基因们（就是找具有生物学意义的HVGs=》即biomarkers）
> marker可以理解成标记，就是一看到有这些基因，就说明是这个细胞群体

主要利用`FindAllMarkers()`函数，它默认会对单独的一个细胞群与其他几群细胞进行比较找到**正、负表达biomarker（这里的正负有点上调、下调基因的意思**；正marker表示在我这个cluster中表达量高，而在其他的cluster中低）

```R
# 找到cluster1中的marker基因
cluster1.markers <- FindMarkers(pbmc, ident.1 = 1, min.pct = 0.25)
head(cluster1.markers, n = 5)
```

上面这个代码中`min.pct`的意思是：一个基因在任何两群细胞中的占比最低不能低于多少，当然这个可以设为0，但会大大加大计算量

另外，如果**要比较cluster5和cluster0、cluster3的marker基因：**

```R
cluster5.markers <- FindMarkers(pbmc, ident.1 = 5, ident.2 = c(0, 3), min.pct = 0.25)
head(cluster5.markers, n = 5)
```

**一步到位的办法`FindAllMarkers`**（对所有cluster都比较一下，并只挑出来正表达marker）：

```R
pbmc.markers <- FindAllMarkers(pbmc, only.pos = TRUE, min.pct = 0.25, logfc.threshold = 0.25)
# 这一步过滤好好理解(进行了分类、排序、挑前2个)
pbmc.markers %>% group_by(cluster) %>% top_n(n = 2, wt = avg_logFC)
```

**Seurat提供了多种差异分析的检验方法**，在`test.use`选项中设置（详见：[DE vignette](https://satijalab.org/seurat/v3.0/de_vignette.html) ）例如：

```R
cluster1.markers <- FindMarkers(pbmc, ident.1 = 0, logfc.threshold = 0.25, test.use = "roc", only.pos = TRUE)
```

##### 多种可视化方法：

- `VlnPlot`：用小提琴图对某些基因在全部cluster中表达量进行绘制

  ```R
  VlnPlot(pbmc, features = c("MS4A1", "CD79A"))
  ```

  ![](https://jieandze1314-1255603621.cos.ap-guangzhou.myqcloud.com/blog/2019-08-19-095336.png)

- `FeaturePlot`：最常用的可视化=》将基因表达量投射到降维聚类结果中

  ```R
  FeaturePlot(pbmc, features = c("MS4A1", "GNLY", "CD3E", "CD14", "FCER1A", "FCGR3A", "LYZ", "PPBP", "CD8A"))
  ```

  ![](https://jieandze1314-1255603621.cos.ap-guangzhou.myqcloud.com/blog/2019-08-19-095641.png)

另外还有：`RidgePlot`, `CellScatter`, and `DotPlot`

- `DoHeatmap` 对给定细胞和基因绘制热图，例如对每个cluster绘制前20个marker

  ```R
  top10 <- pbmc.markers %>% group_by(cluster) %>% top_n(n = 10, wt = avg_logFC)
  DoHeatmap(pbmc, features = top10$gene) + NoLegend()
  ```

---

### 赋予每个cluster细胞类型

重点在于背景知识的理解，能够将marker与细胞类型对应起来，例如这里从各个cluster中找到的marker对应到了不同的细胞（这一过程是比较考验课题理解程度的）：

![](https://jieandze1314-1255603621.cos.ap-guangzhou.myqcloud.com/blog/2019-08-19-100319.png)

有了这个，绘图就很简单：

```R
new.cluster.ids <- c("Naive CD4 T", "Memory CD4 T", "CD14+ Mono", "B", "CD8 T", "FCGR3A+ Mono", "NK", "DC", "Platelet")
names(new.cluster.ids) <- levels(pbmc)
pbmc <- RenameIdents(pbmc, new.cluster.ids)
DimPlot(pbmc, reduction = "umap", label = TRUE, pt.size = 0.5) + NoLegend()
```

![](https://jieandze1314-1255603621.cos.ap-guangzhou.myqcloud.com/blog/2019-08-19-100435.png)



























