# Instant Gratification Solution (47th place)

I entered the Instant Gratification competition when it was 16 days to go, so there was already a lot of important information available in Kernels and Discussions. Shortly, in the Instant-Gratification competition we deal with a dataset of the following structure:
    
- [Data seemed to be generated by groups](https://www.kaggle.com/c/instant-gratification/discussion/92930#latest-541375) corresponding to the column **wheezy-copper-turtle-magic**, which means that we have 512 independent datasets of the size approximately 512 by 255 for training our models.

- Out of 255 features [only features with high variance supposed to be important](https://www.kaggle.com/fchmiel/low-variance-features-useless/). The number of important features varies from 33 to 47.

- [QDA proved to be a good modeling approach](https://www.kaggle.com/speedwagon/quadratic-discriminant-analysis) for this case and can give AUC of 0.966. 

- This high quality can be explained by the fact that the [data probably was generated with make_classification function](https://www.kaggle.com/mhviraf/synthetic-data-for-next-instant-gratification).

Knowing that let's look closer at what make_classification function has under the hood. This function has the following parameters:

- n_samples: The number of samples.
- n_features: The total number of features.
- n_informative: The number of informative features.
- n_redundant: The number of redundant features, linear combinations of informative features.
- n_repeated: The number of duplicated features.
- n_classes: The number of classes.
- n\_clusters\_per_class: The number of clusters per class.
- weights: The proportions of samples assigned to each class.
- flip_y: The fraction of samples whose class are randomly exchanged.
- class_sep: Larger values spread out the clusters/classes and make the classification task easier.
- hypercube: Parameter corresponding to the placement of the cluster centroids.
- shift: Shift of the feature values.
- scale: Scaling of the features by the specified value.
- shuffle: Shuffling the samples and the features.
- random_state: Random state.

I invite you to follow the sequence of my thoughts about it.

***

### Step 1. Size and features (first 6 parameters of make_classification function).

From the data structure (512 groups and 255 variables) and size (262144 = 512 * 512), we can assume that **n_features=255** and **n_samples=1024** were taken to generate both training and test data sets (both private and public).

Now, let's talk about features. First of all, repeated features are exact copies of important features, and because we don't see such columns in competition data we'll set **n_repeated=0**. Let's generate data set with make_classification function and following parameters:

~~~~
fake_data = make_classification(
    n_samples=1024, 
    n_features=255, 
    n_informative=30, 
    n_redundant=30, 
    n_repeated=0, 
    n_classes=2, 
    shuffle=False)
~~~~

By setting shuffle=False, I force first 30 columns to be informative, next 30 to be redundant and others to be just noise. By doing it 1000 times lets look at the distribution of the standard deviation of the informative, redundant and random features.

<a href="https://ibb.co/HG8WWDR"><img src="https://i.ibb.co/L97LLzB/features-std.png" alt="features-std" border="0" width="800px"></a>

We see clear difference in standard deviation for informative, redundant and random features, that's why selecting important features with 1.5 threshold works so well. Moreover, there are no features in the competition data that have std bigger than 5, which leads us to an assumption that **n_redundant=0**.

Number of important features ranges from 33 to 47, so **n_informative** is in **{33,...,47}**. Number of classes is obviously **n_classes=2**. 

***

### Step 2. Shift, Scale and randomness (last 4 parameters of make_classification function).

Because mean and standard deviation for random columns are 0 and 1 respectively, we can assume that shift and scale were set to default **shift=0**, **scale=1**. On top of that, standard deviation for important columns also look the same for competition data and for data that we've just generated. So it's a pretty promising assumption and we can go further. Parameters **shuffle** and **random_state** are not so important, because they don't change the nature of the data set.

***

### Step 3. Most interesting parameters (why QDA is not so perfect?).

Parameters **n_clusters_per_class, weights, class_sep, hypercube** are the ones that we are not very sure about, especially **n_clusters_per_class**. Weights look to be the same, because target seemed to be balanced. What probably makes it imposible to get 100% accurate model is that **flip_y>0**, and we can not predict this random factor in any case. However, what we can try is to build a model that can perfectly predict when **flip_y=0** and leave the rest for the luck.

QDA shown to be a very good approach, but let me show you the case when it's working not so good:

<a href="https://ibb.co/bKhGFTq"><img src="https://i.ibb.co/NyhGr5Q/4-components.png" alt="4-components" border="0" width="800px"></a>

Data set was generated with make_classification:

~~~~
make_classification(
    n_samples=1000, n_features=2, n_informative=2, n_redundant=0, 
    n_classes=2, n_clusters_per_class=2, flip_y=0.0, class_sep=5,
    random_state=7, shuffle=False
)
~~~~

Now, lets look how QDA algorithm can hadle it:

<a href="https://ibb.co/pWRTdtD"><img src="https://i.ibb.co/RhBKcxn/qda-model.png" alt="qda-model" border="0" width="800px"></a>

Pretty good, however, we see that it doesn't see this structure of four clusters in the data. From make_classification documentation one can find that these clusters are gaussian components, so the best way to find them is to apply Gaussian Mixture model. 

_Note: Gaussian Mixture model will only give you clusters, you need to bind these clusters to one of the classes 0 or 1. That you can do by looking at the proportion of the certain class in each cluster and assigning cluster to the dominant class._

Lets look how GMM algorithm performed at this data:

<a href="https://ibb.co/Q9sPPxm"><img src="https://i.ibb.co/0qPCCzJ/gm-model.png" alt="gm-model" border="0" width="800px"></a>

Nearly perfect! Let's move to the step 4.

***

### Step 4. Effect of flipping the target

Knowing that we can build nearly the best classifier, what is the effect of flipping the target? Suppose that your classes are perfectly separable and you can assign probabilities that will lead to a perfect AUC. Let's see an example of 10000 points, 5000 in each class:

Classes = \[0, 0, ..., 0, 1, ..., 1, 1\]

Predictions = \[0.0000, 0.0001, ..., 0.9998, 0.9999\]

Lets flip the target value for 2.5% (250) of the points (1) in the middle, (2) on the sides, (3) randomly, and look at the AUC:

<a href="https://ibb.co/M1cNwkw"><img src="https://i.ibb.co/c86Tfhf/flipping-effect.png" alt="flipping-effect" border="0" width="800px" align="center"></a>

We see that the impact from flips can drammatically change the result. However, let's face two facts:

1. We can not predict this randomness.
2. We can assume that the flips are evenly spread across our predictions.

Let's also look at the distribution of AUC for randomly flipped target.

<a href="https://ibb.co/PjBttyk"><img src="https://i.ibb.co/F3Gnn1p/auc.png" alt="auc" border="0" width="800px"></a>

It means that results on unseen data can deviate by more than 0.005 in AUC even for the perfect model. So our strategy will be not to find the parameters to raise on the LB of the competition, but to find the parameters that can perfectly predict target for most of the data set.

***

### Step 5. Experiments with synthetic data

How we can build a model that perfectly predicts data and not overfit to it? Because we have a strong hypothesis on how data was created we can tune model parameters such that it will predict perfectly for most of the synthetically generated data sets. Not showing competition data to it will save us from overfitting.

We will be generating a data set of the same structure as the competition data:

- **n_samples=768**, 512 for training and 256 for test
- **n_features=255**
- **n_important=33**
- **n_clusters_per_class=3**
- by variating **weights**, we will change the propotion of the dominant class from 0.5 to 0.6
- **flip=0** to experiment if GM model can be completely correct
- **class_sep** will be a random number between 0.9 and 1
- **hypercube** will be chosen randomly between True and False

Next thing to do is to generate hundreds of such data sets and optimize parameters of the GMM.

For the first attempt we'll set the parameter n_components in the GMM to 6, because we know that there will be 3 clusters per each of the 2 classes, and do the modelling with other parameters set to default. Let's look at the minimum, maximum and mean score that we can get for 100 different data sets:

```
Min score = 0.44783168741949336
Mean score = 0.8159223370950822
Max score = 1.0
```

Wow! Sometimes we even reached the perfect classifier. Let's try to optimize some of the GMM parameters. It has parameters n_init and init_params which means that GMM model will be run several times starting from random initializations and then it selects the best iteration. Let's try to set n_init to 20 and init_params to random.

```
Min score = 1.0
Mean score = 1.0
Max score = 1.0
```

Now it's perfect! Let's try it for more than 33 features (for example, let's take 47):

```
Min score =  0.562083024462565
Mean score =  0.7917138432059158
Max score =  1.0
```

For 47 features it worked not so well. The secret sauce is the regularization of the covariance matrix. Let's look at the model quality for different values of reg_covar parameter of GM model:

```
reg_covar = 1: Min score = 0.6936146721462308, Mean score = 0.8782139741161032
reg_covar = 5: Min score = 0.991250080422055, Mean score = 0.9998679217326921
reg_covar = 10: Min score = 0.9987766087594813, Mean score = 0.999945378120418
```

To tell you the short story, by a simple grid search for the best reg_covar parameter one can find that more features require larger reg_covar for better performance. By doing so you can find that the good parameters are:

- 33 features: 2.0,
- 34 features: 2.5,
- 35 features: 3.5,
- 36 features: 4.0,
- 37 features: 4.5,
- 38 features: 5.0,
- 39 features: 5.5,
- 40 features: 6.0,
- 41 features: 6.5,
- 42 features: 7.0,
- 43 features: 7.5,
- 44 features: 8.0,
- 45 features: 9,
- 46 features: 9.5,
- 47 features: 10.

We still have one thing not clear.

### Important note: how do we know the number of clusters in advance?

We were building models in the assumption that there are 3 clusters per class, but how can we know it? Well, the answer is pretty simple, right choice of the n_components will give better results:

#Clusters per class = 4, #Components of GMM = 4: AUC = 0.7170955882352941
#Clusters per class = 4, #Components of GMM = 6: AUC = 0.8263480392156862
#Clusters per class = 4, #Components of GMM = 8: AUC = 0.9587009803921569

#Clusters per class = 3, #Components of GMM = 4: AUC = 0.7400829259236339
#Clusters per class = 3, #Components of GMM = 6: AUC = 1.0
#Clusters per class = 3, #Components of GMM = 8: AUC = 0.9844049755554181

#Clusters per class = 2, #Components of GMM = 4: AUC = 1.0
#Clusters per class = 2, #Components of GMM = 6: AUC = 1.0
#Clusters per class = 2, #Components of GMM = 8: AUC = 0.9924635532493205

For me it seemed like for most of the groups the best option was taking 3 clusters per class, sometimes 2 was slightly better (but it might be the effect of the flipped target). I assumed that this parameter was set to 3 for all groups.

***

### Conclusion

Parameters found in step 5 will give you an AUC of 0.99 minimum (for the true target of course)! So we will use these parameters to build a model for competition data. 

Final kernel with the model can be found [here]().
Jypyter Notebook for this analysis can be found [here]().

Thanks for reading!
