# Medical Image Classification with Convolutional Neural Networks: Two Functional Models (Pretrained and Custom) Trained and Evaluated Individually, as an Ensemble, and Chained


## Executive Summary

Can we train a custom convolutional neural network (CNN) model from scratch to classify CT chest scan images as indicating one of the following four categories (classes): Adenocarcinoma, Large cell carcinoma, Squamous cell carcinoma, or normal cells? How well does the model perform?
  
What happens if we add to our model a pre-trained CNN model by employing transfer learning and model ensembling? Will we see improved accuracy scores with either of these methods?

We defined, compiled, and trained two CNN submodels - one custom and one pre-trained - individually before ensembling and chaining them. We looked for a noticeable improvement in accuracy between the ensembled model and/or the chained model, over each of the two submodels.  
  
 a) the pre-trained ResNet50 model (model_one)   
 b) our custom CNN (model_two)  
 c) ensembling the output of model_one and model_two (ensemble_model)  
 d) chaining model_one and model_two into model_four (transfer learning)  

Concepts discussed:  
Convolutional Neural Networks  
Pretained models  
Model Ensembling  
Transfer Learning/model chaining  
  
The data for this project was obtained here: https://www.kaggle.com/datasets/mohamedhanyyy/chest-ctscan-images  

The notebook for this project is Ensemble_Chain_ResNet_custom_manual_sparse.ipynb.
  
    
## Convolutional Neural Networks  
  
CNNs use convolutional and pooling layers to automatically and hierarchically learn features from images, and use fully connected layers to classify those features into predefined categories. This process enables CNNs to effectively handle and classify complex visual data.  
    
We built our custom CNN (model_two) and our ResNet50-based pre-trained model with the following components:  
  
1. Input Layer: Our input images are represented as matrices of pixel values.   

2. Convolutional Layers: These layers applied convolutional filters (or kernels) to the inputs. Each filter scanned the images and performed a convolution operation involving element-wise multiplication and results summing. These layers extracted features like edges, textures, and patterns from each image and produced a feature map highlighting the presence of specific features in different parts of the image. 
  
3. Activation Function: We applied activation function ReLU (Rectified Linear Unit) to introduce non-linearity into the model, which helped the network learn more complex patterns.  
  
4. Pooling Layers: We used max pooling to reduce the spatial dimensions of the feature maps by taking the maximum value from a subset of the feature map. This reduced the number of parameters and computations, helping the network become more robust to variations in image.  
  
5. Flattening (model_two only): Because the output from the convolutional and pooling layers was a multi-dimensional tensor, we needed to flatten the tensor to a one-dimensional vector before feeding it into the fully connected layers.    
  
6. Fully Connected Layers: Similar to traditional neural networks, where each neuron is connected to every neuron in the previous layer, the fully connected layers combined the features learned by the convolutional and pooling layers to make a final prediction.  
  
7. Output Layer: We chose a softmax function capable of outputting probabilities for each of the four classes, indicating the network's prediction of Adenocarcinoma, Large cell carcinoma, Squamous cell carcinoma, or normal cells.   
    
 
## Pretrained Models

Pre-training a neural network involves training a model on a large, broad, general-purpose dataset before fine-tuning it on a specific task (a new set of specific, likely previously unseen data). The ResNet50 model is a well-known model that was trained on the ImageNet database, a collection of millions of images classified across thousands of categories.   

During pre-training, the model learns to identify and extract general features from the input data, such as images' edges, textures, and shapes. These features become broadly useful across new tasks and data domains, even if the new data was never part of the training data.

The benefits of pre-training include improved performance, better generalization, and reduced training time. Pre-training allows the model to leverage knowledge learned from a large and diverse dataset. This accumulated knowledge can lead to better performance on the new task, especially when the new dataset is small or lacks diversity. Training a model from scratch can be computationally expensive and time-consuming. Pre-training on a large dataset and then fine-tuning it can significantly reduce the time required to achieve good performance.

Pre-trained models often generalize better to new tasks because they start with a solid understanding of basic features and patterns, which can help improve accuracy on the new task. Pre-training can be a powerful technique, especially when data are scarce or where training a model from scratch would be impractical given resource constraints.  

  
## Submodel Compatibility Considerations
  
