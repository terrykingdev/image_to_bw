# Image to Black & White

Demo with Vue 2 to show loading an image and dithering to black and white. Has threshold, Bayer, Floyd Steinberg and Minimized Average Error dithering algorithms.

### Original  
![Original](./images/Mona_Lisa.jpg)

### Threshold  
![Threshold](./images/threshold.png)

### Bayer (Ordered dithering)
A threshold technique where the threshold changes based on the value in an 8x8 array.  
https://en.wikipedia.org/wiki/Ordered_dithering  
![Bayer](./images/bayer.png)

### Floyd Steinberg  
Divides the error in pixel value into 16 parts and distributes  the error to 4 neighboring pixels.  
https://en.wikipedia.org/wiki/Floyd%E2%80%93Steinberg_dithering  
![Floyd Steinberg](./images/floyd.png)

### Minimized Average Error  
Very similiar to Floyd Steinberg but uses a larger kernel (48 error parts among 12 neighboring pixels) which helps reduce stray pixels in black and white areas.  
https://en.wikipedia.org/wiki/Error_diffusion#minimized_average_error  
![Minimized Average Error](./images/minimized.png)

## Project setup
```
npm install
```

### Compiles and hot-reloads for development
```
npm run serve
```

### Compiles and minifies for production
```
npm run build
```

### Lints and fixes files
```
npm run lint
```

### Customize configuration
See [Configuration Reference](https://cli.vuejs.org/config/).
