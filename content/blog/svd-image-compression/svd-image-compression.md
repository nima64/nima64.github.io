---
title: Singular Value Decomposition and it's use in Image compression.
description: Learning how to compress images in C++ using Eigen and stb image.
date: Created
tags:
  - linear algebra
---

While reading through Strang's _Linear Algebra and It's Applications_ I came upon this passage about SVD and one of it's many applications image compression.

{% image "./SVD-Strang.png" ,"test"%}

SVD can be rewritten in a more compact form with the outer product being explicit.

$$\sum_{k=1}^{r}\sigma_{k}u_{k} \otimes v_{k}$$

## How SVD Compression Works

SVD compression lies on the fact most of the information is contained in very few singular values $$\sigma$$ being very large while the rest are close to 0.

The example by Strang: given a 1000x1000 matrix and say 98% of singular values are close to zero then we can get rid of them. So that only leaves us with 20 singular values, r=20, from the summation we only need 20 columns of U and V^T to reproduce the image.

## Testing it in code with stbi and Eigen

```cpp
#include <Eigen/Dense>
#include <cstdio>
#include <cstdlib>
#include <fstream>
#include <iostream>
#include <malloc/_malloc.h>
#include <vector>
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"

using namespace Eigen;

void compressImage(const char *path) {
  int x, y, n;
  unsigned char *data = stbi_load(path, &x, &y, &n, 0);
  int dataSize = x * y;

  if (data == NULL) {
    std::perror("Failed to Open File!");
  } else {
    printf("x:%d y:%d n:%d\n", x, y, n);
  }

  Eigen::MatrixXd imgXY(y, x); // Correct the order of y and x

  Eigen::Vector3f greyConversion(0.299, 0.587, 0.114);

  for (int i = 0; i < dataSize; i++) {
    auto dataRGB =
        Eigen::Vector3f(data[i * n], data[i * n + 1], data[i * n + 2]);
    int greyCode = (int)dataRGB.dot(greyConversion);

    int row = i / x;
    int col = i % x;
    imgXY(row, col) = greyCode;
  }

  BDCSVD<MatrixXd> svd(imgXY, ComputeFullU | Eigen::ComputeFullV);

  Eigen::VectorXd singularValues = svd.singularValues();
  for (auto i = singularValues.size() - 350; i != singularValues.size(); i++) {
    singularValues[i] = 0;
  }
  auto U = svd.matrixU();
  auto V = svd.matrixV();
  Eigen::MatrixXd mSV = singularValues.asDiagonal().toDenseMatrix();
  mSV.conservativeResize(U.cols(), V.cols());

  printf("U cols: %lu rows: %lu\n", U.cols(), U.rows());
  printf("msv cols: %lu rows: %lu\n", mSV.cols(), mSV.rows());
  printf("V cols: %lu rows: %lu\n", V.cols(), V.rows());

  // Reconstruct the image matrix
  MatrixXd cImgMat = U * mSV * V.transpose();

  // convert to char for stbi
  std::vector<char> charVec;

  for (int row = 0; row < cImgMat.rows(); row++) {
    for (int col = 0; col < cImgMat.cols(); col++) {
      charVec.push_back(cImgMat(row, col));
    }
  }

  // stride is size of each row
  stbi_write_png("lennaBW.png", x, y, 1, charVec.data(),
                 sizeof(unsigned char) * x);

  stbi_image_free(data);
}

int main() {
  compressImage("lenna.png");
}
```

## The result

The image was converted gray scale for the sake of simplicity, for color you need to run SVD on all the channels.

{% image "./Lenna.png" ,"test"%}
{% image "./LennaBW.png" ,"test"%}