To ensure our custom CNN and the pre-trained CNN would be compatible with each other for direct ensembling and transfer learning puposes, we took the following precautions and made adaptations to the original ResNet50 model. Note we refer to our ResNet50-based model as first_model.   
  
1. Used the tf.keras.preprocessing.image_dataset_from_directory method to generate our training_set, testing_set, and validation_set.  
   * This method automatically labeled all images based on their subdirectory names. 
   * It also treated each each subdirectory as a class, assigning labels as integers starting from 0.  
    
2. Specified image_size as (224, 224) for first_model because ResNet50-based models expect images of that size. For purposes of consistency, we set the image_size to (224, 224) for second_model as well.  
     
3. Specified channels, img_shape, and class_count in first_model to be identical to those in second_model
   
4. Defined the same data augmentation layers in both submodels and applied data augmentation to the input tensor
  
5. Defined the same rescaling layers in both submodels, and specified the input tensor as the scaled inputs
   
6. Applied data augmentation and rescaling in both submodels, early in the model pipeline
    * When ensembling two models, it is appropriate to apply data augmentation and rescaling in both submodels.
    * In particular, data augmentation should come before rescaling, right after defining the model's input layer.
    * Data augmentation techniques (e.g., RandomRotation, RandomZoom, RandomFlip) are designed to work on raw pixel values in the 0-255 range.
    * If rescaling is done first, pixel values are converted to 0-1, which could interfere with how certain augmentations are applied.
    * Because the ResNet50-based base_model component of our first_model expected inputs' pixel values to be normalized to a range between 0 and 1, we rescaled the augmented input data before passing it to the ResNet50 layers. 
  
7. Specified include_top = False to effectively removed ResNet50's top layer so we could replace it with one suited for our own task.  Because the original ResNet50 model was pretrained to classify over a million ImageNet images into 1,000 classes, it outputs feature maps when its top layer is removed. 

8. Added a pooling='max' layer to the ResNet50-based model to control the shape of the output tensor and ensure compatibility with the subsequent layers that needed to be added.  
   * The ResNet50 model outputs a 4D tensor after its convolutional layers when include_top=False and no pooling is applied. This tensor could not be fed directly into fully connected Dense layers, which require a 2D input.  
   * Setting pooling='max' applied global max pooling to reduce the spatial dimensions into a single value for each channel, producing a tensor compatible with Dense layers.  
   * Without pooling='max', we would have needed to explicitly add a Flatten layer to convert the 4D tensor to 2D in order to avoid a shape mismatch error. Though a Flatten layer would have resolved the shape issue, it would generate a larger input size for the Dense layers, increasing the risk of overfitting.  
   * Unlike Flattening, which preserves all spatial information to return a high-dimensional feature vector, global pooling reduces dimensionality.   
  
9. Specified for layer in base_model.layers: layer.trainable = False, to avoid re-training ResNet50's pre-trained knowledge during model training.  
   * Making these layers untrainable preserved the features ResNet50 learned during pre-training, keeping them from becoming over-written during training.  
   * Layer freezing effectively turned ResNet50 into a feature extractor.  
    
10. Built both submodels with the Functional API because it supports more flexibility than the Sequential API. In particular, the Functional API
    * Affords more flexibility when combining pre-trained models with custom layers or sharing layers between models 
    * Allows for explicit definition of the flow of data, enabl fine control over how layers connect and interact  
    * Supports freezing layers and chaining models  
    * Handles the complexities involved in ensembling models  
     
11. Added custom layers on top of the ResNet50-based base to allow the final model to complete our four-class classification task and to be ensembled and chained with the other submodel.
    * Both the BatchNormalization and Dropout layers helped improve generalization on unseen data.  
    * The Dense(256, activation='relu') layer learned more complex patterns from the high-level features provided by ResNet50
    * These more complex patterns became relevant to our classification task
    * The relu activation function supported the custom layers to model more intricate relationships between features
    * Dropout(0.25) was intended to prevent overfitting by forcing the model to learn more robust features and preventing it from becoming too reliant on specific neurons 
  
12. Defined identical output layers in each submodel:
    * Dense(class_count, activation = 'softmax') to output a probability distribution across the classes (given by class_count).
    * Each value in the probability distribution corresponded to the predicted probability that the input image belonged to a given class
    * We chose the Softmax activation because it can return a probability distribution over three or more classes
    
