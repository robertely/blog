---
author: Robert Ely
date: 2015-03-03 02:51:52+00:00
draft: false
title: Let's add a dirt cheap screen to the Raspberry Pi B+
type: post
cover: http://reasonablycorrect.com/wp-content/uploads/2015/03/2015-02-22-13.07.35-1024x760.jpg
url: /raw-dpi-raspberry-pi/
categories:
- Raspberry Pi
- Displays
- Hardware
---

[![Gunstar Heros over DPI]

Recently [the ](http://www.raspberrypi.org/forums/viewtopic.php?f=100&t=86658)[internet](https://www.youtube.com/watch?v=DTuvzxkVaKs&list=PL1A011279DBD4EB7E&index=3) noticed the Raspberry Pi could drive LCD panels using DPI. This allows very inexpensive displays to be used with basically no additional hardware.  In this post we dive into the hardware required, the software configuration, how to read screen datasheets, and basic troubleshooting.

A quick word of warning, this is an advanced project. The rewards are significant but it's difficult to get right.

There are no specific prerequisite skills necessary, but a reasonable understanding of Linux and digital signals is extremely helpful.

<!-- more -->


## Disclaimer and thanks


I am not an electrical engineer. This post was my first adventure into display technology. I managed to cobble it together from the wonderful posts on the Raspberry Pi forum. Specifically this post: [http://www.raspberrypi.org/forums/viewtopic.php?f=100&t=86658](http://www.raspberrypi.org/forums/viewtopic.php?f=100&t=86658). It's great reading on it's own, and most of the information here was derived from it.

Special thanks is due to Gert van Loo @ [Fenlogic ](http://fenlogic.com/)for this project: [https://github.com/fenlogic/vga666](https://github.com/fenlogic/vga666) and Dom Cobley @ Broadcom for maintaining the VideoCore firmware, and posting config examples on the Raspberry Pi forum.

Thanks to forum users "Fat D", and "ceteras" for pointing me in the right direction and generally being awesome.

Additional thanks is due to [Shannon Geis](http://shannongeis.net/) for editing this post, and [Lady Ada at Adafruit Industries](https://www.adafruit.com/about) who posted the [video](https://www.youtube.com/watch?v=DTuvzxkVaKs&list=PL1A011279DBD4EB7E&index=3) that brought this to my attention.


## DPI is awesome


The main reason is because it's wonderfully stupid.

DPI stands for Display Parallel Interface (or possibly Display Pixel Interface depending on who you ask). It allows you to use very cheap displays by driving them manually. The Raspberry Pi supports this but it may not be right for every project.


#### Pros:





	  * Very fast, easily driving our display at 60hz.
	  * No complicated interface hardware.
	  * Pixel perfect output. Digital, not analog.
	  * Easy to understand protocol.
	  * No bulky connectors.
	  * Very inexpensive.



#### Cons:





	  * Eats a lot of GPIO pins (All of them at _true color_, but more on this later.)
	  * Not widely adopted on the Raspberry Pi, making online help hard to find.
	  * Ribbon cables are easy to break if you're not careful.
	  * Short range. If you want your screen 10 meters from the Pi use HDMI.
	  * There will be maths, this isn't plug-n-play.
	  * Almost zero official documentation.



## Bill of materials





	  * [40-pin TFT Friend](https://www.adafruit.com/products/1932)
	  * [5.0" 40-pin 800x480 TFT Display without Touchscreen](https://www.adafruit.com/products/1680)
	  * [Premium Female/Female Jumper Wires - 40 x 6"](https://www.adafruit.com/products/266) (Optional, but great for prototyping.)

Total cost at time of writing is** less than $50**, putting this into impulse buy/weekend project territory. You can also use the 7-inch screen Adafruit sells or any 40pin DPI screen, but some of the numbers later are probably not going to jive with it. You will have to find your own way there. Similarly, I'm not using touch screens because I've never seen a touch interface that wasn't terrible, and it lowers the cost a bit. If you want to add touch you're going to need a touch controller, but that's another blog post.


## A quick note on breadboards...




I strongly recommend **not** using a bread board for this project. Individual pixels are written at speeds in excess of 23MHz for the example screen and possibly faster for larger panels. Due to [breadboard capacitance ](https://www.youtube.com/watch?v=6GIscUsnlM0)at these frequencies, cross talk between the parallel color bus is significant and very likely to cause problems. For testing make connections as direct as possible.





## Tools




You're probably going to need some diagnostic tools. A multimeter is always great to have around to verify connections, but if you run into any real trouble you're going to need a way to visualize the display data. I made due with a [cheap USB logic analyzer](https://www.saleae.com/logic/), but an oscilloscope with 4 channels or more would have made my life a lot easier. It's not impossible to pull this off with out these tools, but you're going to be doing a lot of guess work and cursing.





## Color depth


The Raspberry Pi B+ has 28 GPIO Pins. We will be using them to drive the TFT directly. If we wanted to drive full 24 bit _true color_ we would need 8 pins each for red, green, and blue as well as pins for hsync, vsync, clock, and display enable. This is a grand total of...28 pins. Great news if you don't need GPIO for any other purposes, but I do. The way around this is to simply omit the two least-significant-bits on each color bus. Giving us 18 bit _high color_. Obviously we have lost some data but it's not as bad as one would expect. I'm using this display for console emulation, and [most older consoles are running with far less than 18 bit color](http://en.wikipedia.org/wiki/List_of_video_game_console_palettes).


## Physical connections


[![Pinout](http://reasonablycorrect.com/wp-content/uploads/2015/03/Pinout.png)
](http://reasonablycorrect.com/wp-content/uploads/2015/03/Pinout.png)

(The original Google Sheet is [available here.](https://docs.google.com/spreadsheets/d/1KtH3ogHTpWotTmeRRp8acbK6KBJvc7CvKB-R7MSeMU0/edit?usp=sharing))

You will notice we need the seldom use pins 27 and 28. These pins are used to detect "Pi hats", since we need these pins for Clock and DEN, you won't be able to piggy back a hat on top of this, and converting this to one will be difficult/possibly impossible. Again, because we are using 18 bit color we have gained back 6 GPIO pins, specifically 22-27. If you want to experiment with other configurations  with more or less color check out [this post on the Raspberry Pi forum](http://www.raspberrypi.org/forums/viewtopic.php?p=622498#p622498) and update your device tree accordingly.


#### Notes on back light drivers


I'm not covering back light drivers here. Mostly because the TFT friend has one built in. It's actually the only part of the board that isn't simply a pass through.  It uses a 24 volt boost converter that can power a 7 LED backlight with selectable current limits. I *think* it also supports dimming via the PWM line but I haven't tested it yet. Your mileage may vary.


## Device tree


Device trees allow you to reconfigure peripherals on the Pi. Here we use it to enable the DPI interface. It's a complicated subject and it's been [well documented by the Raspberry Pi foundation](http://www.raspberrypi.org/documentation/configuration/device-tree.md#part3). So i'm not going to rewrite the details here. The [denlogic/vga666 repo](https://github.com/fenlogic/vga666/) has a copy of the[ device tree used on the VGA666 board. ](https://github.com/fenlogic/vga666/blob/master/setup/dt-blob-dpi.dts)I've trimmed it down to only the B+, and added comments.

You can find it here: [https://github.com/robertely/dpi666/blob/master/setup/dt-blob-dpi.dts](https://github.com/robertely/dpi666/blob/master/setup/dt-blob-dpi.dts)

If you want to use more or less color pins, or are using it on a different board, fork and modify this file accordingly.

This config needs to be compiled into a blob and placed on the /boot partition. This is easily done with utilities on the Pi, and are in the raspbian repos.

    
    $ cd ~



    
    $ sudo apt-get install device-tree-compiler



    
    $ wget https://raw.githubusercontent.com/robertely/dpi666/master/setup/dt-blob-dpi.dts



    
    $ sudo dtc -I dts -O dtb -o /boot/dt-blob.bin dt-blob-dpi.dts




## The data sheet and you


Time to get to know your LCD. Take your time here, there are a lot of details.

If your using the 5-inch Adafruit TFT you may be able to skip this, but you should at least skim it for troubleshooting.

Data sheets _should_ provide you with every thing you need to get hooked up. Unfortunately most of them are written like a bad Chinese/English Google translation dictated over a Walkie Talkie strapped to a wood chipper. There will be mistakes, omissions, nonsense, and--unless you are an electrical engineer--technical details that are way over your head.

Let's skip ahead a little and look at some of the configurable variables we will need in the config.txt.


#### dpi_output_format:


<table style="height: 470px;" border="1" width="700" cellpadding="0" cellspacing="0" dir="ltr" > 
<tbody >
<tr >

<td data-sheets-value="[null,2,"Name"]" >Name
</td>

<td data-sheets-value="[null,2,"Description"]" >Description
</td>

<td data-sheets-value="[null,2,"Unit"]" >Unit
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"output_format"]" >output_format
</td>

<td data-sheets-value="[null,2,"Complex, See below"]" >Complex, see "Additional settings"
</td>

<td data-sheets-value="[null,2,"Binary(1-4, 4 bits)"]" >Binary(1-7, 4 bits)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"rgb_order"]" >rgb_order
</td>

<td data-sheets-value="[null,2,"Complex, See below"]" >Complex, see "Additional settings"
</td>

<td data-sheets-value="[null,2,"Binary(1-7, 4 bits)"]" >Binary(1-4, 4 bits)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"output_enable_mode"]" >output_enable_mode
</td>

<td data-sheets-value="[null,2,"0: Enable data valid / 1: Combined sync"]" >0: Enable data valid / 1: Combined sync
</td>

<td data-sheets-value="[null,2,"Binary (1/0)"]" >Binary (1/0)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"invert_pixel_clock"]" >invert_pixel_clock
</td>

<td data-sheets-value="[null,2,"Invert (0:Stable high / 1:Stable low)"]" >Invert (0:Stable high / 1:Stable low)
</td>

<td data-sheets-value="[null,2,"Binary (1/0)"]" >Binary (1/0)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"hsync_disable"]" >hsync_disable
</td>

<td data-sheets-value="[null,2,"Disable hsync"]" >Disable hsync
</td>

<td data-sheets-value="[null,2,"Binary (1/0)"]" >Binary (1/0)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"vsync_disable"]" >vsync_disable
</td>

<td data-sheets-value="[null,2,"Disable vsync"]" >Disable vsync
</td>

<td data-sheets-value="[null,2,"Binary (1/0)"]" >Binary (1/0)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"output_enable_disable"]" >output_enable_disable
</td>

<td data-sheets-value="[null,2,"Disable DEN pin"]" >Disable DEN pin
</td>

<td data-sheets-value="[null,2,"Binary (1/0)"]" >Binary (1/0)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"hsync_polarity"]" >hsync_polarity
</td>

<td data-sheets-value="[null,2,"Invert (0:Normal / 1:Inverted )"]" >Invert (0:Normal / 1:Inverted )
</td>

<td data-sheets-value="[null,2,"Binary (1/0)"]" >Binary (1/0)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"vsync_polarity"]" >vsync_polarity
</td>

<td data-sheets-value="[null,2,"Invert (0:Normal / 1:Inverted )"]" >Invert (0:Normal / 1:Inverted )
</td>

<td data-sheets-value="[null,2,"Binary (1/0)"]" >Binary (1/0)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"output_enable_polarity"]" >output_enable_polarity
</td>

<td data-sheets-value="[null,2,"Invert (0:Normal / 1:Inverted )"]" >Invert (0:Normal / 1:Inverted )
</td>

<td data-sheets-value="[null,2,"Binary (1/0)"]" >Binary (1/0)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"hsync_phase"]" >hsync_phase
</td>

<td data-sheets-value="[null,2,"0: Rising / 1: Falling edge triggering"]" >0: Rising / 1: Falling edge triggering
</td>

<td data-sheets-value="[null,2,"Binary (1/0)"]" >Binary (1/0)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"vsync_phase"]" >vsync_phase
</td>

<td data-sheets-value="[null,2,"0: Rising / 1: Falling edge triggering"]" >0: Rising / 1: Falling edge triggering
</td>

<td data-sheets-value="[null,2,"Binary (1/0)"]" >Binary (1/0)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"output_enable_phase"]" >output_enable_phase
</td>

<td data-sheets-value="[null,2,"0: Rising / 1: Falling edge triggering"]" >0: Rising / 1: Falling edge triggering
</td>

<td data-sheets-value="[null,2,"Binary (1/0)"]" >Binary (1/0)
</td>
</tr>
</tbody>
</table>


#### hdmi_timings:


<table style="height: 556px;" border="1" width="696" cellpadding="0" cellspacing="0" dir="ltr" > 
<tbody >
<tr >

<td data-sheets-value="[null,2,"Name"]" >Name
</td>

<td data-sheets-value="[null,2,"Description"]" >Description
</td>

<td data-sheets-value="[null,2,"Unit"]" >Unit
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"h_active_pixels"]" >h_active_pixels
</td>

<td data-sheets-value="[null,2,"Horizontal Length (Width)"]" >Horizontal Length (Width)
</td>

<td data-sheets-value="[null,2,"Pixels"]" >Pixels
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"h_sync_polarity"]" >h_sync_polarity
</td>

<td data-sheets-value="[null,2,"Invert (Active heigh/active low)"]" >Invert (Active heigh/active low)
</td>

<td data-sheets-value="[null,2,"Binary (1/0)"]" >Binary (1/0)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"h_front_porch"]" >h_front_porch
</td>

