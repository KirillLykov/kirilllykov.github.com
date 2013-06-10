---
layout: post
title: "Seam carving algorithm"
date: 2013-06-06 22:41
comments: true
categories: 
---
##Intro
Seam carving is an algorithm for content-aware image resizing, it was described in the paper by [S. Avidan & A. Shamir](http://www.win.tue.nl/~wstahw/2IV05/seamcarving.pdf). In contract to stretching, content-aware resizing
allows to remove/add pixels which has less meaning while saving more important. Pictures below demonstrate this - original picture of the size 332x480 is on the top, the picture after applying seam carving (size is 272x400) is on the bottom
<center>
<img src="../../../../../images/seamcarving/sea-thai.jpg">
<img src="../../../../../images/seamcarving/sea-thai-reduced.jpg">
</center>
This algorithm is quite impressive so one may find a lot of articles describing this algorithm. Yet as I found most of the authors haven't read the original paper and provide very basic implementation. In this post I will describe the algorithm with all details it was written by Avidan & Shamir. But I will write from the programmers point of view, without going into Math too deep. In addition to algorithms description, I also provide Matlab code.
<!--more-->
##Energy
For simplification, we will describe only reducing the size of the image. But enlarging process is very similar and described in the last section. 
The idea is to remove content that has smaller meaning for the user (contain less information). We will call this information "energy". Thus we need to introduce an energy function that would map a pixel into
energy value. For instance, we can use gradient of the pixel: <img src="http://latex.codecogs.com/gif.latex?e_1= \left|\frac{\partial I}{\partial x}\right| + \left|\frac{\partial I}{\partial y}\right|" style="border: none; box-shadow: none;vertical-align:middle"/>. If a picture has 3 channels, just sum values of the energy in each channel. The Matlab code below demostates it. The `imfilter` function just applies the mask to each pixel, so the result dI(i, j)/dx = I(i+1)-I(i-1)/dx where dx = 1. 
```matlab
function res = energyRGB(I)
% returns energy of all pixelels
% e = |dI/dx| + |dI/dy|
    res = energyGrey(I(:, :, 1)) + energyGrey(I(:, :, 2)) + energyGrey(I(:, :, 3));
end

function res = energyGrey(I)
% returns energy of all pixelels
% e = |dI/dx| + |dI/dy|
    res = abs(imfilter(I, [-1,0,1], 'replicate')) + abs(imfilter(I, [-1;0;1], 'replicate'));
end
```
Here is energy function:
<center>
<img src="../../../../../images/seamcarving/sea-thai-energy.jpg">
</center>
##Seam
If we delete pixels with minimum energy but random positions, we will get distorted picture. If we delete columns/rows with minimum energy, we will get artifacts. Here by column I mean {(i, j)| j is predefined}, row - {(i, j)| i is predefined}. The solution is to introduce a generalization of column/row (called `seam`). Formally, let I is n x m image, then a vertical seam is <img src="http://latex.codecogs.com/gif.latex?(s^x)_i=(i, x(j)) s.t. \forall i, \left|x(i) - x(i-1)\right| \leq 1" style="border: none; box-shadow: none;vertical-align:middle"/>, where x: [1,..,n] -> [1,..,m]. It means that a vertical seam is path from the top of the picture to the bottom such that the length of the path in pixels is width of the image, and for each seam element (i,j),
the next seam element can be only (i+1, j-1), (i+1, j), (i+1, j+1). Similarly, we can define a horizontal seam. Examples of seams are shown on the figure below in black:
<center>
<img src="../../../../../images/seamcarving/sea-thai-seams.jpg">
</center>
We are looking for a seam with the minimum energy among all seams (in chosen dimension):  <img src="http://latex.codecogs.com/gif.latex?s^* = \[\min_{s} \sum_{i=1}^{n} e(I(s_i)))\]" style="border: none; box-shadow: none;vertical-align:middle"/>.
The way to find such an optimal way is by using dynamic programming:


1. Find M - minimum energy for all possible seams for each (i, j):
   * fill in the first row by energy
   * for all rows starting from second: 
       M[i, j] = e[i, j] + min(M[i - 1, j], M[i, j], M[i - 1, j]); 
2. Find the minimum value in the last row of M and traverse back chousing pixels with minimum energy.


Note that in Matlab code I had to represent seam as n x m bit matrix. If pixel is in the seam it is 0, otherwise 1.
```matlab
function [optSeamMask, seamEnergy] = findOptSeam(energy)
% finds optimal seam
% returns mask with 0 mean a pixel is in the seam

    % find M for vertical seams
    % for vertical - use I`
    M = padarray(energy, [0 1], realmax('double')); % to avoid handling border elements

    sz = size(M);
    for i = 2 : sz(1)
        for j = 2 : (sz(2) - 1)
            neighbors = [M(i - 1, j - 1) M(i - 1, j) M(i - 1, j + 1)];
            M(i, j) = M(i, j) + min(neighbors);
        end
    end

    % find the min element in the last raw
    [val, indJ] = min(M(sz(1), :));
    seamEnergy = val;
    
    optSeamMask = zeros(size(energy), 'uint8');
 
    %go backward and save (i, j)
    for i = sz(1) : -1 : 2
        %optSeam(i) = indJ - 1;
        optSeamMask(i, indJ - 1) = 1; % -1 because of padding on 1 element from left
        neighbors = [M(i - 1, indJ - 1) M(i - 1, indJ) M(i - 1, indJ + 1)];
        [val, indIncr] = min(neighbors);
        
        seamEnergy = seamEnergy + val;
        
        indJ = indJ + (indIncr - 2); % (x - 2): [1,2]->[-1,1]]
    end

    optSeamMask(1, indJ - 1) = 1; % -1 because of padding on 1 element from left
    optSeamMask = ~optSeamMask;
