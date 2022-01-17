# Autonomous SmartDesk 2 WiFi controller

(documentation is very much a WIP)

### Buying the cable

The desk has a keypad connected to the motor driver using a cable that *looks* like ethernet, but which is actually 10 wires instead of 8 - it's a 10P10C connector. I bought a cable+connector on Amazon for $5. Autonomous only uses 4 of the 10 wires (so it's kindof annoying they went with a 10P connector instead of something like a phone jack, which would have been like 5¢).

### The microcontroller

Then I bought an [ESP8266 chip from Amazon](https://www.amazon.com/Aceirmc-ESP8266-Internet-Development-Compatible/dp/B07V84VWSM) (3 for $12), and [flashed Espruino onto it](https://www.espruino.com/ESP8266_Flashing). Espruino is awesome - you plug in your chip via USB, open the Espruino Web IDE, write some JavaScript, and hit "flash" or "send".

### Detour: bundling and minifying the JS before flashing

I got a little into the weeds and built a whole Rollup-based bundler + command-line flashing tool instead of using the IDE because I wanted to squeeze *a lot* more code onto the device. It's called `espz` ([video](https://twitter.com/_developit/status/1367657018174148618)) - you can try it out if you're daring, the only requirement is that you've flashed the Espruino base image (kinda like an OS) using their guide. It can even send code to the chip over wifi! The npm package has some docs:
https://www.npmjs.com/package/espz

### Making ESP8266 talk to the desk

From there, I found Stefan's exceptional reverse-engineered protocol description. With that information in hand, I wrote up some JavaScript for the microcontroller that listens for data from the desk's motor controller and can send it commands like UP/DOWN/M1/M2/etc:
https://gist.github.com/developit/d610e45a522810b5287db61e554ae9c9#file-desk-js

### Making the desk talk to the ESP8266 (for longer than 2 seconds)

I had to do a bunch of performance testing to find the right way to handle the barrage of data from that motor controller in Espruino. Most of my naive attempts couldn't process the data fast enough, and when that happens Espruino starts buffering the received data. That immediately fills the device's memory and it crashes (or overheats!). Two tricks are required to make this work:
1. in Espruino, ["jumping" through a string using `indexOf()`](https://gist.github.com/developit/d610e45a522810b5287db61e554ae9c9#file-desk-js-L180) is way faster than a regex or character loop.
2. the motor controller sends tons of duplicate data - [skipping repeated messages](https://gist.github.com/developit/d610e45a522810b5287db61e554ae9c9#file-desk-js-L174-L177) saves a lot of compute time.

### Exposing a REST API and Web GUI

Then I wrote a bit more code that turns the ESP8266 into a web server (it's actually pretty good at this!). The web server hosts an API and a little controller/keypad web app that hits that API and allows me to operate the desk.
**Web server + API:** https://gist.github.com/developit/d610e45a522810b5287db61e554ae9c9#file-index-js
**UI (just an HTML file with some JS in it):** https://gist.github.com/developit/d610e45a522810b5287db61e554ae9c9#file-ui-html

### Powering it up

Conveniently, when you plug the connector wire into the motor controller, it provides power over one of the wires. It provides 5v, but the ESP8266 chip is 3.3v. Normally that would fry the chip, but as it turns out the ESP8266 is 5v tolerant. This means it boots and runs as long as the wire is connected to the motor controller - no battery or wall adapter needed!

Here's a video of it working (using the UI on my computer for so I could record from my phone):
https://twitter.com/_developit/status/1367842479115034624
