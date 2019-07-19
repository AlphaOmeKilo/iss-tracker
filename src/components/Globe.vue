<template>
  <div class="globe-container">
    <canvas id="globe"></canvas>
  </div>
</template>

<script>
import * as d3 from 'd3';
import * as topojson from 'topojson';
import { setInterval } from 'timers';
export default {
  name: 'Globe',
  data: function() {
      return {
        rotationDelay: 3000,
        scaleFactor: 1,
        degPerSec: 6,
        angles: { x: 0, y: 0, z: 0},
        colorWater: '#fff',
        colorLand: '#111',
        colorGraticule: '#ccc',
        colorCountry: '#a00',
        colorMarker: '#ff0000',
        canvas: null,
        context: null,
        water: {type: 'Sphere'},
        projection: d3.geoOrthographic().precision(0.1),
        graticule: d3.geoGraticule10(),
        path: null,
        v0: null, // Mouse position in Cartesian coordinates at start of drag gesture.
        r0: null, // Projection rotation as Euler angles at start.
        q0: null, // Projection rotation as versor at start.
        lastTime: d3.now(),
        degPerMs: 0.018,
        width: null, 
        height: null,
        land: null, 
        countryList: null,
        autorotate: null, 
        now: null, 
        diff: null, 
        roation: null,
        world: null,
        svg: null,
        markerGroup: null,
        location: {},
        center: null
      }
  },
  methods: {
    startRotation(delay) {
        this.autorotate.restart(this.rotate, delay || 0);
    },
    stopRotation() {
        this.autorotate.stop();
    },
    render() {
        this.context.clearRect(0, 0, this.width, this.height);
        this.fill(this.water, this.colorWater);
        this.stroke(this.graticule, this.colorGraticule);
        this.fill(this.land, this.colorLand);

        var self = this;
       
            this.marker(this.location, self.colorMarker);
       
        
        // this.marker(this.locations[1], this.colorMarker);
    },
    marker(location, color) {
        var p = this.projection([location.latitude, location.longitude]);
        var x = p[0];
        var y = p[1];
        this.context.beginPath();
        this.context.arc(x, y, 6, 0, 2 * Math.PI);
        
        var gdistance = d3.geoDistance([location.latitude, location.longitude], this.projection.invert(this.center));
        this.context.fillStyle = location.gdistance < gdistance ? color : 'transparent';

        location.gdistance = gdistance;

        this.context.fill();

    },
    fill(obj, color) {
        this.context.beginPath();
        this.path(obj);
        this.context.fillStyle = color;
        this.context.fill();
    },
    stroke(obj, color) {
        this.context.beginPath();
        this.path(obj);
        this.context.strokeStyle = color;
        this.context.stroke();
    },
    rotate(elapsed) {
        this.now = d3.now();
        this.diff = this.now - this.lastTime;
        if (this.diff < elapsed) {
            this.rotation = this.projection.rotate();
            this.rotation[0] += this.diff * this.degPerMs;
            this.projection.rotate(this.rotation);
            this.render();
        }
        this.lastTime = this.now;
    },
    loadData(cb) {
        d3.json('https://unpkg.com/world-atlas@1/world/110m.json', function(error, world) {
            if (error) throw error;
            d3.tsv('https://gist.githubusercontent.com/mbostock/4090846/raw/07e73f3c2d21558489604a0bc434b3a5cf41a867/world-country-names.tsv', function(error, countries) {
                if (error) throw error;
                cb(world, countries);
            })
        })
    },
    polygonContains(polygon, point) {
        var n = polygon.length;
        var p = polygon[n - 1];
        var x = point[0], y = point[1];
        var x0 = p[0], y0 = p[1];
        var x1, y1;
        var inside = false;
        for (var i = 0; i < n; ++i) {
            p = polygon[i], x1 = p[0], y1 = p[1];
            if (((y1 > y) !== (y0 > y)) && (x < (x0 - x1) * (y - y1) / (y0 - y1) + x1)) inside = !inside;
            x0 = x1, y0 = y1;
        }
        return inside;
    },
    setAngles() {
        var rotation = this.projection.rotate();
        rotation[0] = this.angles.y;
        rotation[1] = this.angles.x;
        rotation[2] = this.angles.z;
        this.projection.rotate(rotation);
      },
      scale() {
        this.width = document.documentElement.clientWidth;
        this.height = document.documentElement.clientHeight;
        this.canvas.attr('width', this.width).attr('height', this.height);
        this.projection
            .scale((this.scaleFactor * Math.min(this.width, this.height)) / 2)
            .translate([this.width / 2, this.height / 2]);
        this.render();
      },
      async fetchData() {
          let data = await d3.json('https://unpkg.com/world-atlas@1/world/110m.json');
          this.world = data;
          this.land = topojson.feature(this.world, this.world.objects.land);

          this.autorotate = d3.timer(this.rotate);
          this.startRotation(this.rotationDelay);
          
          this.scale();

          var self = this;

          setInterval(function() {
              self.getISS()
          }, 1000);
      },
      async getISS() {
          let data = await d3.json('https://api.wheretheiss.at/v1/satellites/25544');
          this.location = data;
          console.log(data.longitude);
      }
  },
  computed: {
      
  },
  mounted() {
    this.canvas = d3.select('#globe');
    this.context = this.canvas.node().getContext('2d');
    this.path = d3.geoPath(this.projection).context(this.context),
    this.setAngles();

    this.svg = d3.select('svg')
                .attr('width', this.width).attr('height', this.height);
    this.markerGroup = this.svg.append('g');

    this.center = [this.width/2, this.height/2];

    this.fetchData();
    window.addEventListener('resize', this.scale);
  }
}
</script>

<style scoped>
    #globe {
        cursor: move;  
    }
</style>
