/*
 * Photobooth 4 Shots — portable JavaScript image filter / composer
 * Version 1.0.0 (2026-11-05)
 * MIT License. No external dependencies; no network calls or analytics.
 *
 * This file is suitable for hosting as a GitHub Raw .js file.
 * Note: a Raw URL only serves source code. Your host must execute this JS and
 * pass four captured images via Photobooth4Filter.onCapture(image).
 * Requires a JavaScript environment with Canvas and image decoding support.
 */
(function (root, factory) {
  'use strict';
  var api = factory();
  if (typeof module === 'object' && module.exports) module.exports = api;
  root.Photobooth4 = api;
  root.Photobooth4Filter = api.defaultFilter;
})(typeof globalThis !== 'undefined' ? globalThis : this, function () {
  'use strict';

  function canvasFor(width, height) {
    var canvas;
    if (typeof document !== 'undefined' && document.createElement) {
      canvas = document.createElement('canvas');
      canvas.width = width;
      canvas.height = height;
    } else if (typeof OffscreenCanvas !== 'undefined') {
      canvas = new OffscreenCanvas(width, height);
    } else {
      throw new Error('Photobooth4 can su dung Canvas/OffscreenCanvas. May chu JS khong ho tro xu ly anh.');
    }
    return canvas;
  }

  function ctx2d(canvas, willRead) {
    var ctx = canvas.getContext('2d', { willReadFrequently: !!willRead });
    if (!ctx) throw new Error('Khong tao duoc Canvas 2D context.');
    return ctx;
  }

  function canvasBlob(canvas) {
    if (typeof canvas.convertToBlob === 'function') {
      return canvas.convertToBlob({ type: 'image/png' });
    }
    if (typeof canvas.toBlob === 'function') {
      return new Promise(function (resolve, reject) {
        canvas.toBlob(function (blob) {
          if (blob) resolve(blob);
          else reject(new Error('Khong the xuat anh PNG.'));
        }, 'image/png');
      });
    }
    throw new Error('Moi truong Canvas khong ho tro xuat PNG.');
  }

  function loadImageUrl(url) {
    if (typeof Image === 'undefined') {
      return Promise.reject(new Error('Moi truong khong ho tro giai ma anh tu URL.'));
    }
    return new Promise(function (resolve, reject) {
      var image = new Image();
      image.crossOrigin = 'anonymous';
      image.onload = function () { resolve(image); };
      image.onerror = function () { reject(new Error('Khong the mo anh (kiem tra URL/CORS).')); };
      image.src = url;
    });
  }

  async function decodeFrame(input) {
    if (!input) throw new Error('Khong co du lieu anh.');
    if (typeof input === 'string') return loadImageUrl(input);
    if (typeof Blob !== 'undefined' && input instanceof Blob) {
      if (typeof createImageBitmap === 'function') return createImageBitmap(input);
      if (typeof URL === 'undefined' || typeof URL.createObjectURL !== 'function') {
        throw new Error('Khong co bo giai ma anh Blob.');
      }
      var url = URL.createObjectURL(input);
      try { return await loadImageUrl(url); }
      finally { URL.revokeObjectURL(url); }
    }
    if (input.blob && typeof Blob !== 'undefined' && input.blob instanceof Blob) {
      return decodeFrame(input.blob);
    }
    if (input.dataURL && typeof input.dataURL === 'string') return decodeFrame(input.dataURL);
    if (input.src && typeof input.src === 'string' && !input.width && !input.naturalWidth) {
      return decodeFrame(input.src);
    }
    // HTMLImageElement, canvas, ImageBitmap, Video, OffscreenCanvas, etc.
    var w = input.naturalWidth || input.videoWidth || input.width;
    var h = input.naturalHeight || input.videoHeight || input.height;
    if (w > 0 && h > 0) return input;
    throw new Error('Dinh dang dau vao khong ho tro. Hay dua Blob, URL, Image, video hoac canvas.');
  }

  function dims(image) {
    return {
      width: image.naturalWidth || image.videoWidth || image.width,
      height: image.naturalHeight || image.videoHeight || image.height
    };
  }

  function drawCover(ctx, image, x, y, w, h) {
    var size = dims(image);
    var sourceRatio = size.width / size.height;
    var destRatio = w / h;
    var sx = 0, sy = 0, sw = size.width, sh = size.height;
    if (sourceRatio > destRatio) {
      sw = size.height * destRatio;
      sx = (size.width - sw) / 2;
    } else {
      sh = size.width / destRatio;
      sy = (size.height - sh) / 2;
    }
    ctx.drawImage(image, sx, sy, sw, sh, x, y, w, h);
  }

  function roundedRectPath(ctx, x, y, w, h, radius) {
    var r = Math.max(0, Math.min(radius, w / 2, h / 2));
    ctx.beginPath();
    ctx.moveTo(x + r, y);
    ctx.arcTo(x + w, y, x + w, y + h, r);
    ctx.arcTo(x + w, y + h, x, y + h, r);
    ctx.arcTo(x, y + h, x, y, r);
    ctx.arcTo(x, y, x + w, y, r);
    ctx.closePath();
  }

  // Parse standard 3D .cube: numeric triplets ordered with R varying fastest.
  function parseCube(text) {
    if (typeof text !== 'string') throw new Error('cubeText phai la chuoi van ban.');
    var rows = text.replace(/^\uFEFF/, '').split(/\r?\n/);
    var size = 0;
    var min = [0, 0, 0], max = [1, 1, 1];
    var data = [];
    for (var i = 0; i < rows.length; i++) {
      var line = rows[i].split('#')[0].trim();
      if (!line) continue;
      var values = line.split(/\s+/);
      var key = values[0].toUpperCase();
      if (key === 'TITLE') continue;
      if (key === 'LUT_3D_SIZE') {
        size = Number(values[1]);
        if (!Number.isInteger(size) || size < 2 || size > 65) {
          throw new Error('LUT_3D_SIZE phai nam trong khoang 2..65.');
        }
      } else if (key === 'LUT_1D_SIZE') {
        throw new Error('Chi ho tro file LUT 3D (.cube), khong ho tro LUT 1D.');
      } else if (key === 'DOMAIN_MIN' || key === 'DOMAIN_MAX') {
        var triplet = values.slice(1, 4).map(Number);
        if (triplet.length !== 3 || triplet.some(function (v) { return !Number.isFinite(v); })) {
          throw new Error('DOMAIN_MIN/MAX khong hop le.');
        }
        if (key === 'DOMAIN_MIN') min = triplet;
        else max = triplet;
      } else if (values.length === 3 && values.every(function (v) { return Number.isFinite(Number(v)); })) {
        if (!size) throw new Error('Thieu LUT_3D_SIZE truoc du lieu mau.');
        data.push(Number(values[0]), Number(values[1]), Number(values[2]));
      } else {
        throw new Error('Dong .cube khong hop le (dong ' + (i + 1) + '): ' + line.slice(0, 80));
      }
    }
    if (!size || data.length !== size * size * size * 3) {
      throw new Error('Du lieu LUT sai: can ' + (size * size * size) + ' dong RGB, nhan ' + (data.length / 3) + '.');
    }
    if (min.some(function (v, i) { return !(max[i] > v); })) {
      throw new Error('DOMAIN_MAX phai lon hon DOMAIN_MIN.');
    }
    return { size: size, min: min, max: max, data: Float32Array.from(data) };
  }

  function clamp(v, a, b) { return Math.max(a, Math.min(b, v)); }

  function applyCubeToCanvas(canvas, cube) {
    var ctx = ctx2d(canvas, true);
    var img = ctx.getImageData(0, 0, canvas.width, canvas.height);
    var p = img.data, n = cube.size, data = cube.data;
    var mins = cube.min, maxs = cube.max;
    var scale = [
      (n - 1) / (maxs[0] - mins[0]),
      (n - 1) / (maxs[1] - mins[1]),
      (n - 1) / (maxs[2] - mins[2])
    ];
    function sample(ri, gi, bi, c) {
      return data[(((bi * n + gi) * n + ri) * 3) + c];
    }
    for (var i = 0; i < p.length; i += 4) {
      if (!p[i + 3]) continue;
      var r = clamp((p[i] / 255 - mins[0]) * scale[0], 0, n - 1);
      var g = clamp((p[i + 1] / 255 - mins[1]) * scale[1], 0, n - 1);
      var b = clamp((p[i + 2] / 255 - mins[2]) * scale[2], 0, n - 1);
      var r0 = Math.floor(r), g0 = Math.floor(g), b0 = Math.floor(b);
      var r1 = Math.min(n - 1, r0 + 1), g1 = Math.min(n - 1, g0 + 1), b1 = Math.min(n - 1, b0 + 1);
      var tr = r - r0, tg = g - g0, tb = b - b0;
      for (var c = 0; c < 3; c++) {
        var v00 = sample(r0, g0, b0, c) * (1 - tr) + sample(r1, g0, b0, c) * tr;
        var v10 = sample(r0, g1, b0, c) * (1 - tr) + sample(r1, g1, b0, c) * tr;
        var v01 = sample(r0, g0, b1, c) * (1 - tr) + sample(r1, g0, b1, c) * tr;
        var v11 = sample(r0, g1, b1, c) * (1 - tr) + sample(r1, g1, b1, c) * tr;
        var v0 = v00 * (1 - tg) + v10 * tg;
        var v1 = v01 * (1 - tg) + v11 * tg;
        p[i + c] = Math.round(clamp((v0 * (1 - tb) + v1 * tb) * 255, 0, 255));
      }
    }
    ctx.putImageData(img, 0, 0);
  }

  var DEFAULTS = {
    layout: 'strip',            // 'strip' = 4 photos vertical, 'grid' = 2x2 square
    width: 900,
    background: '#fffaf2',
    textColor: '#31251c',
    margin: 48,
    gap: 18,
    radius: 12,
    footer: 'PHOTOBOOTH  ·  4 SHOTS',
    frameAspect: 1.5,         // photo width / height (strip mode)
    cubeText: null,
    onProgress: null,
    onComplete: null
  };

  function create(options) {
    var settings = Object.assign({}, DEFAULTS, options || {});
    if (settings.layout !== 'grid' && settings.layout !== 'strip') {
      throw new Error('layout phai la "strip" hoac "grid".');
    }
    if (!Number.isInteger(settings.width) || settings.width < 320 || settings.width > 2400) {
      throw new Error('width phai la so nguyen tu 320 den 2400.');
    }
    if (!(settings.frameAspect > 0)) throw new Error('frameAspect phai lon hon 0.');
    var lut = settings.cubeText ? parseCube(settings.cubeText) : null;
    var frames = [];
    var output = null;
    var busy = false;

    function render() {
      if (frames.length !== 4) throw new Error('Can du 4 anh de ghep Photobooth.');
      var width = settings.width;
      var margin = Math.round(settings.margin);
      var gap = Math.round(settings.gap);
      var footerHeight = 74;
      var contentWidth = width - margin * 2;
      if (contentWidth <= gap * 2) throw new Error('margin/gap qua lon.');
      var grid = settings.layout === 'grid';
      var photoW = grid ? Math.floor((contentWidth - gap) / 2) : contentWidth;
      var photoH = grid ? Math.round(photoW * 0.9) : Math.round(photoW / settings.frameAspect);
      var rows = grid ? 2 : 4;
      var height = margin * 2 + rows * photoH + (rows - 1) * gap + footerHeight;
      var outputCanvas = canvasFor(width, height);
      var ctx = ctx2d(outputCanvas);
      ctx.fillStyle = settings.background;
      ctx.fillRect(0, 0, width, height);

      for (var i = 0; i < 4; i++) {
        var col = grid ? i % 2 : 0;
        var row = grid ? Math.floor(i / 2) : i;
        var x = margin + col * (photoW + gap);
        var y = margin + row * (photoH + gap);
        ctx.save();
        roundedRectPath(ctx, x, y, photoW, photoH, settings.radius);
        ctx.clip();
        drawCover(ctx, frames[i], x, y, photoW, photoH);
        ctx.restore();
      }
      ctx.fillStyle = settings.textColor;
      ctx.font = 'bold ' + Math.max(15, Math.round(width * 0.025)) + 'px Arial, sans-serif';
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';
      ctx.fillText(String(settings.footer || ''), width / 2, height - margin - footerHeight / 2, width - 2 * margin);
      return outputCanvas;
    }

    async function addFrame(input) {
      if (busy) throw new Error('Anh truoc dang xu ly; hay gui anh ke tiep sau khi hoan thanh.');
      if (frames.length >= 4) throw new Error('Da du 4 anh. Goi reset() de bat dau dot moi.');
      busy = true;
      var decoded;
      try {
        decoded = await decodeFrame(input);
        var snap = canvasFor(1000, 750);
        drawCover(ctx2d(snap), decoded, 0, 0, snap.width, snap.height);
        if (lut) applyCubeToCanvas(snap, lut);
        frames.push(snap);
        if (settings.onProgress) settings.onProgress(frames.length, 4);
        if (frames.length === 4) {
          var outCanvas = render();
          var blob = await canvasBlob(outCanvas);
          output = {
            blob: blob,
            mimeType: 'image/png',
            width: outCanvas.width,
            height: outCanvas.height
          };
          if (settings.onComplete) settings.onComplete(output);
        }
        return { count: frames.length, total: 4, complete: frames.length === 4, result: output };
      } finally {
        if (decoded && decoded !== input && typeof decoded.close === 'function') decoded.close();
        busy = false;
      }
    }

    return {
      addFrame: addFrame,
      getState: function () { return { count: frames.length, total: 4, complete: !!output }; },
      getResult: function () { return output; },
      reset: function () { if (busy) throw new Error('Dang xu ly anh.'); frames = []; output = null; },
      setCube: function (cubeText) { if (busy) throw new Error('Dang xu ly anh.'); lut = cubeText ? parseCube(cubeText) : null; },
      getCanvas: function () { return output ? render() : null; }
    };
  }

  // Generic host adapter: your Locket integration can call onCapture(image)
  // 4 times and then use response.result.blob when complete === true.
  // Does NOT inject itself into, or modify, the official Locket app.
  var defaultCubeText = null;
  var defaultSession = create();
  var defaultFilter = {
    id: 'photobooth-4',
    name: 'Photobooth 4 Shots',
    type: 'multi-capture',
    capturesRequired: 4,
    outputMimeType: 'image/png',
    onCapture: function (image) { return defaultSession.addFrame(image); },
    onReset: function () { defaultSession.reset(); },
    getState: function () { return defaultSession.getState(); },
    setCube: function (cubeText) { defaultSession.setCube(cubeText); defaultCubeText = cubeText; },
    setLayout: function (layout) {
      if (layout !== 'strip' && layout !== 'grid') throw new Error('layout khong hop le.');
      defaultSession = create({ layout: layout, cubeText: defaultCubeText });
    }
  };

  return { version: '1.0.0', create: create, parseCube: parseCube, defaultFilter: defaultFilter };
});
