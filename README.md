Hello,

This is my code submission for the NASA Kibo RPC challenge

The files used for the APK are located in SampleApk

Files used to generate images and sort files are in python

Ai models used are located in saved_model and yolo models

Our code used a yolo model ported to tflite suitable for android devices, having 97% accuracy in detecting images

There is also a java class that was used to properly find the elements in the image without having to move around a lot based on opencv aruco tag detection 

Sample training images that were generated are located in the training_images directory. Overall the model was trained on 40,000 randomly generated images.
