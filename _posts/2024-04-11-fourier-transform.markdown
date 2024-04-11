---
layout: post
title:  "Fourier Transform"
date:   2024-04-11
categories: image
---
The Fourier Transform is a powerful tool for analyzing the frequency components of digital images. Originally rooted in the analysis of signals, Fourier transforms break down complex signals into simpler sinusoidal waves. When applied to images, it transforms the spatial domain of the image (i.e., the pixel intensities) into the frequency domain, revealing the image's underlying spatial frequency components.

**Basic Principles of Discrete FT (DFT)**

The DFT is particularly useful in image processing because it provides a way to analyze the content of an image in terms of its frequency components. The basic idea is that any image can be represented as a sum of sinusoidal functions of varying magnitudes, frequencies, and phases. This transformation from spatial to frequency domain is crucial for many applications because alterations and enhancements in the frequency domain can be inversely transformed back to the spatial domain, allowing modified images to be reconstructed with desired characteristics.

**Formula**

The DFT is essentially a sampled version of the Fourier Transform and thus captures only a subset of frequencies, sufficient to comprehensively describe the original image in the spatial domain. The set of sampled frequencies in the DFT corresponds directly to the pixel count in the spatial domain image. Consequently, both the spatial domain image and its Fourier transform representation maintain the same dimensions.
Given a discrete square image F(x,y) of size NxN the DFT is:

$$F(u,v) = \sum_{x=0}^{N-1} \sum_{y=0}^{N-1} f(x,y)e^{-i2\pi(\frac{xu}{N}+\frac{yv}{N})}$$

**DFT**

Let's see in this example. I took a picture of the tennis balls can. Using OpenCV and DFT (OpenCV uses Fast Fourier Transform - it's an algorithm that computes DFT in $$O(NlogN)$$ instead of $$O(N^2)$$ time) we get this:

