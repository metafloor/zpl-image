
# zpl-image

A pure javascript module that converts images to either Z64-encoded or ACS-encoded GRF bitmaps for use with ZPL.
The term ACS (Alternative Compression Scheme) denotes the run-length compression algorithm described in the section
of the ZPL Reference Manual titled "Alternative Data Compression Scheme".  Z64 typically gives better compression
but is not available on all printers (especially older ones).  The ACS encoding should work on any printer made
since the mid 90s, maybe earlier.

This module provides the following features:

  - Works in both node.js and modern browsers.
  - Converts the image to grayscale, then applies a user-supplied blackness
    threshold to decide which pixels are black.
  - Optionally removes any empty/white space around the edges of the image.
  - Optionally rotates the image to one of the orthogonal angles.  This step
    is often necessary as ZPL does not provide the ability to rotate an image 
    during formatting.
  - Converts the monochrome image to a GRF bitmap.
  - Converts the GRF bitmap to either Z64 or ACS encoding.
  - For Z64, zlib in node.js or pako.js in the browser is used for compression.

The blackness threshold is specified as an integer between 1 and 99 (think of it as a
gray percentage).  Pixels darker than the gray% are converted to black.  The default is 50.

Rotation is specified as one of the values:

  - `'N'` : No rotation, the default.
  - `'L'` : Left, 90 degrees counter-clockwise rotation.
  - `'R'` : Right, 90 degrees clockwise rotation.
  - `'I'` : Inverted, 180 degrees rotation.
  - `'B'` : Same as `'L'` but named to match the ZPL notation.

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

When you are satisfied with the results, select either Z64 or ACS encoding and
click the clipboard icon to copy the ZPL.  The ZPL will have the following format:

```
^FX filename.ext (WxHpx, X-Rotate, XX% Black)^FS
^GFA,grflen,grflen,rowlen,...ASCII-armored-encoding...
```

`^FX ... ^FS` is a ZPL comment.

`^GF` is the ZPL command for use-once image rendering (that is, the image is not
saved to the printer for later recall by other label formats).

The rendered image displayed on the page is the actual data decoded and then drawn
to a canvas.  If you are interested in that bit of functionality, look for `z64ToCanvas`
and `acsToCanvas` in the `zpl-image.html` file.

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
// Works with <img> and <canvas> elements or any element that is
// compatible with CanvasRenderingContext2D.drawImage().
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

The same interfaces exist for ACS encoding, using the functions `imageToACS()` and
`rgbaToACS()`.  The returned object from each function is identical to the above, with
the exception that the encoded text is in the `acs` property instead of `z64`.

## RequireJS Browser Usage

This is untested but the module exports are wrapped in a UMD, so in theory you
should be able to use this with RequireJS.  The exports are the same as with the
generic browser usage:

```javascript
// Use the Z64 interface
const { imageToZ64, rgbaToZ64 } = require("zpl-image");

// Or the ACS interface
const { imageToACS, rgbaToACS } = require("zpl-image");
```

## Node.js Usage

The exports from `require("zpl-image")` are the functions `rgbaToZ64()` and
`rgbaToACS()`.

```javascript
// The Z64 interface
const rgbaToZ64 = require("zpl-image").rgbaToZ64;

// The ACS interface
const rgbaToACS = require("zpl-image").rgbaToACS;

```

Both methods take two or three parameters:

```
rgbaToZ64(rgba, width [, opts])
rgbaToACS(rgba, width [, opts])
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

All of the following examples show Z64 encoding but can be switched to ACS 
by simply renaming `Z64` to `ACS`.

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
let rgba = Buffer.alloc(gif.width * gif.height * 4);

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

let buf = fs.readFileSync('tux.jpg');
let jpg = JPG.decode(buf);
let res = rgbaToZ64(jpg.data, jpg.width, { black:51, rotate:'I' });
```

## MIT License

