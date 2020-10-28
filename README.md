See: `Replication_Predicting_Disk_Replacement_towards_Reliable_Data_Centers.ipynb` for implementation!

Paper: https://www.kdd.org/kdd2016/papers/files/adf0849-botezatuA.pdf

# Discussion
 
## What are the most time-consuming or labor-intensive aspects of the replication process? How could we accelerate this part of the development?
 
The most labor and time-consuming aspect of this process is creating compact representations of the time-series data. The process consists of:
 
For each SMART attribute:
1. For a subset of the data, determine the timestamps where the timeseries of a healthy and failed hard drive begin to significantly differ. Call these timestamps change points.
2. Determine the window size to perform exponential smoothing over. This window size is the median of all change points.
3. Perform exponential smoothing.
 
In order to get a stable estimate for the median of all change points, you need to calculate at least 30 change points. This is very expensive as it requires training a bayesian model each time. So if it takes ~30 seconds to train a model, and you have 40 SMART attributes, then overall just determining the window size takes 10 hours (and that is only per model).

To accelerate this process I'd try to move away from requiring hard defined window sizes for each individual SMART attribute. LSTM / CNN networks would do very well at this. The big advantage of those types of algorithms is that they do not have separate stages for creating representations and classifying these representations. The two steps are linked, one informs the other. 

 
## What are key edge case scenarios you'd like to run to ensure fidelity once deployed across various data centres in the wild?
 
In the wild deployed machine learning algorithms are particularly susceptible to distributional shift. That is if you have one data distribution during training but then during deployment, the distribution slightly shifts, degrading the performance of the algorithm. A key edge case scenario I'd like to explore is to what extent the distribution shifts, and exactly what causes it to shift. I'd expect that the hard-drives degrade differently depending on their exact usage. For example if there's a lot of IO operations, the SMART attributes may shift. If we do not have a good understanding, or detection algorithm, we'd have to re-train each algorithm periodically on the newest data. This could be prohibitively expensive. So detecting them, and fine-tuning on the new data if necessary will be crucial.
 
An edge-case which even occurred in the BackBlaze dataset is the addition or removal of SMART attributes. The ST4000DM000 model has a different set of SMART attributes in 2013 and 2014 than in 2015-2020. If useful attributes are added, it would be important to be able to use the new attribute while profiting from all the data which was previously collected. .
 
## What are the limitations of IBM paper ML methodology?
 
The key limitation of this algorithm is the necessity for a large window size. The median of all change points per SMART attribute is used to determine the beginning of the window. This leads to some windows being 70 time steps long, requiring the model to have to have been running for 70 days, before the algorithm can be applied to it.
 
The base-case solution for this would be to choose an arbitrary and relatively small window size. This would eliminate the most computationally expensive stage of this algorithm and allow it to be applied to hard drives that haven'€t operated for as long. However doing some quick analysis, I reduced the window size to 10 for each SMART attribute, and thereby decreased the f1 score by about 3\%. 
 
The more advanced solution would be to use a variable length encoding LSTM network. The large advantage would be that it:
1. Can operate on a sequence of any length.
2. Can capture higher order patterns (like periodicity) unlike simple exponential smoothing.
 
Another major limitation (as mentioned in the paper) is exponential smoothing. It assumes that each subsequent time-step is more important (i.e that $`S_t`$ is less important than $`S_{t+1}`$). This assumption does not necessarily hold if for example a change in periodicity is indicative of failure.
 
The causal inference algorithm to detect change points in the data is very expensive, and not particularly reliable. I re-ran the algorithm, initialized with a different random seed on the same data, and got wildly different results.
 
In the paper they downsample the healthy dataset by fitting Kmeans to it and choosing the data samples that are closest to a centroid. A problem is that the Kmeans algorithm does not necessarily fit a spherical cluster to the dataset; the clusters can also be elliptical. By just taking the distance to the centroid, you may choose a point along a dimension which has very little expected variance, and thus is not representative of the cluster (see picture below where the blue circle is the cluster, and the two dots are training samples). Instead you should use mahalanobis distance, which takes the variance along each dimension into account.
 
![cluster_illustration.png](https://github.com/fin-vermehr/Replication-Predicting-Disk-Replacement-towards-Reliable-Data-Centers/blob/main/cluster_illustration.png)
 
## What are other avenues of research and modification to improve performance?
 
A major research avenue to pursue is creating better, richer representations. As discussed above, this would most likely entail using LSTM networks. At the moment, the classification accuracy does not inform the compact representation mechanism. In LSTM or CNN networks (both of which are very valid approaches to this problem) the classification loss adjusts the encoding mechanism. In our case the smoothing parameter $`\alpha`$ is determined through MLE. One could treat it as a tuneable parameter, and re-run the hyper-parameter tuning session multiple times. For this architecture that would be prohibitively expensive though. In networks, the gradient of the loss is used to adjust all the encoding parameters, making it a lot more powerful.
 
Another important topic is to look more into down-sampling techniques. I do not understand why you would keep training samples that are closest to a centroid. My understanding is that that would over-sample similar data and not provide the model with a training set that is representative of the true data distribution. Instead you should sample from the area of the cluster, while disrespecting its density. This would avoid over-sampling nearly identical data-points.
 
As became evident by replicating the IBM paper on hard drive models that occur less frequently in the BackBlaze dataset, the performance drops steeply. It would be good to explore transfer learning between different model types. If we could have one general model that is trained on all data, and then fine-tuned on specific models, it would reduce the total number of unique algorithms, and be potentially a lot more powerful. Given that in the IBM paper they were so successful at transfer-learning between different models of the same series with such a basic transfer-learning approach, I'd be surprised if there wasn't an advantage even between series.

Another important research step is finding representative metrics. It's not trivially true that F1, recall and precision is what we care about most.
