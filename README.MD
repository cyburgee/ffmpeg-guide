# INTRODUCTION

If you've worked with media encoding in the past decade it's likely that you've come across [FFmpeg](https://ffmpeg.org). For those of you who are unfamiliar, in their words: 

> FFmpeg is the leading multimedia framework, able to decode, encode, transcode, mux, demux, stream, filter and play pretty much anything that humans and machines have created. It supports the most obscure ancient formats up to the cutting edge. 

It's packed with an enormous amount of functionality. It can be a daunting for beginners. It can be baffling for experts. Maybe you can read the [documentation](https://ffmpeg.org/ffmpeg.html) and make some sense of it, or maybe you feel the same way about reading words as me.

![](inline/no-books.gif)

*JK - the FFmpeg docs are great*

In that light, I wrote this post to share and explain some of it's functionality, especially as it relates to GIF transcoding. To follow along you'll need FFmpeg installed. The easiest way to do that is to go [here](https://www.ffmpeg.org/download.html) and find a static build for whatever platform you're working on.


# GIF TO VIDEO

Let's start with a simple example. Say you have a gif of my ancient, computer illiterate pug, Benji, and you want to convert it to a video format. Here's one example:
```bash
$ ffmpeg -i benji.gif -f mp4 -pix_fmt yuv420p benji.mp4
```
Let's break down this command and its constituent arguments.
```
ffmpeg
```
This launches the FFmpeg executable. I'm assuming here that your shell knows the complete path to it.
```
-i benji.gif
```
This flag indicates our input. I assume that the `i` stands for "input". I'm also assuming that `benji.gif` is a gif of my stubbornly smelly pug and that it is in your shell's current working directory.

```
-f mp4
```
This is optional in most circumstances. This tells FFmpeg that we want to output to an `mp4` media container. FFmpeg will typically infer that from the extension supplied in the output file pattern, but when it comes to working with FFmpeg it doesn't hurt to be specific.

```
-pix_fmt yuv420p
```
This is another optional flag, but I use it because it makes the file play nicely in the QuickTime Player, and subsequently Finder, on my Mac.

```
benji.mp4
```
This last argument is our output. It'll create the file if it doesn't exist and if it already exists, you will be prompted on the command line to confirm if you'd like to overwrite the existing file. If you think you know better and are immune to pain and regret then you can add the ` -y ` flag somewhere in your command and it will automatically overwrite the file if it exists.

![](inline/burning.gif)

*Some people just want to see the world burn*

While the above generally works with most content, you may be interested to learn that GIFs support widely varying frame rates or frame delays that when converted to a video format without transformation, can trip up certain video players.
If I were to use the previous command to convert my contrived and not very useful as a countdown example, `321.gif`, the VLC player on my Mac has trouble playing that mp4. Don't worry, I still love you VLC. If your particular application might be sensitive to such issues, you can try the following command:
```bash
$ ffmpeg -i 321.gif -filter_complex "[0:v] fps=15" -vsync 0 -f mp4 -pix_fmt yuv420p 321.mp4
```
Some of the above arguments look pretty familiar, right? Let's go over what's changed.
```
-filter_complex "[0:v] fps=15"
```
This here is how you specify a filter graph. Filters process raw video or audio based on the name of the filter and the arguments specified. You can read more [here](https://ffmpeg.org/ffmpeg-filters.html#Filtergraph-description). This is a very simple example of a filter, but if you read on you can see things get a bit more complex. Let's decompose this one for now:
```
[0:v]
```
This specifies that we will be using the first video stream fed to FFmpeg as input to this section of the filter. If you had a file with multiple video streams, or you put two inputs in the command you could imagine selecting the second video stream with `[1:v]`, or the third audio stream as `[2:a]`, etc, etc.
```
fps=15
```
This is the [fps](https://ffmpeg.org/ffmpeg-filters.html#fps-1) or "frames per second" or "framerate" filter. If you don't want to lose any visual content, I suggest you set the value to the nominal framerate of the input, so the inverse of the minimum frame delay.

```
-vsync 0
```
This [vsync](https://ffmpeg.org/ffmpeg.html#toc-Advanced-options) flag tells ffmpeg to not modify the timing of the frames as they come from the decoder. This should keep your weird gif timing intact.

That about does it for GIFs to video formats. Here's a [handy guide](https://trac.ffmpeg.org/wiki/Encode/H.264) for encoding H.264 video should you be interested in the ins and outs of that particular format.


# VIDEO TO GIF

Note that these methods don't play nicely with sources with transparency/alpha *yet* - hopefully more on that in a future blog post.

Seeing the simplicity of the above examples, you might be tempted to create a gif with FFmpeg like so:
```bash
$ ffmpeg -ss 61.0 -t 2.5 -i StickAround.mp4 -f gif StickAround.gif
```
But I wouldn't recommend it. File size issues aside, the quality probably just isn't where you want it to be. The issue seems to be that FFmpeg's native gif encoder doesn't take into account the colors in the source image, and as an indexed color format, that just won't do for a GIF.

Let's look at those new parameters though.
```
-ss 61.0
```
The [-ss](https://trac.ffmpeg.org/wiki/Seeking) option tells FFmpeg to seek to `61.0` seconds into the input file. Two "s"s indicate a fast seek but with recent FFmpeg it's both fast AND accurate.
```
-t 2.5
```
The [-t](https://ffmpeg.org/ffmpeg.html#Main-options) option signifies that we only want to read in `2.5` seconds of the input video and then stop. 

So with these two arguments, FFmpeg will seek to second `61.0` and read in the next `2.5` seconds. The placement of these arguments are significant. Since they are in the command before the input `-i StickAround.mp4` they apply to the reading of the input file. The values of the above two argument are totally dependent on what part of the video you want. Feel free to play around with them. 


Thanks to the work of some generous FFmpeg developers, namely [ubitux](http://blog.pkh.me/p/21-high-quality-gif-with-ffmpeg.html), you can now generate a color palette to be used to generate a much higher quality GIF like so:
```bash
$ ffmpeg -ss 61.0 -t 2.5 -i StickAround.mp4 -filter_complex "[0:v] palettegen" palette.png
$ ffmpeg -ss 61.0 -t 2.5 -i StickAround.mp4 -i palette.png -filter_complex "[0:v][1:v] paletteuse" prettyStickAround.gif
```
Let's look at these exciting new arguments.
```
-filter_complex "[0:v] palettegen"
```
[palettegen](https://ffmpeg.org/ffmpeg-filters.html#palettegen-1) is a filter that generates a 256 color palette to be used in GIF encoding in the next step. Like the filter above, it uses the first video stream of the input, indicated by `[0:v]`.
```
palette.png
```
This is our output file. Easy!

Now what about the following command? Let's look at the input arguments. We now have:
```
-i StickAround.mp4 -i palette.png
```
We have two input files this time. The original video, and the palette, `palette.png` we created in the command just before. You can see how we use these in the filter arguments that come next:
```
-filter_complex "[0:v][1:v] paletteuse"
```
So, this is new. The [paletteuse](https://ffmpeg.org/ffmpeg-filters.html#paletteuse) filter takes in two arguments, the first being the video content and the second being the color palette that gets applied to output a nice looking GIF. Sweet! You can see how two inputs are specified in a FFmpeg filter graph:
```
[0:v][1:v]
```
which correspond to 
```
-i StickAround.mp4 -i palette.png
```

Say you don't feel like managing an intermediate palette file. It's critical that you remember the exact timing of where you cut, you have to remember an extra filename, you have to not mess up the spelling of `ffmpeg` twice - it's all too much!

![](inline/no-time.gif)

So, let's get weird with that filter graph. Try this out:
```bash
$ ffmpeg -ss 61.0 -t 2.5 -i StickAround.mp4 -filter_complex "[0:v] split [a][b];[a] palettegen [p];[b][p] paletteuse" FancyStickAround.gif
```
About that filtergraph argument - this is the first one that lives up to the "complex" label.
```
-filter_complex "[0:v] split [a][b];[a] palettegen [p];[b][p] paletteuse"
```
First we've got the [split](https://ffmpeg.org/ffmpeg-filters.html#split_002c-asplit) filter.
```
[0:v] split [a][b]
```
[split](https://ffmpeg.org/ffmpeg-filters.html#split_002c-asplit) takes the first video as input and creates two outputs from the one input, as you might have suspected. I've labeled them `[a]` and `[b]` but you can call them `[dog]` and `[cat]` for all I care.
Next up:
```
;[a] palettegen [p]
```
The semicolon indicates that we're specifying a new filter, and it's the [palettegen](https://ffmpeg.org/ffmpeg-filters.html#palettegen-1) filter we all know and love. As you can see the input to this one is `[a]` that we defined as the output of the [split](https://ffmpeg.org/ffmpeg-filters.html#split_002c-asplit) filter before this one. The output of this filter is `[p]` which stands for "pug" or "palette" - whichever one you want to believe.

![](inline/pug-swarm.gif)
*Approximately 80% of my thoughts are pug-related*

Next our filtergraph has:
```
;[b][p] paletteuse"
```
We know this one too! The only twist here is that it uses the second output of our [split](https://ffmpeg.org/ffmpeg-filters.html#split_002c-asplit) filter, `[b]` as the first input and output of our [palettegen](https://ffmpeg.org/ffmpeg-filters.html#palettegen-1) filter `[p]`. Order is important here folks!

There's a drawback to doing it this way is that it can use more memory all at once, but overall you will probably save on memory usage, especially if you adhere to the suggestions that follow in the upcoming command.

So that was nice, one-liner FFmpeg command to convert video to high quality GIF clip. But there's still a problem. Our GIF is still too big. Unfortunately GIFs aren't well suited to being 1080p even if they are 2.5 seconds long. Let's fix that with this next command:
```bash
$ ffmpeg -ss 61.0 -t 2.5 -i StickAround.mp4 -filter_complex "[0:v] fps=12,scale=480:-1,split [a][b];[a] palettegen [p];[b][p] paletteuse" SmallerStickAround.gif
```
Most of the above is pretty familiar. We just have a couple of new additions to the filter graph.
```
-filter_complex "[0:v] fps=12,scale=w=480:h=-1,split [a][b];[a] palettegen [p];[b][p] paletteuse"
```
Let's look at that first stage.
```
[0:v] fps=12,scale=480:-1,split [a][b];
```
Our [split](https://ffmpeg.org/ffmpeg-filters.html#split_002c-asplit) filter from before has a couple of new friends - [fps](https://ffmpeg.org/ffmpeg-filters.html#fps-1) and [scale](https://ffmpeg.org/ffmpeg-filters.html#scale-1). I've separated them with commas, which you can do with filters that accept a single input and output. 
We've seen [fps](https://ffmpeg.org/ffmpeg-filters.html#fps-1) before. It changes the frame rate. In this case it took a 24 fps input video and halved the framerate to `12` as you can see. I think 12 fps is high enough for this content and gets our filesize down, but decide for yourself. 
The next filter in line here is scale](https://ffmpeg.org/ffmpeg-filters.html#scale-1), which resizes its input video. The scale](https://ffmpeg.org/ffmpeg-filters.html#scale-1) filter needs two parameters, width and height, and they can look a little odd. To complicate matters they can take several forms.
Here's how I've written it:
```
scale=w=480:h=-1
```
I could also be more explicit:

```
scale=width=480:height=-1
```
How about a third option if that wasn't confusing enough:

```
scale=480:-1
```
Any way you write it, I've specified that I want to scale the input video to width 480 pixels and I want the height to be -1 pixels. Nah - that ain't right. `-1` has a special significance to the scale](https://ffmpeg.org/ffmpeg-filters.html#scale-1) filter. It means I want the filter to maintain the aspect ratio or proportions of the video while resizing the other dimension I specified. 

##### "Pro" Tip 1.
`-2` has a similar meaning to `-1` in the scale filter. It means to stay as close to the aspect ratio but round to an even number. This is important for many video codecs which require even dimensions, such as H.264. It doesn't matter for GIFs.

##### "Pro" Tip 2:
If you're changing the frame rate and resizing/scaling your video, resize before increasing the framerate or resize after decreasing the framerate. Your machine won't have to resize as many frames - saving you time and processing power.

Now we should have a lovely GIF that is a manageable size. But what if for some reason you didn't think it was pretty enough? After all, that whole GIF only has 256 colors across all of its frames. Do you know where I'm going with this? 

![](inline/no-idea.gif)

*I have no idea what I'm doing*

What if we could create a GIF with FFmpeg that has an insane 256 colors per frame? This is a pretty unique application that's only really necessary if you've got some extremely varied colors, a long gif with multiple scenes, or both. Imma show you how to do it anyways.
```bash
$ ffmpeg -ss 61.0 -t 2.5 -i StickAround.mp4 -filter_complex "[0:v] fps=12,scale=w=480:h=-1,split [a][b];[a] palettegen=stats_mode=single [p];[b][p] paletteuse=new=1" StickAroundPerFrame.gif
```
Not very different right? This just leverages some of the options of the [palettegen](https://ffmpeg.org/ffmpeg-filters.html#palettegen-1) and `paletteuse` filters. Let's take a look.
```
[a] palettegen=stats_mode=single [p]
```
We're now specifying the `stats_mode` parameter of the filter. The argument `single` tells the filter to generate a new palette for every input frame.
Next we specify a corresponding parameter for [paletteuse](https://ffmpeg.org/ffmpeg-filters.html#paletteuse)
```
[b][p] paletteuse=new=1
```
The `new` parameter for [paletteuse](https://ffmpeg.org/ffmpeg-filters.html#paletteuse) indicates to the filter that it should grab a new palette for each frame, working in beautiful harmony with the `palettegen=stats_mode=single` filter stage.

The drawback of doing a per frame palette is that it can increase the file size. But it could also reduce the file size. GIF optimization is weird. But that would have to be the subject of another blog post. Let us know if you want that one!

I hope this has been a useful breakdown of FFmpeg and some GIF-related commands. 
Happy GIFfing folks!

![](inline/success.gif)
