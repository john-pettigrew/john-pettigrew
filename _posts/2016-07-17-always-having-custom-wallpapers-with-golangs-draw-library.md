---
id: 9
title: Using Golang For Custom Wallpapers
date: 2016-07-17T20:51:42+00:00
author: johnpettigrew404
layout: post
guid: http://blog.pettigrew.rocks/?p=9
permalink: /2016/07/17/always-having-custom-wallpapers-with-golangs-draw-library/
categories:
  - golang
---
If you&#8217;re like me, then you like to constantly have new wallpapers. I often even have scripts to automatically switch my wallpapers. So naturally, I decided that I wanted to always have custom wallpapers with my name on them. And, what better way to achieve this than with Golang&#8217;s draw libraries? So here were my project goals:
  
1. Allow for custom text and icons.
  
2. Allow for easy scripting.
  
3. Allow different image sizes

The code for this post is available on <a href="https://github.com/john-pettigrew/wallpaper-customizer" target="_blank">my Github page</a>. I highly recommend the images from <a href="https://unsplash.com/" target="_blank">unsplash</a> for your wallpaper needs. (That is where I downloaded the wallpapers for this post!) Now, before we get started,

Here&#8217;s our mask.
  
 ![image mask](https://raw.githubusercontent.com/john-pettigrew/wallpaper-customizer/master/john_at_ava_large.png)

and here&#8217;s a sample output.
  
<img src="http://pettigrew.rocks/wp-content/uploads/2016/07/wallpaper2_min-1024x683.jpg" alt="Sample Image" width="840" height="560" class="alignnone size-large wp-image-16" srcset="https://pettigrew.rocks/wp-content/uploads/2016/07/wallpaper2_min-1024x683.jpg 1024w, https://pettigrew.rocks/wp-content/uploads/2016/07/wallpaper2_min-300x200.jpg 300w, https://pettigrew.rocks/wp-content/uploads/2016/07/wallpaper2_min-768x512.jpg 768w, https://pettigrew.rocks/wp-content/uploads/2016/07/wallpaper2_min-1200x800.jpg 1200w" sizes="(max-width: 709px) 85vw, (max-width: 909px) 67vw, (max-width: 1362px) 62vw, 840px" />
  
The go app takes the input image, flips it, inverts the colors, then draws this new image on the original in the area allowed by our mask. Let&#8217;s start by writing the code to read in our source images.

<pre class="lang:go decode:true " title="Read In Files" >dstPath := os.Args[1]
maskPath := os.Args[2]
outputPath := os.Args[3]

//Read in images
dst, err := readImage(dstPath)
if err != nil {
	log.Fatal("Error reading dst")
}

mask, err := readImage(maskPath)
if err != nil {
	log.Fatal("Error reading src")
}

...

func readImage(file string) (image.Image, error) {
	imageFile, err := os.Open(file)
	if err != nil {
		return nil, err
	}

	image, _, err := image.Decode(imageFile)
	if err != nil {
		return nil, err
	}

	return image, nil
}
</pre>

Here, we are using our command line arguments as file paths and passing them to our &#8220;readImage&#8221; function. This function simply reads in the file data and converts it to an Image.

Next, we need to scale our mask so that it will fit when our wallpaper is larger or smaller. We will use our &#8220;dst&#8221; image&#8217;s size.

<pre class="lang:go decode:true " title="Scale Mask" >//Scale mask
finalMask := image.NewRGBA(dst.Bounds())
draw.ApproxBiLinear.Scale(finalMask, dst.Bounds(), mask, mask.Bounds(), draw.Over, nil)</pre>

Now we can actually transform our image. I decided to move these image functions to a different go package. Let&#8217;s start by flipping our &#8220;dst&#8221; image.

<pre class="lang:default decode:true " >changedDst := imgtransforms.Flip(dst)

...

//Flip returns a copy of input that has been flipped horizontally and vertically.
func Flip(input image.Image) image.Image {

	//create new image
	bounds := input.Bounds()
	newImg := image.NewRGBA(bounds)
	for x := 0; x &lt; bounds.Max.X; x++ {
		for y := 0; y &lt; bounds.Max.Y; y++ {
			newImg.Set(bounds.Max.X-x, bounds.Max.Y-y, input.At(x, y))
		}
	}

	return newImg
}</pre>

This function simply switches the pixels horizontally and vertically. 

Next, let&#8217;s invert the colors in our image. 

<pre class="lang:go decode:true " title="Invert Colors" >changedDst = imgtransforms.InvertColors(changedDst)

...

type pixelColor struct {
	r, g, b, a uint32
}

//InvertColors returns a copy of input that has its colors inverted.
func InvertColors(input image.Image) image.Image {

	//create new image
	bounds := input.Bounds()
	newImg := image.NewRGBA(bounds)

	var currentPixelColor color.Color
	var r, g, b, a uint32
	for x := 0; x &lt; bounds.Max.X; x++ {
		for y := 0; y &lt; bounds.Max.Y; y++ {
			r, g, b, a = input.At(x, y).RGBA()
			currentPixelColor = pixelColor{
				r: 0xffff - r,
				g: 0xffff - g,
				b: 0xffff - b,
				a: a,
			}
			newImg.Set(x, y, currentPixelColor)
		}
	}

	return newImg
}</pre>

Here we make a struct to represent a pixel&#8217;s colors. We then simply make a new image and set each pixel to the corresponding pixel from our input with inverse RGB values.

Now that our image functions are done, we can continue. One small thing we need to take care of is converting our &#8220;dst&#8221; object from an &#8220;image.Image&#8221; to an &#8220;image.RGBA&#8221;. We do this by creating a new RGBA object and drawing our image. 

<pre class="lang:go decode:true " title="Image Convert" >//Convert dst
dstB := dst.Bounds()
finalDst := image.NewRGBA(dstB)
draw.Draw(finalDst, finalDst.Bounds(), dst, dstB.Min, draw.Src)</pre>

Finally we can actually draw our image.

<pre class="lang:go decode:true " title="Image Draw" >//Draw our image
draw.DrawMask(finalDst, finalDst.Bounds(), changedDst, image.ZP, finalMask, image.ZP, draw.Over)</pre>

We pass our converted dst image and it&#8217;s bounds, our modified image, the image zero point, our mask, the image zero point (again), and finally the &#8220;Over&#8221; operation. This call does the actual work of drawing our modified image over our original through the mask we provided. 

All that&#8217;s left to do is to save our final image.

<pre class="lang:go decode:true " title="Save Image" >//Create output file
output, err := os.Create(outputPath)
if err != nil {
	log.Fatal(err)
}

//Save output file
options := jpeg.Options{Quality: 100}
err = jpeg.Encode(output, finalDst, &options)
if err != nil {
	log.Fatal(err)
}</pre>

And we should now have a fancy wallpaper to use. Here are some more sample outputs.

<img src="http://pettigrew.rocks/wp-content/uploads/2016/07/bridge-1024x659.jpg" alt="bridge" width="840" height="541" class="alignnone size-large wp-image-38" srcset="https://pettigrew.rocks/wp-content/uploads/2016/07/bridge-1024x659.jpg 1024w, https://pettigrew.rocks/wp-content/uploads/2016/07/bridge-300x193.jpg 300w, https://pettigrew.rocks/wp-content/uploads/2016/07/bridge-768x494.jpg 768w, https://pettigrew.rocks/wp-content/uploads/2016/07/bridge-1200x772.jpg 1200w, https://pettigrew.rocks/wp-content/uploads/2016/07/bridge.jpg 1400w" sizes="(max-width: 709px) 85vw, (max-width: 909px) 67vw, (max-width: 1362px) 62vw, 840px" />

<img src="http://pettigrew.rocks/wp-content/uploads/2016/07/bridge2-1024x683.jpg" alt="bridge2" width="840" height="560" class="alignnone size-large wp-image-39" srcset="https://pettigrew.rocks/wp-content/uploads/2016/07/bridge2-1024x683.jpg 1024w, https://pettigrew.rocks/wp-content/uploads/2016/07/bridge2-300x200.jpg 300w, https://pettigrew.rocks/wp-content/uploads/2016/07/bridge2-768x512.jpg 768w, https://pettigrew.rocks/wp-content/uploads/2016/07/bridge2-1200x800.jpg 1200w, https://pettigrew.rocks/wp-content/uploads/2016/07/bridge2.jpg 1600w" sizes="(max-width: 709px) 85vw, (max-width: 909px) 67vw, (max-width: 1362px) 62vw, 840px" />

<img src="http://pettigrew.rocks/wp-content/uploads/2016/07/hill-1024x769.jpg" alt="hill" width="840" height="631" class="alignnone size-large wp-image-40" srcset="https://pettigrew.rocks/wp-content/uploads/2016/07/hill-1024x769.jpg 1024w, https://pettigrew.rocks/wp-content/uploads/2016/07/hill-300x225.jpg 300w, https://pettigrew.rocks/wp-content/uploads/2016/07/hill-768x577.jpg 768w, https://pettigrew.rocks/wp-content/uploads/2016/07/hill-1200x901.jpg 1200w" sizes="(max-width: 709px) 85vw, (max-width: 909px) 67vw, (max-width: 1362px) 62vw, 840px" />

<img src="http://pettigrew.rocks/wp-content/uploads/2016/07/jump-1024x683.jpg" alt="jump" width="840" height="560" class="alignnone size-large wp-image-41" srcset="https://pettigrew.rocks/wp-content/uploads/2016/07/jump-1024x683.jpg 1024w, https://pettigrew.rocks/wp-content/uploads/2016/07/jump-300x200.jpg 300w, https://pettigrew.rocks/wp-content/uploads/2016/07/jump-768x512.jpg 768w, https://pettigrew.rocks/wp-content/uploads/2016/07/jump-1200x800.jpg 1200w" sizes="(max-width: 709px) 85vw, (max-width: 909px) 67vw, (max-width: 1362px) 62vw, 840px" />

<img src="http://pettigrew.rocks/wp-content/uploads/2016/07/trees-1024x683.jpg" alt="trees" width="840" height="560" class="alignnone size-large wp-image-42" srcset="https://pettigrew.rocks/wp-content/uploads/2016/07/trees-1024x683.jpg 1024w, https://pettigrew.rocks/wp-content/uploads/2016/07/trees-300x200.jpg 300w, https://pettigrew.rocks/wp-content/uploads/2016/07/trees-768x512.jpg 768w, https://pettigrew.rocks/wp-content/uploads/2016/07/trees-1200x801.jpg 1200w" sizes="(max-width: 709px) 85vw, (max-width: 909px) 67vw, (max-width: 1362px) 62vw, 840px" />

I hope this post was a good introduction into Golang&#8217;s image and draw libraries. I went over just a couple of possible transforms one could apply to an image. A challenge to the reader is to add different image manipulations to create even cooler output images. Also, remember that the code for this post can be found on <a href="https://github.com/john-pettigrew/wallpaper-customizer" target="_blank">my Github page</a>.

Thanks for reading,

John