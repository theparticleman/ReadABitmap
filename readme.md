# Read A Bitmap

The BMP file format is a common image storage format. You can learn more about it [here](http://fileformats.archiveteam.org/wiki/BMP) or [here](https://en.wikipedia.org/wiki/BMP_file_format).

For this code kata we will write code that parses some simple BMP files. We will use TDD to drive our code. You can work individually or in a pair.

## Binary

For this kata you will need to be able to read a file. Often when reading a file we read it as text. But BMP is a binary file format. So for this kata we will need to be able to read files in binary.

The first thing we need to do is make sure we can read a binary file. To help with this, you can use [this](https://github.com/theparticleman/ReadABitmap/blob/master/test_file.bin) file. It contains 6 bytes. The values of those bytes are `0`, `1`, `2`, `3`, `4`, and `5`.

*Test #1*: Write a test to ensure that you can read from this test file and get the correct bytes. Make the test pass.

## Bitmap Header

The first part of a bitmap file is the bitmap header. There are quite a few different possible formats. We'll be looking at the [Wikipedia article](https://en.wikipedia.org/wiki/BMP_file_format) for most of our information. For this kata we'll only worry about the Windows 3.1+ BMP format.

This repository includes two sample bitmap files you may use for testing purposes. Both are 10 pixels by 10 pixels. Both are 24 bits per pixel. One is all white. One has a single red pixel in the upper left corner. You can find the all white one [here](https://github.com/theparticleman/ReadABitmap/blob/master/10x10_24bpp_all_white.bmp) and the one with a single red pixel [here](https://github.com/theparticleman/ReadABitmap/blob/master/10x10_24bpp_one_red_pixel.bmp).

*Test #2*: Write a test to ensure that the first two bytes (the bytes at position `0` and `1`) of the BMP image are `B` (decimal `66`, hex `42`) and `M` (decimal `77`, hex `4D`). If the image does not start with those two bytes the file should not be processed further. You may choose how to implement that (throwing an exception, returning a status code, etc.) Make the test pass. You may use the sample files for this test, but you don't have to.

The bitmap header also let's us know where the actual pixel data (the data that represents what the actual pixels in the image) starts. This is in the bytes at positions `10`, `11`, `12`, and `13`. This should be interpretted as a 32 bit integer. The bytes are in [little endian order](https://chortle.ccsu.edu/AssemblyTutorial/Chapter-15/ass15_3.html), meaning the least significant byte comes first. In the example BMPs, the values of these bytes is `[54, 0, 0, 0]`. This specific example should be translated into just `54` in decimal. But all four bytes could have data in them.

*Test #3*: Write one or more tests to ensure that given 4 bytes in little endian order, that you can properly translate them into a 32 bit integer. Make the test pass.

*Test #4*: Write a test to ensure that the bytes `10`-`13` in the image contain the offset to the pixel data. Make the test pass. Again, you may use the sample files for this test, but you don't have to.

## DIB Header

So what is between the end of the bitmap header and the beginning of the pixel data? Well, there can be more than one thing, but the part that we're most interested in is the DIB header. DIB stands for Device (meaning display device) Independent Bitmap. This header contains additional meta data about the image that we'll need in order to properly parse the pixel data.

First off, there are multiple types of DIB headers. We will only be dealing with one type for this kata. The first four bytes (the bytes at positions `14`-`17`) of the header are the header size, which also indicates the header type. For the format we will be dealing with (called `BITMAPINFOHEADER`) the value of these four bytes should be `40`. Again, these bytes are in little endian order. So the actual bytes will be `[40, 0, 0, 0]`. This should get translated into the integer `40`.

*Test #5*: Write a test to ensure that the bytes `14`-`17` in the image file contain the integer value `40`. If they do not the file should not be processed further. You can choose how this failure happens.

Once we know the DIB header is the type we're interested in, there are three key pieces of information we need to parse the pixel data: image height, image width, and bits per pixel.

*Test #6*: Write a test to ensure that the image width is contained in the bytes `18`-`21` (again in little endian order). For the sample images, this value should be `10`. Make this test pass.

*Test #7*: Write a test to ensure that the image height is contained in the bytes `22`-`25` (yep, little endian order again). For the sample images, this value should be `10`. Make this test pass.

The bits per pixel value represents how many bits are used to represent each pixel in the pixel data portion of the file. This value can be several different values. One bit per pixel means the image is black and white. Eight bits per pixel means each pixel can be any one of 256 colors. The sample images are 24 bits per pixel. This is a common format where the red, green, and blue component of each pixel each get eight bits (or one byte).

*Test #8*: Write a test to ensure that the bits per pixel is contained in the bytes `28` and `29`. This value is only 2 bytes instead of 4, but the bytes are still in little endian order.