14. Compiled both submodels with optimizer='adam', loss='sparse_categorical_crossentropy', and metrics=['accuracy']. We chose the 'sparse_categorical_crossentropy' function because our dataset includes integer labels and it works for most multi-classification tasks.  

15. Trained both submodels with identical EarlyStopping and ModelCheckpoint callbacks. 
  
  
### Model Ensembling  
  
Ensembling models entails combining the individual predictions of multiple models on the same dataset, in an attempt to make better predictions on that dataset. Ensemble models can improve upon the predictive performance of individual models. If different models make different types of errors, we may be able to reduce the overall error rate by combining their predictions. 

In this project, we combined our two submodels' predictions in model_three, an ensemble model that averaged the submodels' output. Here, each model contributing to model_three wass weighted equally in the ensemmble model. It is possible to configure a weighted average ensemble in which better-performing submodels contribute more to the ensemble than poorer-performing submodels. 

There are additional techniques for combining submodel predictions. In bootstrap aggregating, multiple models are trained on different subsets of the same training data and then ensembled. Boosting models occurs when models are trained sequentially, allowing later models to correct the errors made by earlier models. The voting technique makes a final prediction by taking a majority vote of the predictions made by the various submodels. 

Ensemble models can yield improved accuracy over their individual submodels by reducing overfitting. They may exhibit more robustness to changes in input data than their submodels. On the other hand, ensemble models can entail increased complexity, reduced ease of interpretability, and greater computational costs than their submodels individually.    
  
  
We took the following steps to prepare for and build the ensemble_mode:

1. Defined the full file paths to our best saved first_model and second_model, and loaded them from saved. keras.
   
2. Extracted labels from the TensorFlow datasets (training_set, testing_set, validation_set) we had created by using the tf.keras.preprocessing.image_dataset_from_directory method. Ensemble models need labels in order to compute loss (by comparing predictions to true lables) and update models during training. 
  
3. Generated submodel predictions for the training and validation datasets with shape (None, 4):  
    * preds_first_model_train = first_model.predict(training_set)
    * preds_second_model_train = second_model.predict(training_set)
    * preds_first_model_val = first_model.predict(validation_set)
    * preds_second_model_val = second_model.predict(validation_set)
  
4. Defined EarlyStopping and ModelCheckpoint callbacks and a filepath to save the best ensemble_model. 
  
5. Built, compiled, and trained the ensemble_model, in the same manner as with the two submodels, to process the combined predictions.   
    * Defined ensemble_input_train as the simple average of the first_model's predictions on the training_set and the second_model's predictions on the training_set.
    * Defined ensemble_input_val as the simple average of the first_model's predictions on the validation_set and the second_model's predictions on the validation set.
    * Defined ensemble_input as the input layer for the ensemble_model, with Input(shape=(4,)) because
        * The submodels output predictions of shape (None, 4), which ensemble_model takes as inputs  
        * The ensemble_model does not take the image datasets fed to the submodels
    *  Added a Dense layer as the output layer, final_output = Dense(4, activation='softmax')(ensemble_input)
    * Defined ensemble_model as Model(inputs=ensemble_input, outputs = final_output)
    * Compiled ensemble_model with optimizer='adam', loss='sparse_categorical_crossentropy', metrics=['accuracy']
    * To train the ensemble_model, we used ensemble_input_train as our x and y_train - the true labels from our training_set as our true labels corresponding with the averaged training_set predictions.
    
      
## Transfer Learning   

An alternative to ensembling two models is chaining them. After pre-training, a model can be applied to a new, specific dataset and classification task, in a process called transfer learning. The pre-trained model's weights, optimized during pre-training, become the starting point for training on a new, often smaller, dataset. The model learns the specifics of the new task while leveraging the general features it learned during pre-training. In our project, the smaller dataset consisted of the CT-Scan images with different types of chest cancer versus normal cells. 
  
  
# Chaining Models  
The considerations we had to take into account when chaining model_one and model_two included:  
   