<td data-sheets-value="[null,2,"Horizontal forward padding from DEN "]" >Horizontal forward padding from DEN
</td>

<td data-sheets-value="[null,2,"Clock Cycles"]" >Clock Cycles
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"h_sync_pulse"]" >h_sync_pulse
</td>

<td data-sheets-value="[null,2,"Hsync pulse width"]" >Hsync pulse width
</td>

<td data-sheets-value="[null,2,"Clock Cycles"]" >Clock Cycles
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"h_back_porch"]" >h_back_porch
</td>

<td data-sheets-value="[null,2,"Vertical back padding from DEN "]" >Vertical back padding from DEN
</td>

<td data-sheets-value="[null,2,"Clock Cycles"]" >Clock Cycles
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"v_active_lines"]" >v_active_lines
</td>

<td data-sheets-value="[null,2,"Vertical Length (Height)"]" >Vertical Length (Height)
</td>

<td data-sheets-value="[null,2,"Pixels"]" >Pixels
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"v_sync_polarity"]" >v_sync_polarity
</td>

<td data-sheets-value="[null,2,"Invert (Active high/Active low)"]" >Invert (Active high/Active low)
</td>

<td data-sheets-value="[null,2,"Binary (1/0)"]" >Binary (1/0)
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"v_front_porch"]" >v_front_porch
</td>

