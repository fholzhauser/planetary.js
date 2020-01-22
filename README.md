Planetary.js
============

Planetary.js is a JavaScript library for building awesome interactive globes, like this one:

![Planetary.js Screenshot](screenshot.png)

Planetary.js is based on [D3.js](http://d3js.org/) and [TopoJSON](https://github.com/mbostock/topojson). It has built-in support for zoom, rotation, mouse interaction, and displaying animated "pings" at any coordinate. Via plugins, Planetary.js can be extended to do whatever you want!

Examples, documentation, and more can be found at [planetaryjs.com](http://planetaryjs.com/).

Requirements
------------

* [D3](http://d3js.org/) version 3
* [TopoJSON](https://github.com/mbostock/topojson) version 1

Download
--------

Download Planetary.js from the [Planetary.js web site](http://planetaryjs.com/download/).

Quick Start
-----------

You'll need to run this page from a web server of some kind so that Planetary.js can load the TopoJSON data via Ajax.

HTML:

```html
<html>
<head>
  <script type='text/javascript' src='http://d3js.org/d3.v3.min.js'></script>
  <script type='text/javascript' src='http://d3js.org/topojson.v1.min.js'></script>
  <script type='text/javascript' src='planetaryjs.min.js'></script>
</head>
<body>
  <canvas id='globe' width='500' height='500'></canvas>
  <script type='text/javascript' src='yourApp.js'></script>
</body>
</html>
```

JavaScript (`yourApp.js`):

```javascript
var planet = planetaryjs.planet();
// You can remove this statement if `world-110m.json`
// is in the same path as the HTML page:
planet.loadPlugin(planetaryjs.plugins.earth({
  topojson: { file: 'http/path/to/world-110m.json' }
}));
// Make the planet fit well in its canvas
planet.projection.scale(250).translate([250, 250]);
var canvas = document.getElementById('globe');
planet.draw(canvas);
```

Congratulations! You've rendered your first globe.

Added functionality (ofrohn)
-------------------

A simple plugin to expose mouse actions  

```js
  // This plugin exposes mouse actions.
  function mouse(options) {
    options = options || {};
    var noop = function() {},
        onClick = options.onClick || noop,
        onMousedown = options.onMousedown || noop,
        onMousemove = options.onMousemove || noop,
        onMouseup = options.onMouseup || noop;


    return function(planet) {
   
      planet.onInit(function() {
         d3.select(planet.canvas).on('click', onClick.bind(planet));
         d3.select(planet.canvas).on('mousedown', onMousedown.bind(planet));
         d3.select(planet.canvas).on('mousemove', onMousemove.bind(planet));
         d3.select(planet.canvas).on('mouseup', onMouseup.bind(planet));
      });

    };
  };
```

A plugin to display and remove geomarkers on a globe

```js
  planetaryjs.plugins.markers = function (config) {
    var marks = [];
    config = config || {};

    var addMark = function(lng, lat, options) {
      options = options || {};
      options.color = options.color || config.color || 'white';
      options.size = options.size || config.size || 5;
      var mark = { options: options };
      if (config.latitudeFirst) {
        mark.lat = lng;
        mark.lng = lat;
      } else {
        mark.lng = lng;
        mark.lat = lat;
      }
      marks.push(mark);
    };

    var removeMark = function(lng, lat) {
      if (lng === "*") {
        marks = [];
        return;
      }    
      if (arguments.length === 1 && lng < marks.length) {
        marks.splice(lng, 1);
        return;
      }
      if (arguments.length === 2) {
        for (var i=0; i <= marks.length; i++) {
          if (marks[i].lng === lng && marks[i].lat === lat) {
            marks.splice(i, 1);
            return;  
          }
        }
      }
    }

    var drawMarks = function(planet, context) {
      for (var i = 0; i < marks.length; i++) {
        var mark = marks[i];
        var rot = planet.projection.rotate(),
            center = [-rot[0], -rot[1]];
        //draw only if distance from center < 90deg
        var dist = d3.geo.distance(center, [mark.lng, mark.lat]);
        if (dist <= Math.PI/2) drawMark(planet, context, mark);
      }
    };

    var drawMark = function(planet, context, mark) {
      var color = mark.options.color,
          size = mark.options.size * 5,
          pos = planet.projection([mark.lng, mark.lat]);
          
      context.fillStyle = color;
      context.beginPath();
      context.moveTo(pos[0], pos[1]);
      context.arc(pos[0], pos[1] - size, size/2, Math.PI, 0);
      context.fill();
      context.fillStyle = "#fff";
      context.beginPath();
      context.arc(pos[0], pos[1]- size, size/4, 0, 2 * Math.PI);
      context.fill();
    };

    return function (planet) {
      planet.plugins.markers = {
        add: addMark,
        remove: removeMark
      };

      planet.onDraw(function() {
        planet.withSavedContext(function(context) {
          drawMarks(planet, context);
        });
      });
    };
  };
```

Plugin to show concentric hemispheres with increasing opacities,  
specifically to show the dark side with increasing levels of twilight.  
```js
  planetaryjs.plugins.hemisphere = function(options) {
    var pos = {},
        options = options || {};
        
    options.color = options.color || 'black';
    options.alpha = options.alpha || 0.12;
    options.sun = options.sun || 1;

    var setOrigin = function(lng, lat) {
      pos.lng = lng;
      pos.lat = lat;
    };

    var drawHemisphere = function(context, planet, pos) {
      if (options.sun > 0) {
        context.fillStyle = "#ff0";
        context.lineStyle = "#000";
        var circle = d3.geo.circle().origin([pos.lng + 180, -pos.lat]).angle(1)();
        context.beginPath();
        planet.path.context(context)(circle);
        context.fill();
        context.stroke();
      }
      context.fillStyle = options.color;
      context.globalAlpha = options.alpha;

      for (var i = 0; i <= 3; i++) {
        circle = d3.geo.circle().origin([pos.lng, pos.lat]).angle(90 - i*6)();
        context.beginPath();
        planet.path.context(context)(circle);
        context.fill();
      }
    };

    return function(planet) {
      planet.plugins.hemisphere = {
        origin: setOrigin
      };
      planet.onInit(function() {});
      planet.onDraw(function() {
        if (!pos.hasOwnProperty("lat")) return;
        planet.withSavedContext(function(context) {
          drawHemisphere(context, planet, pos);
        });
      });
    };
  };
```

Display coordinate grid with 10° lat/long spacing  
```js
  function grid(config) {
    config = config || {};
    config.color = config.color || '#999';
    config.lineWidth = config.lineWidth || 0.25;

    return function(planet) {
      var graticule = null;
      
      planet.onInit(function() {
        graticule = d3.geo.graticule();
      });

      planet.onDraw(function() {
        planet.withSavedContext(function(context) {
          context.beginPath();
          planet.path(graticule());
          context.strokeStyle = config.color;
          context.lineWidth = config.lineWidth;
          context.stroke();
        });
      });
    };
  };
```

Documentation
-------------

In-depth documentation can be found at [planetaryjs.com](http://planetaryjs.com).

Building
--------

Building the project requires [Node.js](http://nodejs.org/). Once you've installed the project's dependencies with `npm install`, you can build the JavaScript to the `dist` directory with `npm run build`.

License
-------

Planetary.js is licensed under the MIT license. See the `LICENSE` file for more information.