1. We removed the final, dense layer from first_model because we didn't want it to generate a vector representing class probabilities. We wanted to use first_model only as a feature extractor, allowing second_model to generate the class probabilities.  
    * We defined a new output layer to serve as feature extraction, mod_first_model_output = x  
    * Thus, the final output of our modified first_model became the result of the Dropout layer, the layer immediately preceding the final output layer.  
    * We saved this modified version of first_model as mod_first_model  
      
2. Data augmentation and rescaling was necessary only in the first of two chained submodels, when using the Functional API. We removed augmentation and rescaling layers from second_model, which we saved and renamed mod_second_model.  
   
3. We removed the dropout, convolutional, and pooling layers present in second_model from mod_second_model because such layers are already present in the ResNet50 base of mod_first_model. Not removing these custom layers risked: 
      
   * Over-Parameterization, which can result in a model with too many parameters. Over-Parameterized models  
       -    Have an increased risk of overfitting, especially on small datasets
       -    May require more computational resources
       -    Can make training unstable  
  
   * Redundant Feature Extraction, which can cause computational inefficiency and
possible degradation of learned features, as custom layers "over-process" the features

   * Loss of Transfer Learning Benefits, if custom layers on top of ResNet50 disrupt the transfer learning process
       
         * If the transfer learning process is disrupted, training essentially begins again from scrath and the benefit of pretrained weights is lost  
         * Custom layers may not complement the ResNet50-extracted features, reducing model effectiveness  
         * Custom layers could undermine the pre-trained model's ability to generalize

   * Training Instability, which can occur when excessive layers make the model architecture deeper and more complex than necessary  
  
   * Increased Risk of Overfitting, resulting in the model memorizing the training data instead of learning generalizable patters  
  
5. We removed the Flatten layer present in second_model, as it was no longer necessary.   
  
6. We defined one Dense and one Dropout layer before defining the output layer, designed to produce a four-class classification
    
7. We defined but din't compile and train mod_first_model and mod_second_model, because we would chain them into chained_model





 Ensembling Models

After training first_model and second_model, we created a third model, ensemble_model, to average the first two models' output. The submodels output predictions of shape (None, 4). The ensemble_model takes these outputs as inputs. We do not feed the same image datasets we fed to the submodels to the ensemble_model. Only the predictions (outputs) from first_model and second_model are inputs to ensemble_model.


### Extracting dataset labels

Unlike the original input datasets, which contained images and class labels, the inputs to the ensemble_model lacks labels. Thus, we needed to extract the labels from the TensorFlow datasets and give them to ensemble_model. These labels are necessary for the ensemble_model to compute loss values, which entails comparing predictions to true labels. 

<img width="921" alt="Screenshot 15" src="https://github.com/user-attachments/assets/349ac984-af66-4000-92e4-8acc58df8fd1">
<img width="932" alt="Screenshot 16" src="https://github.com/user-attachments/assets/43efe309-5737-485f-96d4-0992ab5cf365">


### Preparing data and building ensemble model to average outputs

Before we could build the ensemble_model to process the two submodels's output (predictions), we needed to generate predictions from first_model and second_model using the training_set and validation_set. We had used both training_set and validation_set to train each submodel, so we needed predictions from these same models on the same datasets to be inputs to the ensemble_model

Next, we defined the EarlyStopping and ModelCheckpoint callbacks to be used to train ensemble_model. We kept these callback definitions consistent with those used in the two submodels. If the accuracy on the validation dataset did not improve after 20 epochs, the model training would come to an early stop rather than continue on for 100 epochs. Similarly, we defined a filepath to save the best version of the model (that with the maximum validation accuracy). 


<img width="912" alt="Screenshot 17" src="https://github.com/user-attachments/assets/d97b10b3-7c94-4552-833b-b12ca1dc4789">
<img width="907" alt="Screenshot 18" src="https://github.com/user-attachments/assets/9a4d3a77-3bc1-46f4-91d3-c20df954c988">


We defined ensemble_model to average the training_set and validation_set predictions made by first_model and second_model. Keras performs this averaging element-wise across the class probabilities for each sample. Though the submodels' ouput were of shape (None, 4), with None representing variable batch size and 4 representing class probabilities for each image in the batch, Keras implicitly understood that each sample in each batch had a shape of (4,). This was equivalent to the shape ensemble_model expected for its inputs, tensors representing the 4 class probabilities for each sample. We didn't need to reshape any outputs explicitly. The implicit reshaping made it possible to combine the submodel outputs on a sample-by-sample basis rather than processing whole batches of predictions at once. We simply had to specify the shape of each sample with ensemble_input = Input(shape=(4,)).