<td data-sheets-value="[null,2,"Vertical forward padding from DEN "]" >Vertical forward padding from DEN
</td>

<td data-sheets-value="[null,2,"Clock Cycles"]" >Clock Cycles
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"v_sync_pulse"]" >v_sync_pulse
</td>

<td data-sheets-value="[null,2,"Vsync pulse width"]" >Vsync pulse width
</td>

<td data-sheets-value="[null,2,"Clock Cycles"]" >Clock Cycles
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"v_back_porch"]" >v_back_porch
</td>

<td data-sheets-value="[null,2,"Vertical back padding from DEN "]" >Vertical back padding from DEN
</td>

<td data-sheets-value="[null,2,"Clock Cycles"]" >Clock Cycles
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"v_sync_offset_a"]" >v_sync_offset_a
</td>

<td data-sheets-value="[null,2,"??"]" >Unknown, left at zero
</td>

<td >
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"v_sync_offset_b"]" >v_sync_offset_b
</td>

<td data-sheets-value="[null,2,"??"]" >Unknown, left at zero
</td>

<td >
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"pixel_rep"]" >pixel_rep
</td>

<td data-sheets-value="[null,2,"??"]" >Unknown, left at zero
</td>

<td >
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"frame_rate"]" >frame_rate
</td>

<td data-sheets-value="[null,2,"Screen refresh rate(typically 50-60)"]" >Screen refresh rate(typically 50-60)
</td>

