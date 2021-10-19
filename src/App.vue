<template>
  <div id="app">
    <div class="controls">
      <div class="line">
        <div class="select-file">{{filename}}
          <input type="file" id="upload" hidden @change="processFiles">
          <label for="upload"></label>
          <div class="fake-button mi">&#xe2c6;</div>
        </div>
      </div>
      <div class="line">
        <label class="mr-8" for="width">Width</label><input name="width" type="number" v-model="width" @input="newwidth" maxlength="4" size="4">
        <label class="mh-8" for="height">Height</label><input name="height" type="number" v-model="height" @input="newheight" maxlength="4" size="4">
      </div>
      <div class="line">
        <label class="mr-8" for="bright">Brightness</label>
        <button class="lrad radbtn mi" @click="decBrightness">&#xe15c;</button>
        <input class="slider" name="bright" v-model="brightness" type="range" min="-10" max="10" @change="update">
        <button class="rrad radbtn mi" @click="incBrightness">&#xe147;</button>
        <label class="mh-8" for="cont">Contrast</label>
        <button class="lrad radbtn mi" @click="decContrast">&#xe15c;</button>
        <input class="slider"  name="cont" v-model="contrast" type="range" min="-10" max="10" @change="update">
        <button class="rrad mr-8 radbtn mi" @click="incContrast">&#xe147;</button>
      </div>
      <div class="line">
        <select v-model="dither" @change="changeDither">
          <option v-for="option in dithers" v-bind:value="option.value" :key="option.value">
            {{ option.text }}
          </option>
        </select>
        <label class="mh-8" for="srgb">sRGB</label><input name="srgb" type="checkbox" v-model="sRGB" @change="update">
      </div>
    </div>
    <canvas ref="canvas" :width="width" :height="height" style="border:1px solid #c3c3c3;"></canvas>
  </div>
</template>

<script>