<img width="913" alt="Screenshot 19" src="https://github.com/user-attachments/assets/ce1519c3-a742-4a86-8092-b41b80949a93">
<img width="907" alt="Screenshot 20" src="https://github.com/user-attachments/assets/99d07b95-cb92-4db1-9591-4dfa66fe95c4">
<img width="914" alt="Screenshot 21" src="https://github.com/user-attachments/assets/9121fc34-0c2a-46dc-8223-34c25be54238">
<img width="913" alt="Screenshot 22" src="https://github.com/user-attachments/assets/5ccd775e-92c9-40eb-be30-30db71dbee08">


We compiled and trained ensemble_model in the same manner as we did its two submodels. Here, our x value became the averaged training_set predictions from the first and second model, while our y values became the true labels corresponding to the averaged training predictions. Finally, the validation_data for the ensemble model became the averaged predictions from the two submodels on the validation dataset and the true labels for the validation dataset itself. 


## Chaining Models

Chaining two models together means creating a composite model, where the first model's output becomes the input for the second model's layers. In this scenario, there is no third model to process the outputs of the two submodels. The output of the first model in a two-model chain is not a classification, but features that will help the second model's layers make a classification. Chaining two models results in a single model that can be trained end-to-end.

Model chaining can be performed using the Functional API in a Keras framework, which allows for flexible connection of layers and models. Unlike in the ensembling, where data augmentation and rescaling can be applied in both submodels, chaining two models requires specifying data augmentation and rescaling only in the first model's layers.


## Modifying first_model from classifier to feature extractor

Because we are turning first_model's output into second_model's input, some adjustments to the original versions of these models became necessary. In particular, first_model needed to be redefined from a classifier to a feature extractor. That is, first_model became tasked with processing raw input data (the ct scans) and producing informative features to be used for classification in second_model's layers. Pretrained models like ResNet50 are often used as feature extractors in transfer learning because they have already learned useful patterns from the large datasets on which they were trained. These patterns, or features, are reusable for new tasks. To diffferentiate first_model and it's modified version, we called our modified first_model 'mod_resnet_model'.

<img width="902" alt="Screenshot 23" src="https://github.com/user-attachments/assets/d4cefbbe-8a3f-4922-a52e-db8b0c760121">
<img width="896" alt="Screenshot 24" src="https://github.com/user-attachments/assets/dd23844d-69f2-4ebc-921b-fe6c621f5a55">
<img width="883" alt="Screenshot 25" src="https://github.com/user-attachments/assets/70de0f23-1638-4e77-a7c1-31d83c48cf2a">


## Modifying second_model to be compatibile for chaining

In order to chain second_model with mod_resnet_model, we needed to omit the data augmentation and rescaling layers. We also needed to remove a number of second_model layers that became redundant when chained with mod_resnet_model. We named this altered version of second_model 'mod_custom_cnn_model' to keep the two distinct. 

The layers we dropped from second_model to create mod_custom_cnn_model were the MaxPooling, Conv2D, and Dropout layers. The first_model layers, based on the ResNet50 model, already performed these operations in the first part of the chain. Reapplying these layers, by including them in second_model's layers, would have been redundant and not necessarily improved performance. The point of chaining first_model and second_model was to use ResNet50 for feature extraction and then use second_model to perform further processing on the extracted features for classification purposes.

Because ResNet50 already included downsampling layers, adding additional pooling and convolution operations with second_model could have resulted in too much downsampling or feature over-processing.

It was appropriate to still include data augmentation and rescaling before the ResNet50 layers since these operations were not part of the feature extraction operations. These pre-processing layers simply prepared the input data for feature extraction.


<img width="901" alt="Screenshot 26" src="https://github.com/user-attachments/assets/3273d338-f2fc-4b9b-9045-2c29fc8df042">
<img width="902" alt="Screenshot 27" src="https://github.com/user-attachments/assets/3f22b4f9-ec67-4976-9ad2-37f58f3a97a7">


## Defining, compiling, and training the chained model