end
```
In order to find a horizontal seam, just pass a transposed energy matrix to `findOptSeam`.

##Find optimal order of deleting seams
Now we can compute seams and using the code below we can even remove them from an image:
```matlab
function imageReduced = reduceImageByMaskVertical(image, seamMask)
    imageReduced = zeros(size(image, 1), size(image, 2) - 1, size(image, 3));
    for i = 1 : size(seamMask, 1)
        imageReduced(i, :, 1) = image(i, seamMask(i, :), 1);
        imageReduced(i, :, 2) = image(i, seamMask(i, :), 2);
        imageReduced(i, :, 3) = image(i, seamMask(i, :), 3);
    end
end

function imageReduced = reduceImageByMaskHorizontal(image, seamMask)
    imageReduced = zeros(size(image, 1) - 1, size(image, 2), size(image, 3));
    for j = 1 : size(seamMask, 2)
        imageReduced(:, j, 1) = image(seamMask(:, j), j, 1);
        imageReduced(:, j, 2) = image(seamMask(:, j), j, 2);
        imageReduced(:, j, 3) = image(seamMask(:, j), j, 3);
    end
end
```
It is already a good tool for reducing image in one dimension - just find and delete seam as many times as you need. But what if you need to reduce the
size of the image in both directions? How to decide at every iteration whether it is better (in terms of energy minimization) to delete a column or a row?
This problem is solved, again, using dynamic programming. We introduce a transport matrix T which defines for every n' x m' the cost of the optimal sequence 
of horizontal and vertical seam removal operations (n' < n, m' < m). It is more suitable to introduce r = n - n' and c = m - m' which defines number of horizontal
and vertical removal operations. In addition to T we introduce a map of the size r x c TBM which specifies for every T(i, j) whether we came to this point using 
horizontal (0) or vertical (1) seam removal operation. Pseudocode is shown below:

```c
1) T(0, 0) = 0; 
2) Intialize borders of T:
   for all j {
       T(0, j) = T(0, j - 1) + E(seamVertical);
   }
   for all i { 
       T(i, 0) = T(j - 1, 0) + E(seamHorizontal); 
    }
3) Initialize borders of TBM:
   for all j { T(0, j) = 1; }
   for all i { T(0, i) = 0; }
4) Fill in T and TBM (c-like pseudocode):
   for i = 2 to r {
       imageWithoutRow = image;
       for j = 2 to c {
            energy = computeEnergy(imageWithoutRow);

	    horizontalSeamEnergy = findHorizontalSeamEnergy(energy);
	    verticalSeamEnergy = findVerticalSeamEnergy(energy);
	    tVertical = T(i - 1, j) + verticalSeamEnergy;
	    tHorizontal = T(i, j - 1) _ horizontalSeamEnergy;
	    if (tVertical < tHorizontal) {
	       T(i, j) = tVertical;
	       transBitMask(i, j) = 1
	    } else {
	       T(i, j) = tHorizontal;	    
	       transBitMask(i, j) = 0
	    }
            // move from left to right - delete vertical seam
            imageWithoutRow = removeVerticalSeam(energy);
        }

        energy = computeEnergy(image);
        image = removeHorizontalSeam(energy);
    }