<td data-sheets-value="[null,2,"Hz"]" >Hz
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"interlaces"]" >interlaces
</td>

<td data-sheets-value="[null,2,"Flicker reduction"]" >Flicker reduction
</td>

<td data-sheets-value="[null,2,"Frames"]" >Frames
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"pixel_freq"]" >pixel_freq
</td>

<td data-sheets-value="[null,2,"Clock frequency (Width*Height*Frame rate)"]" >Clock frequency (Width*Height*Frame rate)
</td>

<td data-sheets-value="[null,2,"Hz"]" >Hz
</td>
</tr>
<tr >

<td data-sheets-value="[null,2,"aspect_ratio"]" >aspect_ratio
</td>

<td data-sheets-value="[null,2,"Complex, See below"]" >Complex, see "Additional settings"
</td>

<td >
</td>
</tr>
</tbody>
</table>
Don't be afraid. This is a pretty significant list of confusing options, but the data sheet will provide most of the unknowns. For the rest, we will leave them at defaults or educated guesses, honed with trial and error.

I found my data sheet by searching for the part number: [KD50G21-40NT-A1](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/KD50G21-40NT-A1.pdf)* (PDF LINK.) This was found on an official looking sticker, but a number of other serial numbers and seemingly random strings were also there.

Looking it over, the first bits of useful information are found on page 7 Fig 5.2/5.3 . Here we find resolution, and voltage information. The most important is confirmation that the logic voltage is 3.3v. This allows us to drive the LCD with out a level shiftier:

