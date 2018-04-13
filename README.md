# UPNG.js
A small, fast and advanced PNG / APNG encoder and decoder. It is the main PNG engine for [Photopea image editor](https://www.photopea.com).

* [Examples of PNGs minified by UPNG.js](https://blog.photopea.com/png-minifier-inside-photopea.html#examples)
* [Try UPNG.js in Photopea](https://www.photopea.com) - open an image and press File - Save for web, play with the Quality
* [UPNG.Photopea.com](http://upng.photopea.com) - a separate minifier app, that uses UPNG.js
* Support us by [making a donation](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=ivan%40kuckir%2ecom&lc=CZ&item_name=UPNG%2ejs&currency_code=USD&bn=PP%2dDonationsBF%3abtn_donateCC_LG%2egif%3aNonHosted).

Download and include the `UPNG.js` file in your code, or get it from NPM:

```sh
npm install upng-js
```

## Encoder

UPNG.js supports APNG and the interface expects "frames". Regular PNG is just a single-frame animation (single-item array).

#### `UPNG.encode(imgs, w, h, cnum, [dels])`
* `imgs`: array of frames. A frame is an ArrayBuffer containing the pixel data (RGBA, 8 bits per channel)
* `w`, `h` : width and height of the image
* `cnum`: number of colors in the result;  0: all colors (lossless PNG)
* `dels`: array of delays for each frame (only when 2 or more frames)
* returns an ArrayBuffer with binary data of a PNG file

UPNG.js can do a lossy minification of PNG files, similar to [TinyPNG](https://tinypng.com/) and other tools. It performed quantization with [k-means algorithm](https://en.wikipedia.org/wiki/K-means_clustering) in the past, but now we use [K-d trees](https://en.wikipedia.org/wiki/K-d_tree).

Lossy compression is allowed by the last parameter `cnum`. Set it to zero for a lossless compression, or write the number of allowed colors in the image. Smaller values produce smaller files. **Or just use 0 for lossless / 256 for lossy.**

#### `UPNG.encodeLL(imgs, w, h, cc, ac, depth, [dels])` - low-level encode
* `imgs`: array of frames. A frame is an ArrayBuffer containing the pixel data (corresponding to following parameters)
* `w`, `h` : width and height of the image
* `cc`, `ac`: number of color channels (1 or 3) and alpha channels (0 or 1)
* `depth`: bit depth of pixel data (1, 2, 4, 8, 16)
* `dels`: array of delays for each frame (only when 2 or more frames)
* returns an ArrayBuffer with binary data of a PNG file

This function does not do any optimizations, it just stores what you give it. There are two cases when it is useful:
* saving 16-bit colors (note, that PNG is big-endian, unlike Uint16Array in JS)
* your image is too large, and "expanding" to 8-bit RGBA would use too much memory (e.g. 4-bit grayscale 50,000 x 50,000 = 1.25 GB, 8-bit RGBA would be 10 GB)

### Example

An example to optimise a PNG image that is built by Canvas:

```javascript
function getOptimisedPngFromCanvas (canvas) {
    return new Promise((resolve) => {
        canvas.toBlob((pngBlob) => {
            const canvasPngBlobReader = new FileReader();

            canvasPngBlobReader.onloadend = (canvasPngConvertedToArrayBuffer) => {
                /* global UPNG */
                const rawImage = UPNG.decode(canvasPngConvertedToArrayBuffer.target.result);
                const optimisedPngBlobReader = new FileReader();

                optimisedPngBlobReader.onloadend = (optimisedPngConvertedToBase64) => {
                    resolve(optimisedPngConvertedToBase64.target.result);
                };

                optimisedPngBlobReader.readAsDataURL(new Blob([UPNG.encode(
                    [UPNG.toRGBA8(rawImage)[0]],
                    rawImage.width,
                    rawImage.height,
                    0 // 0 = losslessly.
                )], {type: 'image/png'}));
            };

            canvasPngBlobReader.readAsArrayBuffer(pngBlob);
        });
    });
}

getOptimisedPngFromCanvas(document.querySelector('canvas'))
    .then((pngB64) => window.open(pngB64));
```

## Decoder

Supports all color types (including Grayscale and Palettes), all channel depths (1, 2, 4, 8, 16), interlaced images etc. Opens PNGs which other libraries can not open (tested with [PngSuite](http://www.schaik.com/pngsuite/)).

#### `UPNG.decode(buffer)`
* `buffer`: ArrayBuffer containing the PNG file
* returns an image object with following properties:
* * `width`: the width of the image
* * `height`: the height of the image
* * `depth`: number of bits per channel
* * `ctype`: color type of the file (Truecolor, Grayscale, Palette ...)
* * `frames`: additional info about frames (frame delays etc.)
* * `tabs`: additional chunks of the PNG file
* * `data`: pixel data of the image

PNG files may have a various number of channels and a various color depth. The interpretation of `data` depends on the current color type and color depth (see the [PNG specification](https://www.w3.org/TR/PNG/)).

#### `UPNG.toRGBA8(img)`
* `img`: PNG image object (returned by UPNG.decode())
* returns an array of frames. A frame is ArrayBuffer of the image in RGBA format, 8 bits per channel.

### Example
    var img  = UPNG.decode(buff);        // put ArrayBuffer of the PNG file into UPNG.decode
    var rgba = UPNG.toRGBA8(img)[0];     // UPNG.toRGBA8 returns array of frames, size: width * height * 4 bytes.

PNG format uses the Inflate algorithm. Right now, UPNG.js calls [Pako.js](https://github.com/nodeca/pako) for the Inflate and Deflate method.