export default {
  name: 'App',
  components: {
  },
  data: function () {
    return {
      filename: "Select File...",
      processing: false,
      brightness: 0,
      contrast: 0,
      sRGB: true,
      image: null,
      original_width: 0,
      original_height: 0,
      width:0,
      height:0,
      ctx: null,
      sRgbToLinear: [],
      dithers: [
        {value:'threshold', text:'Threshold'},
        {value:'bayer', text:'Bayer'},
        {value:'floyd', text: 'Floyd Steinberg'},
        {value:'minerror', text: 'Minimized Average Error'}
      ],
      dither: 'floyd',
      bayerPattern: [
        [  0,129, 32,161,  8,137, 40,169], // Effectively a threshold filter which has a different
        [194, 65,226, 97,202, 73,234,105], // cutoff for each of the 8x8 pixels
        [ 48,177, 16,145, 56,185, 24,153],
        [242,113,210, 81,250,121,218, 89],
        [ 12,141, 44,173,  4,133, 36,165],
        [206, 77,238,109,198, 69,230,101],
        [ 60,189, 28,157, 52,181, 20,149],
        [254,125,222, 93,246,117,214, 85]
      ]
    }
  },
  mounted(){
    let canvas = this.$refs.canvas
    this.ctx = canvas.getContext("2d")
    this.draw()
    for(let c=0;c<256;c++){
      let nc=Math.pow(((c/255) + 0.055) / 1.055, 2.4)*255
      this.sRgbToLinear.push(nc)
      console.log(c,nc)
    }
  },
  methods: {
    changeDither(){
      this.update()
    },
    decContrast(){
      this.contrast=Math.max(-10,parseInt(this.contrast)-1)
      this.update()
    },
    incContrast(){
      this.contrast=Math.min(10,parseInt(this.contrast)+1)
      this.update()
    },
    decBrightness(){
      this.brightness=Math.max(-10,parseInt(this.brightness)-1)
      this.update()
    },
    incBrightness(){
      this.brightness=Math.min(10,parseInt(this.brightness)+1)
      this.update()
    },
    newwidth(){
      this.width=Math.max(1,this.width)
      this.height=Math.max(1,Math.floor((this.width*this.imageRatio)+0.5))
      this.update()
    },
    newheight(){
      this.height=Math.max(1,this.height)
      this.width=Math.max(1,Math.floor((this.height/this.imageRatio)+0.5))
      this.update()
    },
    update(){
      this.$nextTick(()=>{
        this.draw()
      })
    },
    draw(){
      if (this.image){
        this.processing=true
        const t0 = performance.now();
        let offCanvas = document.createElement('canvas')
        offCanvas.width=this.width
        offCanvas.height=this.height
        let offCtx = offCanvas.getContext('2d')
        offCtx.drawImage(this.image,0,0,this.width,this.height)
        
        let imageData = offCtx.getImageData(0, 0, this.width, this.height)
        console.log(imageData)
        let pixels=imageData.data
        let brightness=parseInt(this.brightness)*10
        let contrast=this.contrast
        if (contrast<0){
          contrast=1/(-(contrast/10)+1)
        } else if (contrast>0){
          contrast=1+(contrast/10)
        } else contrast=1
        console.log(contrast)
        for(let pos=0;pos<pixels.length;pos+=4){
          let r=pixels[pos]
          let g=pixels[pos+1]
          let b=pixels[pos+2]
          if (this.sRGB){
            r=this.sRgbToLinear[r]
            g=this.sRgbToLinear[g]
            b=this.sRgbToLinear[b]
          }
          let grey=(0.2126*r)+(0.7152*g)+(0.0722*b)
          pixels[pos]=((grey-128)*contrast)+128+brightness
          pixels[pos+1] = pixels[pos+2] = pixels[pos]
        }
        const t1 = performance.now();
        console.log(`Greyscale ${t1 - t0} milliseconds.`);

        if (this.dither=='minerror'){
          let lineSize=this.width*4
          let oldpixel
          let newpixel
          let err
          for(let y=0;y<this.height;y++){
            let pos=y*this.width*4
            for(let x=0;x<this.width;x++){
              oldpixel = pixels[pos]
              newpixel = oldpixel<128?0:255
              err = oldpixel-newpixel
              pixels[pos] = newpixel
              pixels[pos+1] = pixels[pos+2] = pixels[pos]
              pixels[pos+4] += (err*7/48)
              pixels[pos+8] += (err*5/48)
              pixels[pos+lineSize-8] += (err*3/48)
              pixels[pos+lineSize-4] += (err*5/48)
              pixels[pos+lineSize] += (err*7/48)
              pixels[pos+lineSize+4] += (err*5/48)
              pixels[pos+lineSize+8] += (err*3/48)
              pixels[pos+lineSize+lineSize-8] += (err*1/48)
              pixels[pos+lineSize+lineSize-4] += (err*3/48)
              pixels[pos+lineSize+lineSize] += (err*5/48)
              pixels[pos+lineSize+lineSize+4] += (err*3/48)
              pixels[pos+lineSize+lineSize+8] += (err*1/48)
              pos+=4
            }
          }
        }

        if (this.dither=='floyd'){
          let lineSize=this.width*4
          let oldpixel
          let newpixel
          let err
          for(let y=0;y<this.height;y++){
            let pos=y*this.width*4
            for(let x=0;x<this.width;x++){
              oldpixel = pixels[pos]
              newpixel = oldpixel<128?0:255
              err = oldpixel-newpixel
              pixels[pos]=newpixel
              pixels[pos+1] = pixels[pos+2] = pixels[pos]
              pixels[pos+4] += (err*7/16)
              pixels[pos+lineSize-4] += (err*3/16)
              pixels[pos+lineSize] += (err*5/16)
              pixels[pos+lineSize+4] += (err/16)
              pos+=4
            }
          }
        }

        if (this.dither=='bayer'){
          let pos=0
          for(let y=0;y<this.height;y++){
            let my=y & 7
            for(let x=0;x<this.width;x++){
              pixels[pos] = pixels[pos] > this.bayerPattern[x & 7][my]?255:0
              pixels[pos+1] = pixels[pos+2] = pixels[pos]
              pos+=4
            }
          }
        }
        if (this.dither=='threshold'){
          for(let pos=0;pos<imageData.data.length;pos+=4){
            pixels[pos] = pixels[pos] > 127?255:0
            pixels[pos+1] = pixels[pos+2] = pixels[pos]
          }
        }
        const t2 = performance.now();
        console.log(`Dither ${t2 - t1} milliseconds.`);

        offCtx.putImageData(imageData,0,0)
        let image = new Image()
        image.src = offCanvas.toDataURL()
        image.onload = ()=>{
          // console.log("Resize to "+this.width)
          // let newWidth=400
          // let newHeight=Math.floor((newWidth*this.imageRatio)+0.5)
          // console.log(this.width,newWidth,this.height,newHeight)
          // this.ctx.imageSmoothingEnabled = true
          // this.ctx.mozImageSmoothingEnabled = true
          // this.ctx.webkitImageSmoothingEnabled = true
          // this.ctx.msImageSmoothingEnabled = true
          // this.ctx.drawImage(image,0,0,newWidth,newHeight)
          this.ctx.drawImage(image,0,0)
          offCanvas.remove()
          const t3 = performance.now();
          console.log(`Draw ${t3 - t2} milliseconds.`);
        }
        this.processing=false
      }
    },
    processFiles(event){
      var input = event.target;
      // Ensure that you have a file before attempting to read it
      if (input.files && input.files[0]) {
          this.filename = input.files[0].name
          // create a new FileReader to read this image and convert to base64 format
          var reader = new FileReader();
          // Define a callback function to run, when FileReader finishes its job
          reader.onload = (e) => {
              // Note: arrow function used here, so that "this.imageData" refers to the imageData of Vue component
              // Read image as base64 and set to imageData
            let img = new Image()
            img.onload = ()=>{
              console.log(img.width,img.height)
              this.original_width=img.width
              this.original_height=img.height
              this.imageRatio=img.height/img.width
              this.width=Math.floor(1200/this.imageRatio)
              this.height=1200
              this.image=img
              this.$nextTick(()=>{
                this.draw() // ??
              })
            }
            img.src = e.target.result
          }
          // Start the reader job - read file as a data url (base64 format)
          //reader.readAsBinaryString(input.files[0])
          reader.readAsDataURL(input.files[0]);
      }      
    }
  }
}
</script>