[![voltage](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/voltage.png)
](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/voltage.png)

Page 10 Fig. 5.6/5.7 gives us a pinout of the ribbon cable and led driver information:

[![pinout](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/pinout.png)
](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/pinout.png)

Page 14 Fig. 6.2.3 gives us a lot of data on timings. These are pretty impossible to figure out with out documentation.

[![timing](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/timing.png)
](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/timing.png)

Additionally, you're going to find a lot of diagrams showing what the display output looks like, these are going to be pretty meaningless with out a scope or logic analyzer.

*This pdf was found hosted on adafruit.com, but not linked on the screen sales page. I don't know why.


## Additional settings


Lets now look at three settings you didn't see in the data sheet. Consult the following tables for output_format, rgb_order, and aspect_ratio.


#### output_format:


<table cellpadding="0" cellspacing="0" border="1" dir="ltr" > 
<tbody >
<tr >

<td data-sheets-value="[null,3,null,1]" >1
</td>

<td data-sheets-value="[null,2,"DPI_OUTPUT_FORMAT_9BIT_666"]" >DPI_OUTPUT_FORMAT_9BIT_666
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,2]" >2
</td>

<td data-sheets-value="[null,2,"DPI_OUTPUT_FORMAT_16BIT_565_CFG1"]" >DPI_OUTPUT_FORMAT_16BIT_565_CFG1
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,3]" >3
</td>

<td data-sheets-value="[null,2,"DPI_OUTPUT_FORMAT_16BIT_565_CFG2"]" >DPI_OUTPUT_FORMAT_16BIT_565_CFG2
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,4]" >4
</td>

<td data-sheets-value="[null,2,"DPI_OUTPUT_FORMAT_16BIT_565_CFG3"]" >DPI_OUTPUT_FORMAT_16BIT_565_CFG3
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,5]" >5
</td>

<td data-sheets-value="[null,2,"DPI_OUTPUT_FORMAT_18BIT_666_CFG1"]" >DPI_OUTPUT_FORMAT_18BIT_666_CFG1
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,6]" >6
</td>

<td data-sheets-value="[null,2,"DPI_OUTPUT_FORMAT_18BIT_666_CFG2"]" >DPI_OUTPUT_FORMAT_18BIT_666_CFG2
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,7]" >7
</td>

<td data-sheets-value="[null,2,"DPI_OUTPUT_FORMAT_24BIT_888"]" >DPI_OUTPUT_FORMAT_24BIT_888
</td>
</tr>
</tbody>
</table>
We will be using option 6 for 18 bit color, CFG1. I don't for the life of me know what CFG1 is, but CFG2 didn't work (trial and error.)


#### rgb_order:


<table cellpadding="0" cellspacing="0" border="1" dir="ltr" > 
<tbody >
<tr >

<td data-sheets-value="[null,3,null,1]" >1
</td>

