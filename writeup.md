# **Traffic Sign Recognition** 

## Writeup Template


**Build a Traffic Sign Recognition Project**

The goals / steps of this project are the following:
* Load the data set (see below for links to the project data set)
* Explore, summarize and visualize the data set
* Design, train and test a model architecture
* Use the model to make predictions on new images
* Analyze the softmax probabilities of the new images
* Summarize the results with a written report
 
### Project files

The project page is on [Github](https://github.com/pvishal-carnd/Traffic-Sign-Classifier). This writeup can at [writeup.md](writeup.md) and the Jupyter notebook, that has all the code can be accessed at [Traffic_Sign_Classifier.ipynb](Traffic_Sign_Classifier.ipynb). 

### Data Set Summary & Exploration
---
The dataset exploration was done primarily with `numpy`. `pandas` was used to read in the `csv` file with sign descriptions. The training, validation and test set has already been provided. The following list shows a summary.   


* The size of training set is `34799`
* The size of the validation set is `4410`
* The size of test set is `12630`
* The shape of a traffic sign image is `32x32`
* The number of unique classes/labels in the data set is `43`

##### Visualization of the dataset.

Each image is a 32x32x3 RGB array with integer values in the range `[0 255]'. We plot a random images of a few classes to see how they look like. 

![Sample Images](doc-images/datavis.png)

Next, we plot the histogram of the number of images for each class.

![Data histograms](doc-images/data-histograms.png)

From the shapes of each of the three distributions,  the validation and the test datasets, at least at the outset do seem to be drawn from the same distribution. However, the dataset is unbalanced. Certain classes are significantly under-represented with respect to the number of available samples in the training set. We will keep an eye out for these classes when we analyze the prediction model performance at a later stage.  

### Pre-processing

As a first step of the preprocessing pipeline, the images were converted to grayscale. This step is not strictly required but we noticed small improvement in the overall validation set performance with this conversion.   In their paper, Pierre Sermanet and Yann LeCun also report a marginal improvement with grayscale conversions. Having said that, certain classes of sign did significantly better after grayscale conversion. We will address this issue in a later section when we analyze the model performance.

After grayscaling, the images were normalized. We did this by linearly mapping all the pixel values to lie between `0` and `1`. Apart from helping the learning by keeping the means small and the cost function well distributed along all axes, this normalization steps also helps make the algorithm invariant to brightness changes. The image histograms become flatter and this helps bring up faint features. However, this method fails to work under cases where parts of the image are overexposed (perhaps due to the sky in the background). Local adaptive histogram equalization, local normalization or applying a non-linear transformation like gamma correction can potentially help such cases. We, nevertheless, continued with our simple min-max normalization scheme and the results are as follows:   

![Preprocessing results](doc-images/preproc.png)

The result are as expected. The variation in the brightnesses of has been reduced and the darker features have been brought up. Also as expected, our simplistic linear normalization scheme has a very little effect on signs that have sky in the background.   

### Data augmentation

There are two reasons for considering data augmentation for this model:
1. The dataset is highly unbalances. In some cases, there are only a little more than 200 images per class. We could therefore augment those under-represented classes with generated data.
2. As we shall see later, our models mostly overfit the data. In such cases, there is almost always room for more variation in the training set. We bring about this variation by generating jittered data.

We can choose among a very large set of transformations to create an augmented dataset. We however, focus on transformations that are realistic for our case. For example, flipping will not help as it completely changes most of the traffic signs. Tranformations like warp will not occur in practice and neither will shear (expect probably as a rolling shutter artifact at high speeds). Brightness changes, while very possible, get normalized away in our pre-processing step. Any color-based transformations are lost in our conversion to grayscale. These transformations are unlikely to improve the field performance of our model.  

We therefore, limit ourselves to the following transformations.  
- Translation
- Rotation
- Constrast 
- Crops
- Gaussion blur
- Additive noise
- Perspective

We use a library `imgaug` for this task. This library allows us to easily create a random augmentation pipeline and apply it to large image sets. It is also easily available on `PyPi` and can be simply installed with 

```
pip install imgaug
```

The following code snippet sets up this pipeline for us. The code comments are self-explanatory. We don't translate a lot of images because convolutions are fairly robust to translations. 

```
augPipeline = iaa.Sequential([
	# Crop all images with random percentage from 0 to 22%
    iaa.Crop(percent=(0, 0.22)), # random crops
    
	# With a probability of 0.3, apply either a random Gaussian 
    # blur or add random additive noise. 
    iaa.Sometimes(0.3, 
    iaa.OneOf([
    iaa.GaussianBlur(sigma=(0, 1)),
    iaa.AdditiveGaussianNoise(loc=0, scale=(0 ,0.07*255))
    ])),
    
    # Strengthen or weaken the contrast in some images 
    iaa.Sometimes(0.25, iaa.ContrastNormalization((0.75, 1.5))),
    
    # Translate some images
    iaa.Sometimes(0.1, iaa.Affine(translate_percent={"x": (-0.1, 0.1), "y": (-0.1, 0.1)})),
    
	# With a high probability, apply a perspective transform
    iaa.Sometimes(0.97,
                  iaa.PerspectiveTransform(scale=(0, 0.1))
                 ),

	# Apply affine transformations to each image.
    iaa.Affine(
        scale={"x": (0.9, 1.1), "y": (0.9, 1.1)},
        rotate=(-6, 6),
        shear=(-15, 15)
    )
], random_order=True) # Apply the above transformations in a random order
```
The results of this pipeline are shown for a single randomly selected image.

![Augmentation pipeline test](doc-images/augmented.png)

We apply this augmentation mostly only to the under-represented classes to bring up each of their sample size to about 1500 images per class. We confirm this by looking at the histogram of the augmented dataset.

![Augmented histogram](doc-images/augmented-histogram.png)

### The Model Architecture
---
The final model architecture is as shown the the following figure. This figure has been generated through `Tensorboard`. The table that follows the model shows the parameters of each layer.

![Model architecture](doc-images/model-arch-tf.png)


| Name      | Layer         		|     Description	        					| 
|:---------:|:---------------------:|:---------------------------------------------:| 
| Input     | Input         		| 32x32x1 grayscale image						|
| CONV1     | Convolution 5x5     	| 1x1 stride, same padding, Outputs 32x32x16 	|
| .       	| Max Pooling 2x2       | Output 16x16x16                               |
| CONV2     | Convolution 5x5		| 1x1 stride, same padding, Outputs 16x16x32 	|
| .         | Max Pool 2x2          | Output 8x8x16                                 |
| MaxPool   | Max pooling 2x2       | Applied on CONV1, 2x2 stride, Outputs 16x16x16|
| flatten   | Fully connected       | Fully connected layer, 3072 units             |
| dropout1  | Dropout       		| Keep probabilty of 0.4                        |
| FC1       | Fully connected       | Fully connected layer, 1024 units             |
| dropout2  | Dropout				| Keep probabilty of 0.4                        |
| FC2       | Fully connected       | Fully connected layer, 43 units               |


Our model has 5 layers - 2 convolutional layers for feature extraction and 3 fully connected layer for classification. The architecture is based on the LeNet-5 model from the [LeNet MNIST lab](https://github.com/udacity/CarND-LeNet-Lab). As in most convolutional neural network models, the number of filters increase in the deeper convolutional layers and the image sizes decrease. Unlike the LeNet-5 model above, we use same padding instead of valid padding. In case of valid padding, features around the edges get less importance. This could reduce performance as our images, certainly not the augmented ones, are not guaranteed to be centered well. 
 
Another highlight of the model is that is it uses multi-scale features for classification as proposed in this paper by Pierre Sermanet and Yann LeCunn. This is achieved by feeding in the output of both the convolutional layers to the classifier. It is easy to see this in the architecture diagram above - the output of `CONV1` is also max-pooled and fed into the fully-connected `flatten` layer. An addition max-pooling is done so that the features of the `CONV1` layer are weighed in similarly to that of `CONV2`. 

### Training and hyperparameters

Our final choice of training hyperparameters is as follows:
```
learning_rate = 0.002
BATCH_SIZE    = 128
EPOCHS        = 50
L2_REG        = 0.0
keep_prob1    = 0.4
keep_prob2    = 0.4
```

The model was trained using the Adam optimizer. It clearly performed better than the Stocastic Gradient Descent and very often better than RMSProp.

#### Regularization
Adding regularization was one of the first modifications we made to the default LeNet-5 model. A later section describes our iterations on the regularization parameters.
- Dropout. We added dropout layers to all the fully-connected layers in the model. Dropout is not usually used in the convolutional layers and we mostly stuck to this using our iterations. 
- L2-regularization. Again, L2-regularization was added only to the weights in the fully-connected layers. The loss function from the prediction errors was augmented with L2-norm of the layer weights, scaled with a hyperparameter. While L2-regularization have us significant benefits in reducing overfitting, we set it to zero in favour of dropout as our validation accuracy approached closer to the training accuracy.    

#### Development history

Since the number of iterations we large, we present only a few important changes to the model and the hyper-parameters that we took towards our target validation accuracy 

1. The starting LeNet-5 model from the [LeNet MNIST lab](https://github.com/udacity/CarND-LeNet-Lab) gave us an accuracy of around 87%. Changing the number of epochs, learning rates and batch sizes did not a significant increase in accuracy. 
2. Implemented data normalization and conversion to grayscale. This improved the validation accuracy to over 90%. Most of the gains were due to normalization but it was decided to stick with the grayscale conversion anyway. More about this aspect in a later section.
3. The model mostly overfit the training set. Training set accuracy has always been around 98% to 100%. To help solve this, dropout layers were added to all the fully-connected layers. L2 regularization was also added and tuned. This helped the difference to reduce and it improved the validation accuracy to around 92%. 
4. Data augmentation pipeline was implemented. Number of layers and filters were tuned further to try and reduce the overfit with additional nodes. This helped achieve a target and take the validation accuracy to beyond 93%.   
5. The multi-scale LeNet architecture was implemented by connecting the first convolution layer to the fully-connected layer. After tuning the number of filters in each layer and nodes in the classification layers, the final performance numbers were achieved. Removing L2 regularization gave better validation accuracy numbers and hence it was disabled.      

### Model performance

The final model results were as follows:
* Training set accuracy of `100%`
* Validation set accuracy of `96.6%`
* Test set accuracy of `94.1%`

It is not surprising that the accuracy with test set is less than that of the validation set. The model was being continuously tuned with a goal of reducing the validation error. This causes the validation set to indirectly creep into our model training leading to a high accuracy.

Overall, the model still overfits the dataset while surpassing our target on validation accuracy. Later sections address how the performance could be improved. 
 
 
### Test a Model on New Images

##### Choose five German traffic signs found on the web and provide them in the report. For each image, discuss what quality or qualities might be difficult to classify.

Here are five German traffic signs that I found on the web:

![alt text][image4] ![alt text][image5] ![alt text][image6] 
![alt text][image7] ![alt text][image8]

The first image might be difficult to classify because ...

##### Discuss the model's predictions on these new traffic signs and compare the results to predicting on the test set. At a minimum, discuss what the predictions were, the accuracy on these new predictions, and compare the accuracy to the accuracy on the test set (OPTIONAL: Discuss the results in more detail as described in the "Stand Out Suggestions" part of the rubric).

Here are the results of the prediction:

| Image			        |     Prediction	        					| 
|:---------------------:|:---------------------------------------------:| 
| Stop Sign      		| Stop sign   									| 
| U-turn     			| U-turn 										|
| Yield					| Yield											|
| 100 km/h	      		| Bumpy Road					 				|
| Slippery Road			| Slippery Road      							|


The model was able to correctly guess 4 of the 5 traffic signs, which gives an accuracy of 80%. This compares favorably to the accuracy on the test set of ...

##### Describe how certain the model is when predicting on each of the five new images by looking at the softmax probabilities for each prediction. Provide the top 5 softmax probabilities for each image along with the sign type of each probability. (OPTIONAL: as described in the "Stand Out Suggestions" part of the rubric, visualizations can also be provided such as bar charts)

The code for making predictions on my final model is located in the 11th cell of the Ipython notebook.

For the first image, the model is relatively sure that this is a stop sign (probability of 0.6), and the image does contain a stop sign. The top five soft max probabilities were

| Probability         	|     Prediction	        					| 
|:---------------------:|:---------------------------------------------:| 
| .60         			| Stop sign   									| 
| .20     				| U-turn 										|
| .05					| Yield											|
| .04	      			| Bumpy Road					 				|
| .01				    | Slippery Road      							|


For the second image ... 

### (Optional) Visualizing the Neural Network (See Step 4 of the Ipython notebook for more details)
####1. Discuss the visual output of your trained network's feature maps. What characteristics did the neural network use to make classifications?

