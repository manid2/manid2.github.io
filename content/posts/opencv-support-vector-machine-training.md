+++
title = "OpenCV Support Vector Machine training"
description = """Train a Support Vector Machine (SVM) model to recognize \
  facial features using OpenCV (Computer Vision) Cpp API and Histogram of \
  Oriented Gradients (HOG)."""
draft = false
date = "2018-12-30"
author = "Mani Kumar"
categories = ["examples", "tutorials", "computer-vision"]
tags = ["opencv", "svm"]
+++

Introduction
------------

In this tutorial we learn how to train a model of Support Vector Machine (SVM)
on facial image dataset to recognize the facial features such as emotions, age
and gender.

This example is based on the sample [train_HOG.cpp][cv_train_hog] provided in
OpenCV repository to train a SVM model with HOG. This is adopted in
[example_cpp_train_HOG.cpp][ffr_train_hog] to train a SVM model to recognize
the facial features and all the code snippets below are taken directly from
that file.

The first step in training any Machine Learning (ML) model is to collect the
data. For this example we must collect the facial image data. It can be
collected from the web such as google image search or any dedicated public
dataset such as [Flickr Faces][ffhq_dataset].

Organize data
-------------

The face dataset must be organized meaningfully so that it is easy to access
it programmatically. Organize the data into a folder structure that maps
directly to a facial feature and its label.

```text
FFR_dataset/
|-- Age
|   |-- adult
|   |-- child
|   |-- old
|   |-- teen
|-- Emotion
|   |-- anger
|   |-- contempt
|   |-- happy
|   |-- neutral
|   |-- sad
|   |-- surprise
|-- Gender
    |-- female
    |-- male
```

The face dataset directory is self explanatory as the top level directories
map to facial feature e.g. `Age`, `Emotion` etc and its sub-directories map to
a label e.g.  `adult`, `child`, `anger` etc.

A minimum of 50 images in each folder is required to train the models to get
good prediction results. Training more images can improve the results but not
recommended as it takes more time and resources to execute.

After collecting and organizing face image dataset we must pre-process the
images before feeding them to SVM training API.

Read face images into a list of cv::Mat objects
-----------------------------------------------

For each facial feature e.g. `Age`, `Emotion` etc loop through the labels e.g.
`adult`, `child`, `anger` etc and read all the images at the path
`FFR_DATASET_PATH/FEATURE/LABEL`, e.g. for facial feature `Age` an image will
be at `FFR_dataset/Age/adult/img-NNN.jpg`.

```cpp
// CTrainTestHOG::get_ft_dataset ()
cv::String folderName("FFR_dataset/Age/adult/");
std::vector<cv::String> files;
std::vector<cv::Mat> imgs;
cv::glob(folderName, files);

for (size_t i = 0; i < files.size(); ++i) {
	// load the image in grayscale for faster processing
	imgs.push_back(cv::imread(files[i], cv::IMREAD_GRAYSCALE));
}
```

Crop and resize face images to 64x64 (small size)
-------------------------------------------------

Crop images to faces and resize to `64x64 (small size)` to use less memory and
speed up training SVM model.

```cpp
// CTrainTestHOG::get_preprocessed_faces ()
std::vector<cv::Mat> cropped_faces;

for (auto& img : imgs) {
	cv::equalizeHist(img, img);
	std::vector<Rect> faces_r;
	// detect faces using Haar Cascade algorithm
	m_faceCascade.detectMultiScale(img, faces_r);

	for (auto& face : faces_r) {
		cv::Mat cFace = img(face);
		cv::resize(cFace, cFace, cv::Size(64, 64));
		cropped_faces.push_back(cFace);
	}
}
```

Randomly shuffle, split and label the dataset
---------------------------------------------

Shuffle the face images randomly to train the model on random data for better
prediction in real time. Split the dataset into 80% training and 20%
prediction data. Assign the facial feature value as label to each image in
training and prediction data.

```cpp
// CTrainTestHOG::get_ftfv_dataset ()
std::random_shuffle(imgList.begin(), imgList.end());  // optional
int trainPart = imgList.size() * 0.8;  // 80% for training
int predPart = imgList.size() - trainPart;  // 20% for predicting
trainData.reserve(trainPart);
predData.reserve(predPart);
ft_t::iterator ft_iter = m_FeatureList.find(ft);
fv_t::iterator fv_iter = ft_iter->second.find(fv);
int label = std::distance(ft_iter->second.begin(), fv_iter);

int i = 0;
for (; i < trainPart; ++i) {
	trainData.push_back(imgList.at(i));
	trainLabels.push_back(label);
}

for (; i < imgList.size(); ++i) {
	predData.push_back(imgList.at(i));
	predLabels.push_back(label);
}
```

