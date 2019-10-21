
# zpl-image

A pure javascript module that converts images to Z64-encoded GRF bitmaps for use with ZPL.
Works in both node.js and modern browsers.

This module provides the following features:

  - Converts the image to grayscale, then applies a user-supplied blackness
    threshold to decide which pixels are black.
  - Optionally removes any empty/white space around the edges of the image.
  - Optionally rotates the image to one of the orthogonal angles.  This step
    is often necessary as ZPL does not provide the ability to rotate an image 
    during formatting.
  - Converts the monochrome image to a GRF bitmap.
  - Uses zlib in node.js or pako.js in the browser to compress the GRF bitmap.
  - Encodes the compressed bitmap in base64 and calculates the required CRC16 checksum.

The blackness threshold is specified as an integer between 1 and 99 (think of it as a
gray percentage).  Pixels darker than the gray% are converted to black.  The default is 50.

Rotation is specified as one of the values:

  - `'N'` : No rotation, the default.
  - `'L'` : Left, 90 degrees counter-clockwise rotation.
  - `'R'` : Right, 90 degrees clockwise rotation.
  - `'I'` : Inverted, 180 degrees rotation.
  - `'B'` : Same as `'I'` but named to match the ZPL notation for bottom up.

Blackness and rotation are passed via an options object.  For example, to specify
a black threshold of 56% and rotation of -90 degrees, you would pass in:

```javascript
	{ black:56, rotate:'L' }
```

Trimming of empty space around the image is enabled by default.  To disable, specify
the option `notrim:true`.

## Demo

Included with this module is the file `zpl-image.html`.  You can run it directly
from the browser using the `file://` scheme.  It lets you drag and drop an image
and then interactively adjust the blackness threshold and rotation.

When you are satisfied with the results, you can copy the generated ZPL to the clipboard.
The ZPL will have the following format:

```
^FX filename.ext (WxHpx, X-Rotate, XX% Black)^FS
^GFA,grflen,grflen,rowlen,:Z64:...base64...encoding...:crc16
```

`^FX ... ^FS` is a ZPL comment.

`^GF` is the ZPL command for use-once image rendering (that is, the image is not
saved to the printer for later recall by other label formats).

The rendered image is the actual Z64 data decoded and then drawn to a canvas.
If you are interested in that bit of functionality, look for `z64ToCanvas`
in the html file.

## Generic Browser Usage

To use in the browser, include the following two scripts:

```html
<script type="text/javascript" src="url-path-to/pako.js"></script>
<script type="text/javascript" src="url-path-to/zpl-image.js"></script>
```

There is a version of pako.js included with this module, but it will not be updated
frequently.  It is primarily intended for the demo html file but should be sufficient
for production use.

```javascript
// Works with <img> and <canvas> elements or any element that is compatible with
// CanvasRenderingContext2D.drawImage().
let img = document.getElementById('image');
let res = imageToZ64(img);	// Uses all defaults

// res.length is the uncompressed GRF length.
// res.rowlen is the GRF row length.
// res.z64 is the Z64 encoded string.
let zpl = `^GFA,${res.length},${res.length},${res.rowlen},${res.z64}`;
```

An alternative for when you already have the pixel values in RGBA format
(either in a Uint8Array or Array of integers clamped to 0..255) is 
`rgbaToZ64()`.  This function is the lower-level converter used
by both node.js and `imageToZ64()`.  See the node.js section for more details.

```javascript
// `rgba` is an array of RGBA values.
// `width` is the width of the image, in pixels.
// The return value is the same as above.
let res = rgbaToZ64(rgba, width, { black:55, rotate:'I' });
```

## RequireJS Browser Usage

This is untested but the module exports are wrapped in a UMD, so in theory you
should be able to use this with RequireJS.  The exports are the same as with the
generic browser usage:

```javascript
const { imageToZ64, rgbaToZ64 } = require("zpl-image");
```

## Node.js Usage

The return from `require("zpl-image")` is currently a single named function
`rgbaToZ64()`.

```javascript
const rgbaToZ64 = require("zpl-image").rgbaToZ64;
```

The method takes two or three parameters:

```
rgbaToZ64(rgba, width [, opts])
```

`rgba` is an array-like object with length equal to `width * height * 4`.
An array-like object can be a Buffer, Uint8Array, or Array of integers 
clamped to 0..255.  `width` and `height` are the dimensions of the image, in pixels.
Each "quad" of the RGBA array is structured as:

```javascript
rgba[i]   // red 0..255
rgba[i+1] // green 0..255
rgba[i+2] // blue 0..255
rgba[i+3] // alpha (0 == fully transparent, 255 == fully opaque)
```

Because of the varied nature of the node.js ecosystem, zpl-image does not include
any dependencies for image modules.  You need to decide what types of images to
support and which image processing package(s) to use.  Below are some simple
examples showing three different image modules:

  - [pngjs](https://www.npmjs.com/package/pngjs)
  - [omggif](https://www.npmjs.com/package/omggif)
  - [jpeg-js](https://www.npmjs.com/package/jpeg-js)

## pngjs (PNG Conversion)

[pngjs](https://www.npmjs.com/package/pngjs)

```javascript
// Synchronous pngjs usage
const fs = require('fs');
const PNG = require('pngjs').PNG;
const rgbaToZ64 = require('zpl-image').rgbaToZ64;

let buf = fs.readFileSync('tux.png');
let png = PNG.sync.read(buf);
let res = rgbaToZ64(png.data, png.width, { black:53 });

// res.length is the uncompressed GRF length.
// res.rowlen is the GRF row length.
// res.z64 is the Z64 encoded string.
let zpl = `^GFA,${res.length},${res.length},${res.rowlen},${res.z64}`;
```

```javascript
// Async pngjs usage
const fs = require('fs');
const PNG = require('pngjs').PNG;
const rgbaToZ64 = require('zpl-image').rgbaToZ64;

fs.createReadStream('tux.png')
    .pipe(new PNG({ filterType: 4 }))
	.on('parsed', function() {
		// res is the same as above
		let res = rgbaToZ64(this.data, this.width, { black:52, rotate:'R' });
	});
```

## omggif (GIF Conversion)

[omggif](https://www.npmjs.com/package/omggif)

```javascript
const fs = require('fs');
const GIF = require('omggif');
const rgbaToZ64 = require('zpl-image').rgbaToZ64;

let buf = fs.readFileSync('tux.gif');
let gif = new GIF.GifReader(buf);
let rgba = Buffer.alloc(gif.width * gif.height);

// Decode only the first frame
gif.decodeAndBlitFrameRGBA(0, rgba);

let res = rgbaToZ64(rgba, gif.width, { black:47 });
```

## jpeg-js (JPEG Conversion)

[jpeg-js](https://www.npmjs.com/package/jpeg-js)

```javascript
const fs = require('fs');
const JPG = require('jpeg-js');
const rgbaToZ64 = require('zpl-image').rgbaToZ64;

let buf = fs.readFile('tux.jpg');
let jpg = JPG.decode(buf);
let res = rgbaToZ64(jpg.data, jpg.width, { black:51, rotate:'I' });
```

## MIT License

