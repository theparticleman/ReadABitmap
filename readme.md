# Read A Bitmap

The BMP file format is a common image storage format. You can learn more about it [here](http://fileformats.archiveteam.org/wiki/BMP) or [here](https://en.wikipedia.org/wiki/BMP_file_format).

For this code kata we will write code that parses some simple BMP files. We will use TDD to drive our code. You can work individually or in a pair.

## Binary

For this kata you will need to be able to read a file. Often when reading a file we read it as text. But BMP is a binary file format. So for this kata we will need to be able to read files in binary.

The first thing we need to do is make sure we can read a binary file. To help with this, you can use [this](https://github.com/theparticleman/ReadABitmap/blob/master/test_file.bin) file. It contains 6 bytes. The values of those bytes are `0`, `1`, `2`, `3`, `4`, and `5`.

*Test #1*: Write a test to ensure that you can read from this test file and get the correct bytes. Make the test passes.

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

*Test #8*: Write a test to ensure that the bits per pixel is contained in the bytes `28` and `29`. This value is only 2 bytes instead of 4, but the bytes are still in little endian order. (Feel free to add another test to ensure that you can convert from two little endian bytes to an integer instead of four bytes if you want.) In the sample files these bytes are `[24, 0]` and should translate into the integer `24`. Make the test pass.

## Pixel Data

At this point we have enough information to start parsing the actual pixel data. We know where the data begins. We know how many bits each pixel takes up. We'll also need to know the width and height of the image, which we have. We do need to know how the data is packaged though.

The pixel data is grouped into rows. Each row roughly translates into a row in the image. But each row does get padded out to a multiple of 4 bytes. So in our simple images, each row is 10 pixels. Each of those pixels takes up 3 bytes. So each row has 30 bytes of pixel data. However, each row will get padded out to a multiple of 4. So in this case each row will be 32 bytes. The first 30 bytes contain the actual values for the 10 pixels in the row and then the last 2 bytes will just be zeroes.

How about a few examples?

The first pixel in the pixel data section is stored in bytes `54`, `55`, and `56`. That's simply the offset we read from the header (`54`), then add 0, 1, and 2. for the 3 bytes (24 bits).

The first pixel in the second row in the pixel data section is stored in bytes `86`, `87`, and `88`. To calculate the first index of that pixel you need to know that the row size is 32 bytes (see above for why it's 32 instead of 30). We start at the offset (`54`) and then skip one entire row's worth of bytes (`32`) for `54 + 32 = 86`. We then add 0, 1, and 2 like before.

What about the last pixel in the last row? That pixel is stored in bytes `369`, `370`, and `371`. You may have noticed that each of the 10x10 images are 374 bytes. Why isn't the last index `374` then? Two reasons. First, because the indexes are zero-based. So the last byte in the file is at position `373`. Second, because the rows are padded out to an even multiple of 4 bytes. So the last 2 bytes of the file (`372` and `373`) are simply padding for the last row.

How do we calculate the first index of that last pixel you ask? We take our original offset from the header (`54`), the add the row size (`32`) times the number of rows we're skipping (`9` or `height - 1`), then the number of bytes per pixel (`3` bytes because each pixel is 24 bits) times the number of pixels we're skipping (`9` or `width - 1`) which gives us `54 + (32 * 9) + (3 * 9) = 369`. Tada!

*Test #9*: Write a test to ensure that the pixel data can be parsed correctly. If you use the sample, all white image you could ensure that the first and last pixel are all white, as a place to start. Make the test pass.

## From A User Perspective

Being able to get all of the individual pieces of data from a BMP image file is good, but that might not be the most helpful for actually interacting with a BMP file. For this part, create a data structure of some kind (if you haven't already) that will hold the basic meta data about the image (width, height, bits per pixel) and allows getting the color of an individual pixel by it's `x` and `y` coordinates. You can create another data structure to represent the color if you want.

The one last fun bit of BMP devilry we'll deal with in this kata is order of the data in the pixel data. If you were to create a system to address individual pixels using the all white sample image this would be difficult to detect, but if you use the image with one red pixel you would notice that rather than being the first three bytes in the pixel data, the one red pixel is the first three bytes of the _last_ row in the pixel data. Yes, the rows in the pixel data are in the *opposite* order when compared to the actual image.

If that's not enough to bake your noodle, in the images we're using with 24 bits per pixel, the RGB values aren't stored in the order you might think: red, green, blue. Rather they are stored blue, green, red. So the three bytes of the one red pixel are `[0, 0, 255]` rather than `[255, 0, 0]`.

*Test #10*: Write a test to ensure that your image data structure works correctly. The width, height and bits per pixel should be correct based on the input. Indexing individual pixels should work correctly. The index `(0, 0)` should return the color data for the pixel in the upper left-hand corner of the image. The index `(width - 1, height - 1)` should return the color data for the pixel in the lower right-hand corner of the image. For the image with one red pixel, the pixel at `(0, 0)` is the red one.

## Bonus Points

If you get done with the above tests, some other tests and functionality you could add include
- Make sure your code works with BMP files that have a different bits per pixel value. If you want to test with an actual file you may need to generate some additional files beyond the sample ones provided. Write tests to make sure this functionality works properly.
- Add the ability to modify the color of individual pixels and save the modified BMP out to a file. Write tests to make sure your modified file can be read back in by your code and that your changes were made correctly. Also test with an image viewer.