We created the chained model by chaining the modified ResNet50-based model, 'mod_resnet_model', with the modified base cnn model, 'mod_custom_cnn_model'. The two individual models were, themselves, variations of first_model and second_model that made them compatible for chaining. 

By defining mod_resnet_output as mod_resnet_model.output, we specified mod_resnet_model's layers as the first 'link' in the chain. By specifying mod_custom_cnn_output = mod_custom_cnn_model(mod_resnet_output), we passed the first 'link's' output to the second 'link' in the chain, mod_custom_cnn_model, and defined the resulting output as mod_custom_cnn_output. This allowed us to define the composite model, chained_model, as Model(inputs=mod_resnet_model.input, outputs=mod_custom_cnn_output). Before training chained_model, we specified optimizer = Adam(), defined a filepath to save chained_model's best model, and defined equivalent EarlyStopping and ModelCheckpoint callbacks as we'd used previously. We trained chained_model on the dataset training_set and set validation_set as the validation_data.   


<img width="889" alt="Screenshot 28" src="https://github.com/user-attachments/assets/4a27f82a-462b-4287-af38-e2984f576bf3">
<img width="913" alt="Screenshot 29" src="https://github.com/user-attachments/assets/78e3e396-7543-421a-a851-cb66eb0a264f">
<img width="886" alt="Screenshot 30" src="https://github.com/user-attachments/assets/6c06d877-6cad-420b-9d57-806e501c9b28">


## Evaluating all four models
When it came to evaluating all four models, first_model and second_model (the submodels) needed to be evaluated on the unseen testing_set dataset to get unbiased performance metrics. 

With first_model, second_model, and chained_model already trained on the training_set and validated on the validation_set, and the best versions of these models saved, we evaluated these three models on the testing_data with the following statements:

first_model_loss, first_model_accuracy = first_model.evaluate(testing_set)
print(f"First model - Loss: {first_model_loss}, Accuracy: {first_model_accuracy}")

second_model_loss, second_model_accuracy = second_model.evaluate(testing_set)
print(f"Second model - Loss: {second_model_loss}, Accuracy: {second_model_accuracy}")

chained_model_loss, chained_model_accuracy = first_model.evaluate(testing_set)
print(f"Chained model - Loss: {chained_model_loss}, Accuracy: {chained_model_accuracy}")

Evaluating the ensemble model was a matter of 
a) averaging the predictions from the two submodels models on the unseen testing_set,  
b) extracting the labels from the testing_set, and   
c) estimating ensemble loss and ensemble accuracy by requesting ensemble_model.evaluate(ensemble_predictions, y_test)


<img width="897" alt="Screenshot 31" src="https://github.com/user-attachments/assets/671b15de-1eb8-48b2-b493-ed34f330a754">
<img width="902" alt="Screenshot 32" src="https://github.com/user-attachments/assets/6fd3212f-6ce8-4f7c-b133-d82cfdb1e2de">


## Table of results

| model          |   train_loss  | train_accuracy |   val_loss   | val_accuracy |   test_loss   |  test_accuracy  |
|----------------|---------------|----------------|--------------|--------------|---------------|-----------------|
| first_model    | 0.6271 | 0.7406 | 1.009 | 0.6250 | 1.0288 | 0.5301 | 
| second_model   | 0.4488 | 0.8189 | 0.6815 |  0.8194 | 2.9589 | 0.3746 |
| ensemble_model | 1.4364 | 0.2120 | 1.4026 | 0.2777 | 1.4087 |  0.2444 |
| chained_model  | 0.8707 | 0.6215 | 0.9116 | 0.5833 | 1.0094 |  0.5142 |

We noted some unexpected results when combining the two models. Neither the ensemble_model nor the chained_model outperformed the first_model (the ResNet50-based classifier). The accuracy results for the ensemble_model were especially low, when compared with the two submodels.

It is unusual for an ensemble model that combines its submodels' output to have lower accuracy than its individual submodels. Such results can indicate that there's an issue with prediction averaging, if the models' outputs are raw logits or probabilities. Because the two classification models are generating averageable probabilities, however, averaging errors are not at play.

