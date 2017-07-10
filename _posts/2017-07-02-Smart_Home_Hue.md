---
layout: post
title: Smart Home Integration of an Echo, a Raspberry Pi, and Hue Lights
---

I connect an Echo, a Raspberry Pi, and Hue Lights to play audio synced to my lights via a voice command.

Smart devices in our homes is quickly becoming the norm.  Amazon, Google, and Apple all have smart speaker products that can communicate and control other devices, such as lights, TVs, speakers, thermostats, and pretty much any other device that can connect to the internet.  Though the out of box support and compatibility has been pretty strong, I've found that these interactions are limited to two devices: the smart speaker (e.g. Echo, Google Home) and one smart device (e.g. lights).  If you wanted to control several smart devices, you'd generally need to have several voice commands.  From a convenience standpoint, this is definitely not ideal compared to a single voice command.

Luckily, another increasingly popular trend in tech can solve this problem.  Affordable, miniature, yet capable general purpose computers are now plentiful in the market and are great for small tech projects.  In this post, I use a Raspberry Pi Zero W to integrate my Echo, Hue Lights, and set of speakers to play a song synced to my lights from a single voice command.  I leverage the Echo's smart device skills, the Hue's API, and the Raspberry Pi's linux OS and audio capabilities.  To detail everything, I'll go through the following sections:

1. The Equipment
2. Echo to Pi
3. Pi to Hue
4. Pi Audio
5. All Together
6. Next Steps

### The Equipment
I used following products to complete this project:

- Amazon Echo Dot
- Raspberry Pi Zero W
- USB sound card
- PC speakers
- Phillips Hue Color Ambiance Lights

In addition, the Echo, Raspberry Pi, and Hue Lights were connected to the same Wifi network.

### Echo to Pi
The first step is to get the Echo to communicate with the Pi.  There are a couple ways to enable to this communication: AWS Lambda and local smart home communication.  

The Lambda option sends your voice commands to an AWS server, which interprets your command and returns an output.  This is very powerful.  The Lambda function can be coded to interpret almost any voice command with the help of external APIs and can return code that can control several devices across the internet.  There is a price for this flexibility and power, however.  First, there's the actual price.  Creating a Lambda function requires an AWS account, which means you will get billed for each call of your Lambda function.  The price isn't very high and you do receive plenty of free Lambda calls, but this service will cost money at some point.  Second, the Lambda function introduces some lag to your Echo command.  Since your voice command travels from your local network to an AWS server and then back to your local network, the response time isn't instantaneous.  Again, this con isn't a deal breaker, but it is something to consider when using the Lambda function.

The other option, which is the one I used, is local smart home communication.  The Echo has native support for communicating directly with smart devices in your home.  This feature offers fast, direct, and free communication.  The only con is that the feature is very simple in terms of supported voice commands.  You are usually limited to turning a device on or turning it off.  You don't get the unlimited number of voice commands of the Lambda function here.  For this project, this simplicity doesn't really matter as all I want to do is to turn on music and turn it off.  Since there's no limitation there and because of the speed and free price of this feature, I chose to implement this native smart device communication for the project.

On my Raspberry Pi, I downloaded this [GitHub Repo] (https://github.com/toddmedema/echo), which provides code that allows your Pi to emulate a WeMo smart device.  I modified the '''example-minimal.py''' file to [this] (add_link).  After discovering this device on my Echo, I was able to use this command (I named my command Pitbull):

"Echo, turn Pitbull on"

### Pi to Hue
Now that my Pi received the command from the Echo, the Pi had two simultaneous tasks: control the Hue Lights and play the music.  I'll start with controlling the Hue Lights.  To achieve this, I referenced the Phillips Hue API [link] (https://developers.meethue.com/philips-hue-api) to connect to my Hue bridge.  The Hue API allows any computer to control the lights in almost any possible way.  You can set the color, response time, brightness, number of lights, and more with this single API.  For my project, I kept it pretty simple by just modifying the color of the lights.  There are several different methods to control the color of the Hue lights, but the most turn-key solution was modifying the "hue" setting to the integer value that corresponds to your desired color.  Using this [site] (https://www.homegear.eu/index.php/Philips_hue_Color_Light_Reference), I found the "hue" values for green, red, and white, which are the primary colors I used for the project.  Using a timer, I set my Hue Lights to change colors at specific times [code] (insert_here).

### Pi Audio
Next, I configured my Pi to play audio through PC speakers.  Since the Raspberry Pi Zero W does not have a 3.5mm headphone port, I had to buy a USB sound card to connect the speakers to the Pi.  After the connecting the sound card to the Pi, i followed this [guide] (https://computers.tutsplus.com/articles/using-a-usb-audio-device-with-a-raspberry-pi--mac-55876), which walks through setting the USB sound card as the default audio output.  To my knowledge, the Pi doesn't come with software to play mp3s, but there are many utilities you can download.  I used mpg321 for my specific project.


### All Together
Now that everything was built out, I tested my project:

Ran "example-minimal.py" on Pi
Me: "Echo Turn Pitbull On"
Echo: "Turning on Pitbull"

(Video)

### Next Steps and Conclusion
Although the project works, there are several things to improve for next time.

First, I found that once I start the command, I cannot turn it off--the music keeps on looping for some reason.  To solve this, I need to either stop the process after one loop or find out how to get the "off" command to work.

I also would like to add more songs.  Right now, the Pi will only respond to one command and will play only one song.  To add multiple songs, I would need to either duplicate my code to add a separate device or use an AWS Lambda function that could interpret which song I wanted from the voice command.  The Lambda function would be more powerful, but it would have downsides, mentioned earlier in the post.

Finally, I would like the light visualization of the songs to be automatic.  Currently, I hard-coded the light sequences and colors to sync with the song.  Ideally, I could write code that analyzes the song's beat and automatically create a light sequence that corresponds with the music.  To accomplish this, I'd need to find sound processing software and then use some sort of algorithm to determine when to change the lights.  

Overall, the project was a success!  I managed to control my lights and speakers through a voice command, which is not possible with out of the box solutions.  I also got some experience with a Raspberry Pi and just scratched the surface of what it's capable of doing.  Combined with an Echo, the Pi has a ton of potential for cool, use projects around the house.