Compute Histogram of Oriented Gradients (HOG) of each face
----------------------------------------------------------

Compute Histogram of Oriented Gradients (HOG) for each face image in training
data with `8 * 8` window size. Store the computed HOG for each face image in a
list of `cv::HOGDescriptor` objects.

```cpp
// CTrainTestHOG::computeHOGs ()
cv::HOGDescriptor hog;
std::vector<cv::Mat> hogMats;
std::vector<float> descriptors;

for (auto& img : imgHogList) {
	hog.winSize = img.size() / 8 * 8;
	hog.compute(img, descriptors);
	cv::Mat descriptors_mat(Mat(descriptors).clone());
	hogMats.push_back(descriptors_mat);
}
```

Convert the list of HOG into cv::Mat
------------------------------------

Convert the list of computed HOG descriptor matrices into a single OpenCV
`cv::Mat` object suitable as input to train the SVM model.

```cpp
// CTrainTestHOG::convert_to_ml()
const int rows = (int) train_samples.size();
const int cols = (int) std::max(train_samples[0].cols,
			        train_samples[0].rows);

cv::Mat tmp(1, cols, CV_32FC1);  //< used for transposition if needed
cv::Mat trainData = cv::Mat(rows, cols, CV_32FC1);

for (size_t i = 0; i < train_samples.size(); ++i) {
	CV_Assert(train_samples[i].cols == 1 || train_samples[i].rows == 1);
	if (train_samples[i].cols == 1) {
		cv::transpose(train_samples[i], tmp);
		tmp.copyTo(trainData.row((int)i));
	}
	else if (train_samples[i].rows == 1) {
		train_samples[i].copyTo(trainData.row((int)i));
	}
}
```

Train and save the SVM model
----------------------------

Use the `cv::Mat` object converted from HOG descriptors list as input to SVM
train model API and save the trained model with file name containing the
facial feature label i.e. `Age`, `Emotion` etc.

```cpp
// CTrainTestHOG::Run()
trainLabels.resize(ml_train_data.rows);
m_pSVM->train(ml_train_data, ROW_SAMPLE, trainLabels);
cv::String svmModelFileName = cv::format("%s/cv4_svm_%s_model.xml",
                                         getenv(FFR_DATASET_PATH),
                                         ft.first.c_str());

m_pSVM->save(svmModelFileName.c_str());
```

Load trained SVM model and predict facial features
--------------------------------------------------

To verify the correctness and accuracy of the trained SVM model we take the
prediction dataset we had reserved earlier and compute HOG descriptor for each
face image in the prediction dataset, convert the list of HOG descriptors into
a single `cv::Mat` object to pass this to SVM predict API.

```cpp
// CTrainTestHOG::Run()
this->computeHOGs(predData);

// convert HOG feature vectors to SVM data
cv::Mat ml_pred_data;
std::vector<int> resultLabels;
this->convert_to_ml(predData, ml_pred_data);
predLabels.resize(ml_pred_data.rows);

// Get result labels from SVM predict OpenCV API
cv::Mat responses_mat;
m_pSVM->predict(ml_pred_data, responses_mat);
for (size_t i = 0; i < ml_pred_data.rows; ++i) {
	resultLabels.push_back(responses_mat.at<int>(i));
}
```

Calculate prediction accuracy of the model
------------------------------------------

Calculate the percentage of prediction accuracy by comparing the expected
prediction labels with the result labels.

```cpp
// CTrainTestHOG::get_prediction_accuracy ()
// pred labels and result labels must be of same size
int misMatchCount = 0;
for (size_t i = 0; i < rLables.size(); ++i) {
	if (rLables.at(i) != pLables.at(i))
		misMatchCount++;
}
int correct = rLables.size() - misMatchCount;
accuracy = correct * 100.0f / rLables.size();
```

Run for N times and get mean prediction accuracy
------------------------------------------------

Run the above steps for N run times and calculate the mean accuracy to get
consistent prediction accuracy each time the model is used to predict the
facial feature on a face image.

```cpp
// CTrainTestHOG::Run()
float sum_of_accuracies = std::accumulate(
	predictionAccuracyList.begin(),
	predictionAccuracyList.end(), 0.0);
float mean_accuracy = sum_of_accuracies / predictionAccuracyList.size();
```

[ffhq_dataset]: https://github.com/NVlabs/ffhq-dataset
[cv_train_hog]: https://github.com/opencv/opencv/blob/4.x/samples/cpp/train_HOG.cpp
[ffr_train_hog]: https://github.com/manid2/FacialFeaturesRecognizer/blob/master/util/example_cpp_train_HOG.cpp