5) Go backward starting from T(r, c) and following the TBM.
```
And Matlab imeplementation:
```matlab
function [T, transBitMask] = findTransportMatrix(sizeReduction, image)
% find optimal order of removing raws and columns

    T = zeros(sizeReduction(1) + 1, sizeReduction(2) + 1, 'double');
    transBitMask = ones(size(T)) * -1;

    % fill in borders
    imageNoRow = image;
    for i = 2 : size(T, 1)
        energy = energyRGB(imageNoRow);
        [optSeamMask, seamEnergyRow] = findOptSeam(energy');
        imageNoRow = reduceImageByMask(imageNoRow, optSeamMask, 0);
        transBitMask(i, 1) = 0;

        T(i, 1) = T(i - 1, 1) + seamEnergyRow;
    end;

    imageNoColumn = image;
    for j = 2 : size(T, 2)
        energy = energyRGB(imageNoColumn);
        [optSeamMask, seamEnergyColumn] = findOptSeam(energy);
        imageNoColumn = reduceImageByMask(imageNoColumn, optSeamMask, 1);
        transBitMask(1, j) = 1;

        T(1, j) = T(1, j - 1) + seamEnergyColumn;
    end;

    % on the borders, just remove one column and one row before proceeding
    energy = energyRGB(image);
    [optSeamMask, seamEnergyRow] = findOptSeam(energy');
    image = reduceImageByMask(image, optSeamMask, 0);

    energy = energyRGB(image);
    [optSeamMask, seamEnergyColumn] = findOptSeam(energy);
    image = reduceImageByMask(image, optSeamMask, 1);

    % fill in internal part
    for i = 2 : size(T, 1)

        imageWithoutRow = image; % copy for deleting columns

        for j = 2 : size(T, 2)
            energy = energyRGB(imageWithoutRow);

            [optSeamMaskRow, seamEnergyRow] = findOptSeam(energy');
            imageNoRow = reduceImageByMask(imageWithoutRow, optSeamMaskRow, 0);

            [optSeamMaskColumn, seamEnergyColumn] = findOptSeam(energy);
            imageNoColumn = reduceImageByMask(imageWithoutRow, optSeamMaskColumn, 1);

            neighbors = [(T(i - 1, j) + seamEnergyRow) (T(i, j - 1) + seamEnergyColumn)];
            [val, ind] = min(neighbors);

            T(i, j) = val;
            transBitMask(i, j) = ind - 1;

            % move from left to right
            imageWithoutRow = imageNoColumn;
        end;

        energy = energyRGB(image);
        [optSeamMaskRow, seamEnergyRow] = findOptSeam(energy');
         % move from top to bottom
        image = reduceImageByMask(image, optSeamMaskRow, 0);
    end;

end
```
##Enlarging an image
In order to enlarge a picture, we compute k optimal seams for deleting but then, instead of deleting, copy average between neighbors:
```matlab
unction imageEnlarged = enlargeImageByMaskVertical(image, seamMask)

    avg = @(image, i, j, k) (image(i, j-1, k) + image(i, j+1, k))/2;

    imageEnlarged = zeros(size(image, 1), size(image, 2) + 1, size(image, 3));
    for i = 1 : size(seamMask, 1)
        j = find(seamMask(i, :) ~= 1);
        imageEnlarged(i, :, 1) = [image(i, 1:j, 1), avg(image, i, j, 1), image(i, j+1:end, 1)];
        imageEnlarged(i, :, 2) = [image(i, 1:j, 2), avg(image, i, j, 2), image(i, j+1:end, 2)];
        imageEnlarged(i, :, 3) = [image(i, 1:j, 3), avg(image, i, j, 3), image(i, j+1:end, 3)];
    end;
end

function imageEnlarged = enlargeImageByMaskHorizontal(image, seamMask)

    avg = @(image, i, j, k) (image(i-1, j, k) + image(i+1, j, k))/2;

    imageEnlarged = zeros(size(image, 1) + 1, size(image, 2), size(image, 3));
    for j = 1 : size(seamMask, 2)
        i = find(seamMask(:, j) ~= 1);
        imageEnlarged(:, j, 1) = [image(1:i, j, 1); avg(image, i, j, 1); image(i+1:end, j, 1)];
        imageEnlarged(:, j, 2) = [image(1:i, j, 2); avg(image, i, j, 2); image(i+1:end, j, 2)];
        imageEnlarged(:, j, 3) = [image(1:i, j, 3); avg(image, i, j, 3); image(i+1:end, j, 3)];
    end;
end
```
##Source code
The full code of the program is available [here](https://github.com/KirillLykov/cvision-algorithms/blob/master/seamCarving.m).
Seam carving is also implemented in [ImageMagick](http://www.imagemagick.org/Usage/resize/#liquid-rescale). So if you need a
C++ implementation, check out ImageMagick code.