![DFT](https://github.com/BlinkID/blinkid-ios/assets/26868155/3da7aaab-e094-472e-ab22-89c9a77fa0f1)

The image is represented in a complex form, comprising both magnitude and phase components. For visualization purposes, the magnitude component is particularly significant as it reflects the strength of each frequency component, offering a clear insight into the image's frequency spectrum. However, because the center of the image typically exhibits significantly higher values than other areas, magnitudes are often displayed on a logarithmic scale to compress the dynamic range and enhance visibility.

Within the Fourier domain, the central areas correspond to low-frequency components, while the outer regions denote high-frequency components. The very center of the image signifies the DC value, which is the zero frequency component representing the overall intensity of the image.

Compressing the dynamic range of an image can be achieved by transforming each pixel value into its logarithm. This transformation effectively enhances the visibility of lower intensity pixels. Such a technique is particularly valuable in scenarios where the dynamic range is too extensive for conventional display screens or to be captured accurately by film. The logarithmic operation functions as a straightforward point processor, applying a logarithmic curve to map the original pixel values. While the choice between using a natural logarithm or a base 10 logarithm varies, it primarily affects the scale of the output values rather than the curve's shape. This ensures that regardless of the logarithmic base used, the compression effect on the dynamic range remains consistent. The values are then scaled to suit an 8-bit display system. The mapping function for this operation can be described as follows:

$$\log_{10}(1 + ||F(u,v)||)$$

**Inverse DFT**

The Inverse Discrete Fourier Transform (IDFT) is an essential mathematical tool in digital signal processing and image processing, serving as the counterpart to the Discrete Fourier Transform (DFT). While the DFT converts a signal or image from the spatial domain (or time domain) to the frequency domain, the IDFT performs the reverse, transforming the data back to its original spatial domain from the frequency domain. This process is crucial for applications that modify signals in the frequency domain and require a reconstruction of the original signal or image.

The mathematical formula for the IDFT in a two-dimensional space, typical for images, is expressed as follows for NxN images:

$$F(x,y) = \frac{1}{N^2} \sum_{u=0}^{N-1} \sum_{v=0}^{N-1} f(u,v)e^{i2\pi(\frac{xu}{N}+\frac{yv}{N})}$$

**Filters**

Filters in image processing function exactly as their name implies—they filter out unwanted elements. These filters are generally implemented as a mask array, the same size as the original image. When this mask is placed over the original image, it selectively retains only the desired attributes.

As previously discussed, in an image transformed by the DFT, low frequencies are located at the center, while high frequencies are dispersed around the periphery. By designing a mask array with a central circle of zeros surrounded by ones, this filter can be applied to emphasize high frequencies in the image.

There are mainly three types of filter used:

- Low Pass Filter (LPF)
- High Pass Filter (HPF)
- Band Pass Filter (BPF)

**Edge Detection**

Edge detection is a crucial technique in computer vision. By identifying the edges within an image, we can utilize this information for feature extraction or pattern detection purposes.

Edges are typically characterized by high frequencies in an image. Therefore, after performing a DFT on an image to convert it into the frequency domain, a High-Pass Filter is applied. This filter effectively blocks all low frequencies while allowing high frequencies to pass through. Subsequently, applying an inverse DFT to this filtered image reveals distinct edge features in the original image, making it a powerful tool for analysis and processing in various applications.

When applying HPF on the image above, and inverse DFT, we get:

![Inverse_DFT](https://github.com/BlinkID/blinkid-ios/assets/26868155/f71e1961-ebf3-49bf-870e-3924a2d8c325)

**Blur Detection**

Detecting blur in an image using the Fourier Transform involves analyzing the frequency domain representation of the image to determine the absence or presence of high-frequency components, which are typically diminished in blurred images. Below, I provide a C++ example that uses the OpenCV library to perform this task.

```
#include <opencv2/opencv.hpp>
#include <iostream>

bool isBlurry(const cv::Mat& image) {
    cv::Mat gray, floatGray, dftImage, magnitudeImage;
    
    // Convert the image to grayscale
    cv::cvtColor(image, gray, cv::COLOR_BGR2GRAY);
    
    // Convert the grayscale image to float
    gray.convertTo(floatGray, CV_32F);
    
    // Compute the DFT
    cv::dft(floatGray, dftImage, cv::DFT_COMPLEX_OUTPUT);
    
    // Compute the magnitude of the complex numbers (real and imaginary)
    std::vector<cv::Mat> planes;
    cv::split(dftImage, planes); // planes[0] = Re(DFT(I)), planes[1] = Im(DFT(I))
    cv::magnitude(planes[0], planes[1], magnitudeImage);
    
    // Shift the DC component to the center of the image
    int cx = magnitudeImage.cols / 2;
    int cy = magnitudeImage.rows / 2;
    cv::Mat q0(magnitudeImage, cv::Rect(0, 0, cx, cy));   // Top-Left
    cv::Mat q1(magnitudeImage, cv::Rect(cx, 0, cx, cy));  // Top-Right
    cv::Mat q2(magnitudeImage, cv::Rect(0, cy, cx, cy));  // Bottom-Left
    cv::Mat q3(magnitudeImage, cv::Rect(cx, cy, cx, cy)); // Bottom-Right

    // Swap quadrants (Top-Left with Bottom-Right, Top-Right with Bottom-Left)
    cv::Mat tmp;
    q0.copyTo(tmp);
    q3.copyTo(q0);
    tmp.copyTo(q3);
    q1.copyTo(tmp);
    q2.copyTo(q1);
    tmp.copyTo(q2);
    
    // Threshold to determine the presence of significant high frequencies
    double meanValue = cv::mean(magnitudeImage)[0];
    std::cout << "Mean value of DFT magnitude: " << meanValue << std::endl;

    // Threshold the mean value to determine if the image is blurry
    return meanValue < 0.2; // Adjust the threshold based on experimentation
}

```

Let's see what's happening here:

1. **Grayscale Conversion**: The image is converted to grayscale because the color information is not necessary for blur detection.
2. **Fourier Transform**: The grayscale image is converted to the frequency domain using the DFT.
3. **Magnitude Calculation**: The magnitude of the DFT results is calculated to analyze the frequency components.
4. **Quadrant Swapping**: This step shifts the zero-frequency component to the center of the spectrum, which makes analysis more intuitive.
5. **Mean Value Check**: By calculating the mean of the magnitude spectrum, you can infer the presence of high-frequency components. A lower mean suggests fewer high frequencies, indicative of a blur.