<td data-sheets-value="[null,2,"DPI_RGB_ORDER_RGB"]" >DPI_RGB_ORDER_RGB
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,2]" >2
</td>

<td data-sheets-value="[null,2,"DPI_RGB_ORDER_BGR"]" >DPI_RGB_ORDER_BGR
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,3]" >3
</td>

<td data-sheets-value="[null,2,"DPI_RGB_ORDER_GRB"]" >DPI_RGB_ORDER_GRB
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,4]" >4
</td>

<td data-sheets-value="[null,2,"DPI_RGB_ORDER_BRG"]" >DPI_RGB_ORDER_BRG
</td>
</tr>
</tbody>
</table>
Option 1 here is fine unless you wanted to get creative with your wiring.


#### aspect_ratio:


<table cellpadding="0" cellspacing="0" border="1" dir="ltr" > 
<tbody >
<tr >

<td data-sheets-value="[null,3,null,1]" >1
</td>

<td data-sheets-value="[null,2,"HDMI_ASPECT_4_3"]" >HDMI_ASPECT_4_3
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,2]" >2
</td>

<td data-sheets-value="[null,2,"HDMI_ASPECT_14_9"]" >HDMI_ASPECT_14_9
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,3]" >3
</td>

<td data-sheets-value="[null,2,"HDMI_ASPECT_16_9"]" >HDMI_ASPECT_16_9
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,4]" >4
</td>

<td data-sheets-value="[null,2,"HDMI_ASPECT_5_4"]" >HDMI_ASPECT_5_4
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,5]" >5
</td>

<td data-sheets-value="[null,2,"HDMI_ASPECT_16_10"]" >HDMI_ASPECT_16_10
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,6]" >6
</td>

<td data-sheets-value="[null,2,"HDMI_ASPECT_15_9"]" >HDMI_ASPECT_15_9
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,7]" >7
</td>

<td data-sheets-value="[null,2,"HDMI_ASPECT_21_9"]" >HDMI_ASPECT_21_9
</td>
</tr>
<tr >

<td data-sheets-value="[null,3,null,8]" >8
</td>

<td data-sheets-value="[null,2,"HDMI_ASPECT_64_27"]" >HDMI_ASPECT_64_27
</td>
</tr>
</tbody>
</table>
While not exactly correct I use option 6 with the best results.


## /boot/config.txt


Every thing above has to be added to the config.txt and only in 2 lines:

    
    dpi_output_format=458773
    hdmi_timings=800 1 0 48 0 480 0 13 3 32 0 0 0 60 0 23040000 6


Points lost for readability, but the data is there.

dpi_output is a uint32 representation of all the binary values in the dpi_output table above.

hdmi_timings is a space delimited list of the hdmi_timings table above.

Rewriting these by hand became very old, very quickly. So I've created a spreadsheet to aid those trying to do the same.