Likewise, we can rule out the possibility that the model was trained using pseudo-lables rather than true labels, since we explicitly specified the relevant true lables as the validation_data. Pseudo-labeling would have occured if we trained the ensemble model on the submodel predictions as labels, instead of using the true labels.

Finally, a problem with the evaluation methodology doesn't explain the lower accuracy scores for the ensemble_model. The evaluation of the ensemble model wass consistent with how the model was trained (e.g., on averaged predictions). 

It could be that the two submodels are underperforming or have biases. If this is the case, averaging the two submodels' predictions would not necessarily improve performance. It's also possible that averaging the submodels' predictions is exacerbating weaknesses in the two models if the models are making similar errors. 

Overfitting or underfitting could also be a factor. Averaging the predictions of models that overfit the training data could result in poor generalization to unseen data. Averaging predictions could also be problematic if the submodels are underfitting the training data because the averages could be failing to capture complex patterns. 

Alternatively, it might be the case that simple averaging is not appropriate when ensembling our two models. Averaging predictions when one model submodel is significantly better than the other can dilute the effectiveness of the stronger model. Using weighted averaging instead of simple averaging might be called for in this case, as first_model has significantly better accuracy scores than second_model. A possible next step could be ensembling first_model and second_model with weighted averages. 











## Executive Summary   
  
In this document, we trained a convolutional neural network (CNN) model from scratch to classify CT chest scan images as either cancerous or normal, and then investigated the added benefit of using a pre-trained CNN model on the same dataset by employing transfer learning and model ensembling.

The images to be classified were CT chest images from the dataset found at   https://www.kaggle.com/datasets/mohamedhanyyy/chest-ctscan-images. Each image revealed one of the following four outcomes: adenocarcinoma, large cell carcinoma, squamous cell carcinoma, or healthy cells.

| model | training loss | training accuracy | validation loss | validation accuracy |
|-------|------|----------|----------|--------------|
| first_model  | 0.6271  | 0.7406  | 1.009 | 0.6250 |
| second_model  | 0.4488 | 0.8189 | 0.6815 | 0.8194 |
| ensemble_model | 1.4364 | 0.2120 | 1.4026 | 0.277 |
| chained_model | 0.8707 | 0.6215 | 0.9116 | 0.5833 |

## Project Overview

The data for this project was obtained at: https://www.kaggle.com/datasets/mohamedhanyyy/chest-ctscan-images. Images reveal the presence of one of three chest carcinomas or the presence of healthy cells. The models' tasks are thus to classify each image as an one of the following four outcomes: adenocarcinoma, large cell carcinoma, squamous cell carcinoma, or healthy cells. All images were divided into training (613), testing (315), and validation (72) sets containing four types of images. 

We trained a convolutional neural network (CNN) model from scratch to classify CT chest scan images as either cancerous or normal, and then investigated the added benefit of using a pre-trained CNN model on the same dataset by employing transfer learning and model ensembling.

CNNs use convolutional and pooling layers to automatically and hierarchically learn features from images, followed by fully connected layers to classify those features into predefined categories. This process enables CNNs to effectively handle and classify complex visual data.  

Pre-training a model in the context of neural networks involves training a model on a large dataset before fine-tuning it on a specific task, with the end result known as transfer learning. Pre-training is a powerful technique, especially in scenarios where data are scarce or where training a model from scratch would be impractical due to resource constraints. In this project, we used the ResNet50 CNN, trained to classifiy 14 million images in the ImageNet dataset into 1,000 different categories, as our pre-trained model. We applied this model, by itself, and in combination with our own CNN model, to the chest images for classification. 

Because we were working with four distinct image lables classes (adenocarcinoma, large cell carcinoma, squamous cell carcinoma, and healthy cells), we chose the sparse categorical crossentropy loss function. We anticipated lables as integers because we used the tf.keras.preprocessing.image_dataset_from_directory' function from TensorFlow’s Keras API to load image data from the train, test, and validate directories, which automatically labels images based on directory structure.






<img width="914" alt="Screenshot 00a" src="https://github.com/user-attachments/assets/1c553bd2-3721-4a88-a616-a1523fc68a76">
<img width="915" alt="Screenshot 00b" src="https://github.com/user-attachments/assets/a056c7aa-509f-4825-8a81-d6ed52e130a4">
## ensemble_model, combining the predictions of first_model and second-model
