# TinyML-on-Arduino-Nano-33-BLE-Sense-with-Teachable-Machine
Building a TinyML model on the Google Teachable Machine and Deploying it on Arduino Nano 33 BLE Sense.

[![You can watch the full tutorial here:](https://img.youtube.com/vi/nM_KmuM0QJU/0.jpg)](https://youtu.be/nM_KmuM0QJU)

## Background
I hope you haven’t struggled a lot to get your model trained on Google Teachable Machine to be uploaded on Arduino Nano 33 BLE Sense before finding this tutorial, everything will be well explained and solved in this guide. We will solve this error:

```
C:\Users\<my user id>\AppData\Local\Temp\arduino-sketch-D67097C3451979EF653B299D62F9FCA5/linker_script.ld:138 cannot move location counter backwards (from 20053278 to 2003fc00)
collect2.exe: error: ld returned 1 exit status
```

Let us get into it, one good thing that we can do with the Arduino Nano 33 BLE Sense is the deployment Tiny Machine Learning (TinyML) workloads on it but before we deploy any model, it sure has to be trained first and there are different ways we can do this, the traditional method is to write some code to train the TensorFlow model and then convert it to TensorFlow Lite to be able to stay on a small microcontroller like the Arduino and the other easy method is to make use of platforms like Google Teachable Machine, we just pass in our data and we get the TFLite model as output, in this guide, we will be looking at to achieve this method, the challenges involved and how to overcome them.

## Hardware Setup
Feel free to skip through this if you already have everything set but for the case of others reading this for the first time, let's start from the beginning. What hardware will you need for this guide?

1. Arduino Nano 33 BLE Sense
2. OV767X Camera (I have OV7675 with me, anyone will work)
3. Breadboard and Jumper Wires
4. Optional: Tiny Machine Learning Shield

The first thing you want to do is connect your camera to your Arduino board, just plug in the Arduino and the Camera to a breadboard and wire everything up according to this diagram
![image](https://github.com/user-attachments/assets/6f2a739f-bdbf-4d6a-b614-485344c2e4da)

Once this is done, you are good to go, I mentioned above that it is optional but if you have the Tiny Machine Learning kit, it makes things easier, just plug in the Arduino and the camera as shown below

![image](https://github.com/user-attachments/assets/8d3d7a2d-4e6e-4d3a-b9f0-a333416d2245)

## Software Setup

As earlier mentioned, we are using Teachable Machine so the first thing you have to do is open up Google Teachable Machine in a browser by going to the link: https://teachablemachine.withgoogle.com, click on get started, after that, you will be presented with three options: Image Project, Audio Project, and Pose Project, select image project and then embedded image model.

![image](https://github.com/user-attachments/assets/1785de8f-cdb5-4ce4-b602-e3352c733e12)

After this, you will be presented with the interface to collect data and train your model. There are three options for importing data: Camera Webcam, Files Upload, and direct Arduino upload, the best is the direct Arduino upload and that is what we will be considering, don’t forget to name your class (Object) by clicking the pencil icon.

![image](https://github.com/user-attachments/assets/34a52905-4bb9-4417-9228-a1afdf7c1ec3)

## Uploading Data Directly from Arduino
Click on Device to upload data directly from Arduino, there are three things to do here: Download two source codes and the Processing IDE, and the platform will present to you two codes, TM_Uploader and TM_Connector.

![image](https://github.com/user-attachments/assets/03c42cfa-3324-40f3-8723-31f0634a46dc)

Downloads these zip files to your laptop and then you can go grab the Processing IDE. We need this IDE to be able to open the code that connects the Arduino to the Teachable Machine. Kindly go to the site: https://processing.org/releases and download the 3.5.4 Version, as when preparing this guide, August 2024, the latest version 4.3 won’t work.

![image](https://github.com/user-attachments/assets/2753684a-264f-4d4c-8778-a9d5e463452c)

Unzip both the Processing IDE code (TM_Connector) and the Arduino Code (TM_Uploader).

## Now to the interesting part

*A quick house cleaning before we go: Go to your Arduino Libraries and download the Arduino_OV767X Library, we also need the Arduino_Tensorflow_Lite 2.0-ALPHA Library which is not in the Libary manager, you can find it here: Arduino TensorFlow_Lite 2.0-ALPHA and then add the .zip library.*

If you open up the TM_Uploader Arduino code, you should have three tabs: TM_Uploader.ino, ImageProvider.cpp, and ImageProvider.h. Our main focus is ImageProvider.cpp.

If you study the code, you will notice that the code takes 320 X 240 image pixels and crop it to 96 X 96, basically, the image is converted from QVGA to RGB565. The problem is that if we are to work with this original design, by the time we are ready to deploy our model, it will be too big for the Arduino and we will run into the error:
```
C:\Users\<my user id>\AppData\Local\Temp\arduino-sketch-D67097C3451979EF653B299D62F9FCA5/linker_script.ld:138 cannot move location counter backwards (from 20053278 to 2003fc00)
collect2.exe: error: ld returned 1 exit status
```
What we need to do is reduce the image size to 160 X 120 (QQVGA format) and because of this, we have to readjust the ImageProvider.cpp code. We will effect the following changes:

1. Change the image dimension

Original:
```
const int kCaptureWidth = 320;
const int kCaptureHeight = 240;
```
Replace with:
```
const int kCaptureWidth = 160;
const int kCaptureHeight = 120;
```
2. Change the conversion code

Original: Please note I have cut out some parts to minimize the length. The three dots mean every code that falls in the middle also should be cut out (Everything inside the for loop).
```
  // Color of the current pixel
  uint16_t color;
  for (int y = 0; y < imgSize; y++) {
    for (int x = 0; x < imgSize; x++) {
      int currentCapX = floor(map(x, 0, imgSize, 40, kCaptureWidth - 80));
      int currentCapY = floor(map(y, 0, imgSize, 0, kCaptureHeight));
      // Read the color of the pixel as 16-bit integer
      int read_index = (currentCapY * kCaptureWidth + currentCapX) * 2;//(y * kCaptureWidth + x) * 2;
.
.
.
      // Convert to signed 8-bit integer by subtracting 128.
      //
      //      // The index of this pixel` in our flat output buffer
      int index = y * image_width + x;
      image_data[index] = static_cast<int8_t>(gray_value);
//      delayMicroseconds(10);
    }
  }
//  flushCap();
  //  Serial.println("processed image");
```

Replace with:

```
  for (int y = 0; y < imgSize; y++) {
    for (int x = 0; x < imgSize; x++) {
      int currentCapX = floor(map(x, 0, imgSize, 0, kCaptureWidth));
      int currentCapY = floor(map(y, 0, imgSize, 0, kCaptureHeight));

      int read_index = (currentCapY * kCaptureWidth + currentCapX) * 2;
      uint8_t Y1 = captured_data[read_index];
      uint8_t U = captured_data[read_index + 1];
      uint8_t Y2 = captured_data[read_index + 2];
      uint8_t V = captured_data[read_index + 3];

      // Convert YUV to RGB
      int r = Y1 + 1.402 * (V - 128);
      int g = Y1 - 0.344136 * (U - 128) - 0.714136 * (V - 128);
      int b = Y1 + 1.772 * (U - 128);

      r = min(max(r, 0), 255);
      g = min(max(g, 0), 255);
      b = min(max(b, 0), 255);

      // Convert RGB to grayscale
      float gray_value = 0.2126 * r + 0.7152 * g + 0.0722 * b;
      int index = y * image_width + x;
      image_data[index] = static_cast<int8_t>(gray_value);
    }
  }
```
What we did here is to convert from QQVGA (160 X 120) to YUV422. We need to pass this to the GetImage function, go down to the function, and replace the line:*!Camera.begin(QVGA, RGB565, 1)* with:
```
(!Camera.begin(QQVGA, YUV422, 1))
```
Once you are done with this, go ahead and upload the code on the Arduino board, once uploaded, don’t open the serial monitor, open up the other code we downloaded, TM_connector, and click the play button (no correction here)

![image](https://github.com/user-attachments/assets/22750194-fded-4333-bdd3-07a898df7cfe)

You will be presented with a screen displaying your camera field after selecting the port. **In case you are getting the error: Port busy** double-click on the Arduino reset button and then click once, from experience, this should make things work.

![image](https://github.com/user-attachments/assets/05d5b324-4e28-4465-916d-100e1d084041)

Go back to the Teachable machine and click on **Attempt to connect to Device**. You should be seeing your image frames in TM now, hold down and record your images, once you have the perfect image coming from the Arduino. Do this for all other classes you want to train your model with, once done, click **train embedded model.**

![image](https://github.com/user-attachments/assets/666a5700-2b6b-4f86-83cc-31831813bc02)

Once trained, you can test your model by clicking on **Attempt to connect to Device** under the Preview section, once you are satisfied, you can click on **Export Model**, then select _TensorFlow Lite for Microcontrollers_ under TensorFlow Lite, and then download the model.

## Model Deployment

Once the model is downloaded, you will have a folder that looks like this:

![image](https://github.com/user-attachments/assets/85771eee-4894-472d-80d9-3476994fa581)

Open up the Arduino tm_template_script file, it will ask you to add it to a folder (you know the way Arduino sketches work), do this and then go back to copy the rest of the files in the folder that the Arduino IDE created so they all show up on your Arduino screen.

If you try to upload this directly, you will get an error regarding the variable **catDataLen** I don’t know why the TensorFlow team forgot this but just go back to your former TM_Uploader code and copy the capDataLen variable declaration: _const int capDataLen = kCaptureWidth * kCaptureHeight * 2;_ your code should now looks like this:

![image](https://github.com/user-attachments/assets/ece7ac55-f4ce-43c0-bdb9-1679c6a845ea)

We added line 49.

If you try to recompile, you will run into another error, probably this is the error you have been facing if you try to upload this directly, that linker script error:

![image](https://github.com/user-attachments/assets/53344d09-2117-405a-94b2-5c9102c1ca20)

What we need to do is the same as what we did above to the uploader code, just go over to **arduino_image_provider.cpp** tab and change from 320 X 240 to 160 X 220, replace the whole conversion for loop with what we used above, and change the GetImage function to (_!Camera.begin(QQVGA, YUV422, 1_)) If you have any doubt on how to do this, you can check out the complete repository here (https://github.com/adesolasamuel/TinyML-on-Arduino-Nano-33-BLE-Sense-with-Teachable-Machine). Even though I know this is the most important part, I guess you can just easily use the code above when changing the uploader code instead of still repeating the steps here, in case you are not able to do this, please drop a comment and I will show you the step again or you can just watch my YouTube demo(https://www.youtube.com/watch?v=nM_KmuM0QJU).

Upload the code now and everything should compile fine and get uploaded and you will be able to make your image classification, 128 means 100%, and — 127 means 0%, so that is it guys, I am sorry if the writeup is too long, I tried to make it shorter!

![image](https://github.com/user-attachments/assets/9b67e681-5a16-49f4-a102-2a7f5ac45817)

If you want to watch a video of all the steps explained here, kindly check this Video (https://www.youtube.com/watch?v=nM_KmuM0QJU) and please like and subscribe to my channel.

Wait, hold on, in case you want to be able to do this same project without Teachable Machine but writing everything from scratch on CoLab, check this out: https://www.youtube.com/watch?v=Fy_dqeNu2IA&t

Thanks so much, guys, drop any comment at all and please you can follow me to support me and see my articles. Happy Coding!!!