<style>
#app{
  margin:10px;
}
body,html{
  margin:0;
  padding:0;
  background-color: #222;
  color: #fff;
  font-family: Arial, Helvetica, sans-serif;
}
.mi{
  font-family: 'Material Icons';
  font-size: 15px;
}
.mh-8{
  margin-left:8px;
  margin-right:8px;
}
.mr-8{
  margin-right:8px;
}
.line{
  margin-bottom:8px;
}
.radbtn{
  border: 0;
  background-color: #FFF;
}
.lrad{
  border-radius: 6px 0 0 6px;
}
.rrad{
  border-radius: 0 6px 6px 0;
}
.controls{
  padding:8px;
  background-color: #555;
  border-radius: 10px;;
}
.slider {
  border-radius: 0;
  margin:0;
  position:relative;
  top:1px;
  -webkit-appearance: none;
  width: 200px;
  height: 17px;
  background: #d3d3d3;
  outline: none;
  opacity: 0.7;
  -webkit-transition: .2s;
  transition: opacity .2s;
}

.slider:hover {
  opacity: 1;
}
.slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 21px;
  height: 17px;
  background: #0f04aa;
  cursor: pointer;
}
.slider::-moz-range-thumb {
  width: 21px;
  height: 21px;
  background: #0f04aa;
  cursor: pointer;
}
.select-file{
  position:relative;
  display:inline-block;
  border: 1px solid #FFFFFF;
  border-radius: 4px;
  padding:2px;
  width:300px;
}
.select-file label{
font-size: 14px;
  color: #FFF;
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(255,255,255,0.1);
  padding: 3px 10px 4px 10px;
  border-radius: 4px;
}
.select-file .fake-button{
  display: inline;
  position: absolute;
  right: 0;
  background-color: rgba(255,255,255,0.5);
  color: #000;
  padding: 3px 10px 4px 10px;
  border-radius: 3px;
  top: 0;
  bottom: 0;
}
</style>
