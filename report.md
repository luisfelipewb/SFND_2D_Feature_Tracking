# Project: 2D Feature Tracking

[//]: # (Image References)
[gif_intro]: ./media/intro.gif
[img_detector_table]: ./media/detector_table.png
[img_matches_num_table]: ./media/number_of_matches_table.png
[img_detector_plot]: ./media/detector_plot.png
[img_combination_table]: ./media/combination_table.png
[img_AKAZE_detector]: ./media/AKAZE_DetectorResults.png
[img_BRISK_detector]: ./media/BRISK_DetectorResults.png
[img_FAST_detector]: ./media/FAST_DetectorResults.png
[img_HARRIS_detector]: ./media/HARRIS_DetectorResults.png
[img_ORB_detector]: ./media/ORB_DetectorResults.png
[img_SHITOMASI_detector]: ./media/SHITOMASI_DetectorResults.png
[img_SIFT_detector]: ./media/SIFT_DetectorResults.png
[img_fast_and_brief]: ./media/Kpts_DetFAST_DescBRIEF.png
[img_fast_and_orb]: ./media/Kpts_DetFAST_DescORB.png
[img_brisk_and_orb]: ./media/Kpts_DetBRISK_DescORB.png


## Mid-Term Report

This report summarizes the work done for the 2D Feature Tracking Project in the Sensor Fusion Nanodegree from Udacity.
The main goal is to compare performance of different keypoint detectors and track those points across successive images.
Instead of providing a comprehensive explanation, this document is organized according to the rubric points required for the project.

![][gif_intro]

### MP.0 Mid-Term Report

> Provide a Writeup / README that includes all the rubric points and how you addressed each one. You can submit your writeup as markdown or pdf.

Each section of this document addresses the corresponding rubric points.

## Data Buffer

### MP.1 Data Buffer Optimization

> Implement a vector for dataBuffer objects whose size does not exceed a limit (e.g. 2 elements). This can be achieved by pushing in new elements on one end and removing elements on the other end.

The Data Buffer was implemented using the `deque` (double-ended queue) data structure available in the std library.
According to the reference documentation "they provide a functionality similar to vectors, but with efficient insertion and deletion of elements also at the beginning of the sequence, and not only at its end."

Basically, `pop_front()` function is used to remove the `DataFrame` from the beginning of the buffer when the maximum size has been reached.

The implementation is quite simple and can be seen in the following commit: [MP.1 Data Buffer Optimization](https://github.com/luisfelipewb/SFND_2D_Feature_Tracking/commit/1031a648622623aa4b0b5b146c02b3328ee2f0cd)


## Keypoints

### MP.2 Keypoint Detection

> Implement detectors HARRIS, FAST, BRISK, ORB, AKAZE, and SIFT and make them selectable by setting a string accordingly.

The functions `detKeypointsHarris` and `detKeypointsModern` were implemented in the file `matching2D_Student.cpp` according to the signatures already available in the `matching2D.hpp` header.
The selection of the detector is done via a `string` as requested and the code can be seen in the following commit: [MP.2 Keypoint Detection ](https://github.com/luisfelipewb/SFND_2D_Feature_Tracking/commit/161f33cc5c4890211aabfd5b1189d9b2ff5a27d3)


Naturally, this implementation could be improved by using a `enum` type. Beyond making it more robust against typing mistakes, it provides a clear overview of the available detectors for the developer.

Additionally, the function `visualizeKeypoints` was implemented since it can be reused from different parts.
```cpp
void visualizeKeypoints(const cv::Mat& img, const std::vector<cv::KeyPoint>& keypoints, const std::string& windowName)
{
        cv::Mat output;
        cv::drawKeypoints(img, keypoints, output, cv::Scalar::all(-1), cv::DrawMatchesFlags::DRAW_RICH_KEYPOINTS);
        cv::imshow(windowName, output);
        cv::waitKey(0);
}
```


### MP.3 Keypoint Removal

> Remove all keypoints outside of a pre-defined rectangle and only use the keypoints within the rectangle for further processing.

The keypoints outside of a pre-defined rectangle are removed using the `contains()` function available int the `cv::Rect` type.

The implementation can be seen bellow:

```cpp
cv::Rect vehicleRect(535, 180, 180, 150);
if (bFocusOnVehicle)
{
    vector<cv::KeyPoint> filtered;
    for (auto it = keypoints.begin(); it != keypoints.end(); it++)
    {
        if(vehicleRect.contains(it->pt))
        {
            filtered.push_back(*it);
        }
    }
    cout << "Reduced from " << keypoints.size() << " to " << filtered.size() << " keypoints" << endl;
    keypoints = filtered;
}
```

Another approach would be to crop or mask the image prior to the keypoint detection. This approach would more efficient than computing all keypoints and discarding those outside the region of interest (ROI).


## Descriptors

### MP.4 Keypoint Descriptors

> Implement descriptors BRIEF, ORB, FREAK, AKAZE and SIFT and make them selectable by setting a string accordingly.

The descriptors are implemented in the `matching2D_Student.cpp` file, within the `descKeypoints` function.
The complete implementation can be inspected in the following commit: [Implement additional Descriptors extraction](https://github.com/luisfelipewb/SFND_2D_Feature_Tracking/commit/ea935aa93cfc90253479c538c2c51e4422381139)

In all cases, the extractor object was created using the default parameters.

As mentioned before, an `enum` type could be used for selection instead of a `string`.

### MP.5 Descriptor Matching

> Implement FLANN matching as well as k-nearest neighbor selection. Both methods must be selectable using the respective strings in the main function.

Both matchers were implemented in the file `matching2D_Student.cpp`, in the function `matchDescriptors`.

#### FLANN

```cpp
else if (matcherType.compare("MAT_FLANN") == 0)
{
     if (descSource.type() != CV_32F)
    { // OpenCV bug workaround : convert binary descriptors to floating point due to a bug in current OpenCV implementation
        descSource.convertTo(descSource, CV_32F);
        descRef.convertTo(descRef, CV_32F);
    }

    matcher = cv::FlannBasedMatcher::create();
}
```

#### KNN

```cpp
else if (selectorType.compare("SEL_KNN") == 0)
{ // k nearest neighbors (k=2)

    std::vector<std::vector<cv::DMatch>> knn_matches;
    double t = (double)cv::getTickCount();
    matcher->knnMatch(descSource, descRef, knn_matches, 2); // finds the 2 best matches
    t = ((double)cv::getTickCount() - t) / cv::getTickFrequency();

//...
```

For the FLANN based matching it is necessary to convert binary descriptors to floating point due to a bug in OpenCV implementation.



### MP.6 Descriptor Distance Ratio

> Use the K-Nearest-Neighbor matching to implement the descriptor distance ratio test,
which looks at the ratio of best vs. second-best match to decide whether to keep an
associated pair of keypoints.

The implementation of the distance ration test is quite simple and can be seen on the code bellow.

```cpp
double minDescDistRatio = 0.8;
for (auto it = knn_matches.begin(); it != knn_matches.end(); ++it)
{
    if ((*it)[0].distance < minDescDistRatio * (*it)[1].distance)
    {
        matches.push_back((*it)[0]);
    }
}
```

The implementation of the descriptors can be visualized in the commit [MP.5 MP.6 Descriptor matching and distance ratio ](https://github.com/luisfelipewb/SFND_2D_Feature_Tracking/commit/5b6b0269473a8b90ddc975987b557408d43dcddc).


## Performance

In order to facilitate the evaluation of the results, the output was customized to have a CSV format. To enable this type of output, the `csvOutput` boolean flag was created. The output can be redirected to a file and evaluated using the [PerformanceEvaluation.ipynb](./PerformanceEvaluation.ipynb) Jupyter Notebook. The summarized results are presented in the sections bellow.

### MP.7 Performance Evaluation 1

> Count the number of keypoints on the preceding vehicle for all 10 images and take note of the distribution of their neighborhood size. Do this for all the detectors you have implemented.

For all detectors, the number of keypoints detected and the processing time was registered in the file [performance.csv](./performance.csv).
The table below summarizes the average of the values across each of the 10 images.

![][img_detector_table]

The FAST detector presented the highest number of detected keypoints (more than 400) in the region of interest (ROI) and the lowest computing time of 2 (ms).

![][img_detector_plot]

#### AKAZE

Average number of detected keypoints with distribution mostly around the edges of the preceding vehicle. Medium to small neighborhood size.

![][img_AKAZE_detector]

#### BRISK

Second highest number of detected keypoints, very distributed across the preceding vehicle. Presents all types of neighborhood sizes, including small, medium and large.

![][img_BRISK_detector]

#### FAST

Highest number of detected keypoints (more than 400). Only small neighborhood sizes and very distributed in different regions of the preceding vehicle.
![][img_FAST_detector]

#### HARRIS

Smallest number of detected keypoints (around 25). Only small size neighborhood sizes and concentrated mostly in the roof of the preceding vehicle and some around the taillights.

![][img_HARRIS_detector]

#### ORB

Average number of keypoints with several detections around the taillights. Majority of detections with large neighborhood size.

![][img_ORB_detector]

#### SHITOMASI

Average number of keypoints with distributed detection across the preceding vehicle. All detections with small neighborhood size.

![][img_SHITOMASI_detector]

#### SIFT

Average number of keypoints with several detections distributed across the preceding vehicle including a couple detections in the road. Presents all types of neighborhood sizes, including small, medium and large.

![][img_SIFT_detector]


All algorithms tend to detect points around the taillights, license plate and edges of the car.
Since the region of interested is filtered with a basic rectangle, a few unwanted points on the road and other cars are also detected.

The size of the neighborhood depends on the algorithms. Shi-tomasi, Harris, and Fast have mainly small neighborhood. Akaze, Brisk and Sift have a diverse neighborhood size. Orb has mainly large neighborhood.


### MP.8 Performance Evaluation 2

> Count the number of matched keypoints for all 10 images using all possible combinations of detectors and descriptors. In the matching step, the BF approach is used with the descriptor distance ratio set to 0.8.

To evaluate all possible combinations two vectors containing all detector and descriptor identification strings were created. Using both lists it was possible to easily iterate through all possible combinations in the main loop.

```cpp
// Loop over each possible combination of detector and descriptor types
vector<string> detectorTypes = {"SHITOMASI", "HARRIS", "FAST", "BRISK", "ORB", "AKAZE", "SIFT"};
vector<string> descriptorTypes = {"BRISK", "BRIEF", "ORB", "FREAK", "AKAZE", "SIFT"};
string detectorType;
string descriptorType;

for (auto detIt = detectorTypes.begin(); detIt != detectorTypes.end(); detIt++)
{
    for (auto descIt = descriptorTypes.begin(); descIt != descriptorTypes.end(); descIt++)
    {
        detectorType = *detIt;
        descriptorType = *descIt;
        // Skip combinations that are not possbile.
        if ((descriptorType.compare("AKAZE") == 0) && (detectorType.compare("AKAZE") != 0))
        {
            cout << "Detector " << detectorType << " is not compatible with descriptor " << descriptorType << endl;
            continue;
        }
        if ((descriptorType.compare("ORB") == 0) && (detectorType.compare("SIFT") == 0))
        {
            cout << "Detector " << detectorType << " is not compatible with descriptor " << descriptorType << endl;
            continue;
        }
        cout << "Evaluate detector " << detectorType << " and descriptor " << descriptorType << endl;

        //
        // MAIN LOOP OVER ALL IMAGES
        //

        // Cleanup dataBuffer before testing new combination of detector and descriptor
        dataBuffer.erase(dataBuffer.begin(), dataBuffer.end());
    }
}
```
****

The following combinations could **not** be tested and were skipped using the `continue` statement to 'jump out' of the loop.

- AKAZE descriptor with a non-AKAZE detector. The OpenCV documentation states that this is not possible.
- SIFT detector with ORB descriptor due to out-of-memory error.

The summarized result can be seen in the table bellow. The numerical values are the averages computed across all image combinations

![][img_matches_num_table]

The FAST detector in combination with any descriptor provides the largest amount of matches. In the other hand, the HARRIS detector gives the lowers amount of matches.

### MP.9 Performance Evaluation 3


> Log the time it takes for keypoint detection and descriptor extraction. The results must be entered into a spreadsheet and based on this data, the TOP3 detector / descriptor combinations must be recommended as the best choice for our purpose of detecting keypoints on vehicles.



The resulting times are entered into the table bellow where the numerical values represents the average time in milliseconds across all 10 images.

![][img_combination_table]

Inspecting the table above, it is possible to observe that FAST and HARRIS are the fastest detectors with around 2 and 5 ms respectively.

For the descriptor, we can observe that the BRISK and BRIEF are the fastest when combined with most of the detectors, with times below 2 ms in most of the cases.

The rightmost column `TotalTime` is the summation of the detection and extraction time which represents combined computation time in miliseconds. The **fastest** combination is FAST + ORB with an average just below 3.5 (ms). The **slowest** combination is SIFT + SIFT with and average around 173 (ms) making it less appropriate for any application that requires fast calculation and have limited computing resources.

#### TOP 3 detector and descriptor combination

The top three combination of detector and descriptor were selected using the following factors.
* _Computational time_ - Faster is better for systems with limited resources.
* _Number of keypoints_ - Higher amount of keypoints is better.
* _Localization of keypoints_ - Detections including different parts of the car are more robust
* _Matching between images_ - Better matching leads to fewer false positives and errors.


1. Detector: **FAST**  Descriptor: **BRIEF**

One of the quickest combined processing time with an average of 3.6 (ms) across all images. Also has a large number of matches (average of 242), making it a robust solution.

![][img_fast_and_brief]

2. Detector: **FAST**  Descriptor: **ORB**

The quickest combined processing time with an average of 3.45 (ms). Also has a high amount of matches (average of 229). The reason it was not selected as the first option is because it is possible to observe a few false positives indicating that the combination might not be as robust as the first option.

![][img_fast_and_orb]


3. Detector: **BRISK**  Descriptor: **ORB**

This combination of detector and descriptor is not so fast having a combined processing time of 44.7 (ms). This combination is included in the top three because the keypoints are distributed across many different parts of the preceding vehicle, including taillights, license plate, tires, and rooftop edges. In addition to that, the matching is apparently very effective. False positives cannot be easily observed in the image below.

![][img_brisk_and_orb]