It can be downloaded [here](https://github.com/robertely/dpi666/blob/master/documents/DPI%20%20Options.ods), or viewed live in Google Sheets [here](https://docs.google.com/spreadsheets/d/15KRhR_ewzdGEeD576rL36FbblRVt5HGhNZakOgW-zg4/edit?usp=sharing).

The rest of the options are comparatively trivial:

    
    ###
    # For use with the adafruit 5' tft only.
    ##
    
    #Overscan Information.
    overscan_left=-32
    overscan_right=-32
    overscan_top=-32
    overscan_bottom=-32
    framebuffer_width=800
    framebuffer_height=480
    
    # Disable spi and i2c, we need these pins.
    dtparam=spi=off
    dtparam=i2c_arm=off
    
    #Enable the lcd, enable custom display sizes with CVT, set as the default output.
    enable_dpi_lcd=1
    dpi_group=2
    dpi_mode=87 # Hdmi CVT
    display_default_lcd=1
    
    dpi_output_format=458773
    hdmi_timings=800 1 0 48 0 480 0 13 3 32 0 0 0 60 0 23040000 6
    


With these changes you should be able to reboot and get some kind of picture.


## Testing the frame buffer


At first, the best way to test is with images, I used these in testing:

[![colorbars](http://reasonablycorrect.com/wp-content/uploads/2015/03/colorbars-150x150.gif)
](http://reasonablycorrect.com/wp-content/uploads/2015/03/colorbars.gif) [![VWBlue](http://reasonablycorrect.com/wp-content/uploads/2015/03/VWBlue-150x150.jpg)
](http://reasonablycorrect.com/wp-content/uploads/2015/03/VWBlue.jpg) [![REDChecker](http://reasonablycorrect.com/wp-content/uploads/2015/03/REDChecker-150x150.png)
](http://reasonablycorrect.com/wp-content/uploads/2015/03/REDChecker.png) [![placekitten](http://reasonablycorrect.com/wp-content/uploads/2015/03/placekitten-150x150.jpg)
](http://reasonablycorrect.com/wp-content/uploads/2015/03/placekitten.jpg)

To display these images directly to the frame buffer with out all the fuss involved with X, use a utility called fbi.

    
    $ cd ~



    
    $ sudo apt-get install fbi



    
    $ wget http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/colorbars.gif



    
    $ sudo fbi -T 1 colorbars.gif


The "-T 1" lets you run the utility over ssh. I didn't have a keyboard attached for most of my work.

Once you are confident you have color down you may want to test acceleration. HD video with omxplayer is the fastest way. [Big Buck Bunny](https://peach.blender.org/download/) is a free as in beer open source video available in a variety of formats and sizes.

    
    $ cd ~



    
    $ wget https://archive.org/download/BigBuckBunny_328/BigBuckBunny_512kb.mp4



    
    $ sudo apt-get update



    
    <span class="kw2">$ sudo</span> <span class="kw2">apt-get install</span> omxplayer



    
    $ omxplayer BigBuckBunny_512kb.mp4


*As a side note that kitten looks [horrifying](http://reasonablycorrect.com/wp-content/uploads/2015/03/ohgod.png) with just red connected.


## Troubleshooting


My first attempt at this was on a bread board. I lost an entire afternoon to this because of cross talk on the color lines and my inability to recognize it.

Here is an ideal color bar test image:

[![colorbars](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/colorbars-150x150.gif)
](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/colorbars.gif)

Here is what I was seeing(The smudge is from a screen protector.):

[![badcolors](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/badcolors-150x150.jpg)
](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/badcolors.jpg)

As you can see It manifests in a really interesting way. Color extremes will look perfectly normal. for instance pure red, or #FF0000 in hex will look fine, but #CC0000 will look "weird"

Lets visualize what the bus state will be showing those colors in a perfect world:

_Black boxes "off" or logical zero, and white boxes "on" or logical one_

#FF0000

[![FF0000](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/FF00001.png)
](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/FF00001.png)

#CC0000

[![CC0000](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/CC0000.png)
](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/CC0000.png)

Now lets look at what #CC0000 could look like in the real world with cross talk:

[![CC0000-real](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/CC0000-real.png)
](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/CC0000-real.png)

Are pins 6, 3, or 2 On or Off? The screen doesn't know either.

Capacitance from the breadboard allowed current to seep into the adjacent lines. With out a way to visualize this I would have torn my hair out trying to figuring it out.

Once I setup my logic analyzer there was obvious something was up. This image represents one horizontal line. The top trace shows the signal on the 7th and most significant bit of the blue bus. It should line up perfectly with the falling edge of the clock pulse but it's all over the place.

[![badlogic](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/badlogic-1024x393.png)
](http://blog.reasonablycorrect.com/wp-content/uploads/2015/03/badlogic.png)


## Wrapping up


It took me a while to get every thing working, but that was mostly because there was a serious lack of documentation. Besides getting a cheap screen working I gained a pile of knowledge on display technologies and the inner workings of the Raspberry Pi.  I wrote this partially because it helped me commit my findings to memory, but mostly because I wanted it documented for others. I hope you get as much out of this article as I did writing it.


## TL;DR


If you're feeling brave/stupid: Download [this](https://github.com/robertely/dpi666/raw/master/setup/dt-blob.bin), and [this](https://raw.githubusercontent.com/robertely/dpi666/master/setup/config.txt). Wire it like [this](https://raw.githubusercontent.com/robertely/dpi666/master/documents/DPI%20Connection%20table.png), reboot, and hope.



Edit: 3/5/15, Extended thanks section to include Dom