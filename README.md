[![Build Status](https://travis-ci.org/lukeapage/pngjs2.svg?branch=master)](https://travis-ci.org/lukeapage/pngjs2) [![Build status](https://ci.appveyor.com/api/projects/status/tb8418jql1trkntd/branch/master?svg=true)](https://ci.appveyor.com/project/lukeapage/pngjs2/branch/master) [![Coverage Status](https://coveralls.io/repos/lukeapage/pngjs2/badge.svg?branch=master&service=github)](https://coveralls.io/github/lukeapage/pngjs2?branch=master) [![npm version](https://badge.fury.io/js/pngjs2.svg)](http://badge.fury.io/js/pngjs2)

pngjs2
========
Simple PNG encoder/decoder for Node.js with no dependencies.

Based on [pngjs](https://github.com/niegowski/node-pngjs) with the follow enhancements.

  * Support for reading 1,2,4 & 16 bit files
  * Support for reading interlace files
  * Support for reading `tTRNS` transparent colours
  * Sync interface as well as async
  * API compatible with pngjs and node-pngjs

Known lack of support for:

  * Extended PNG e.g. Animation
  * Writing in different formats
  * Synchronous write

Requirements
============

* Async - Node.js 0.10 / 0.12 / IO.js
* Sync - Node.js 0.12 / IO.js

Comparison Table
================

Name     |  Forked From | Sync | Async | 16 Bit | 1/2/4 Bit | Interlace | Gamma | Encodes | Tested
---------|--------------|------|-------|--------|-----------|-----------|-------|---------|--------
pngjs2   | pngjs        | Read | Yes   | Yes    | Yes       | Yes       | Yes   | Yes     | Yes
node-png | pngjs        | No   | Yes   | No     | No        | No        | Hidden| Yes     | Manual
pngjs    |              | No   | Yes   | No     | No        | No        | Hidden| Yes     | Manual
png-coder| pngjs        | No   | Yes   | Yes    | No        | No        | Hidden| Yes     | Manual
pngparse |              | No   | Yes   | No     | Yes       | No        | No    | No      | Yes
pngparse-sync | pngparse| Yes  | No    | No     | Yes       | No        | No    | No      | Yes
png-async|              | No   | Yes   | No     | No        | No        | No    | Yes     | Yes
png-js   |              | No   | Yes   | No     | No        | No        | No    | No      | No


Native C++ node decoders:
 * png
 * png-sync (sync version of above)
 * pixel-png
 * png-img

Tests
=====

Tested using [PNG Suite](http://www.schaik.com/pngsuite/). We read every file into pngjs2, output it in standard 8bit colour, synchronously and asynchronously, then compare the original
with the newly saved images.

To run the tests, run `node test`.

The only thing not converted is gamma correction - this is because multiple vendors will do gamma correction differently, so the tests will have different results on different browsers.

In addition we use a tolerance of 3 for 16 bit images in PhantomJS because PhantomJS seems to have non-compliant rules for downscaling 16 bit images.

Installation
===============
```
$ npm install pngjs2  --save
```

Example
==========
```js
var fs = require('fs'),
    PNG = require('node-png').PNG;

fs.createReadStream('in.png')
    .pipe(new PNG({
        filterType: 4
    }))
    .on('parsed', function() {

        for (var y = 0; y < this.height; y++) {
            for (var x = 0; x < this.width; x++) {
                var idx = (this.width * y + x) << 2;

                // invert color
                this.data[idx] = 255 - this.data[idx];
                this.data[idx+1] = 255 - this.data[idx+1];
                this.data[idx+2] = 255 - this.data[idx+2];

                // and reduce opacity
                this.data[idx+3] = this.data[idx+3] >> 1;
            }
        }

        this.pack().pipe(fs.createWriteStream('out.png'));
    });
```
For more examples see `examples` folder.

Async API
================

As input any color type is accepted (grayscale, rgb, palette, grayscale with alpha, rgb with alpha) but 8 bit per sample (channel) is the only supported bit depth. Interlaced mode is not supported.

## Class: PNG
`PNG` is readable and writable `Stream`.


### Options
- `width` - use this with `height` if you want to create png from scratch
- `height` - as above
- `checkCRC` - whether parser should be strict about checksums in source stream (default: `true`)
- `deflateChunkSize` - chunk size used for deflating data chunks, this should be power of 2 and must not be less than 256 and more than 32*1024 (default: 32 kB)
- `deflateLevel` - compression level for delate (default: 9)
- `deflateStrategy` - compression strategy for delate (default: 3)
- `filterType` - png filtering method for scanlines (default: -1 => auto, accepts array of numbers 0-4)


### Event "metadata"
`function(metadata) { }`
Image's header has been parsed, metadata contains this information:
- `width` image size in pixels
- `height` image size in pixels
- `palette` image is paletted
- `color` image is not grayscale
- `alpha` image contains alpha channel
- `interlace` image is interlaced


### Event: "parsed"
`function(data) { }`
Input image has been completly parsed, `data` is complete and ready for modification.


### Event: "error"
`function(error) { }`


### png.parse(data, [callback])
Parses PNG file data. Can be `String` or `Buffer`. Alternatively you can stream data to instance of PNG.

Optional `callback` is once called on `error` or `parsed`. The callback gets
two arguments `(err, data)`.

Returns `this` for method chaining.

+#### Example
```js
new PNG({ filterType:4 }).parse( imageData, function(error, data)
{
	console.log(error, data)
});
```

### png.pack()
Starts converting data to PNG file Stream.

Returns `this` for method chaining.


### png.bitblt(dst, sx, sy, w, h, dx, dy)
Helper for image manipulation, copies a rectangle of pixels from current (i.e. the source) image (`sx`, `sy`, `w`, `h`) to `dst` image (at `dx`, `dy`).

Returns `this` for method chaining.

For example, the following code copies the top-left 100x50 px of `in.png` into dst and writes it to `out.png`:
```js
var dst = new PNG({filterType: -1, width: 100, height: 50});
fs.createReadStream('in.png')
    .pipe(new PNG({
        filterType: -1
    }))
    .on('parsed', function() {
        this.bitblt(dst, 0, 0, 100, 50, 0, 0);
        dst.pack().pipe(fs.createWriteStream('out.png'));
    });
```

### Property: adjustGamma()
Helper that takes data and adjusts it to be gamma corrected. Note that it is not 100% reliable with transparent colours because that requires knowing the background colour the bitmap is rendered on to.

In tests against PNG suite it compared 100% with chrome on all 8 bit and below images. On IE there were some differences.

The following example reads a file, adjusts the gamma (which sets the gamma to 0) and writes it out again, effectively removing any gamma correction from the image.

```js
fs.createReadStream('in.png')
    .pipe(new PNG({
        filterType: -1
    }))
    .on('parsed', function() {
        this.adjustGamma();
        this.pack().pipe(fs.createWriteStream('out.png'));
    });
```

### Property: width
Width of image in pixels


### Property: height
Height of image in pixels


### Property: data
Buffer of image pixel data. Every pixel consists 4 bytes: R, G, B, A (opacity).


### Property: gamma
Gamma of image (0 if not specified)

# Sync API

## PNG.sync

### PNG.sync.read(buffer)

Take a buffer and returns a PNG image. The properties on the image include the meta data and `data` as per the async API above.

```
var data = fs.readFileSync('in.png');
var png = PNG.sync.read(data);
```

### PNG.adjustGamma(src)

Adjusts the gamma of a sync image. See thr async adjustGamma.

```
var data = fs.readFileSync('in.png');
var png = PNG.sync.read(data);
PNG.adjustGamma(png);
```


Changelog
============

### 1.0.0 - 08/08/2015
  - More tests
  - source linted
  - maintainability refactorings
  - async API - exceptions in reading now emit warnings
  - documentation improvement - sync api now documented, adjustGamma documented
  - breaking change - gamma chunk is now written. previously a read then write would destroy gamma information, now it is persisted.

### 0.0.3 - 03/08/2015
  - Error handling fixes
  - ignore files for smaller npm footprint

### 0.0.2 - 02/08/2015
  - Bugfixes to interlacing, support for transparent colours

### 0.0.1 - 02/08/2015
  - Initial release, see pngjs for older changelog.

License
=========

(The MIT License)

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
