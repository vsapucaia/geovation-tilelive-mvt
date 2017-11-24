Setup a local server to serve and consume vector tiles.

Based on:

https://geovation.github.io/build-your-own-static-vector-tile-pipeline

# Requirements

* Tippecanoe
* ogr2ogr

certify that you have the requirements:
```
ogr2ogr --version
tippecanoe -v
```

# How to build


```
export LAYER=Road
mkdir -p build/www/tiles/
cat src/opmplc_essh_tq_chunks* > opmplc_essh_tq.zip
find "src" | grep ".zip" | xargs -n1 unzip -d "build/shapefiles"
find "build/shapefiles" | grep "_${LAYER}.shp" > "build/${LAYER}_shapefiles.txt"
ogr2ogr -f 'ESRI Shapefile' "build/$LAYER.shp" -nln "${LAYER}" "`cat "build/${LAYER}_shapefiles.txt" | head -n 1`"
cat "build/${LAYER}_shapefiles.txt" | tail -n +2 | while read line; do
    ogr2ogr -f 'ESRI Shapefile' -update -append "build/$LAYER.shp" "$line" -nln "${LAYER}"
done
mkdir build/geojson
ogr2ogr -dim 2 -f 'GeoJSON' -s_srs "+proj=tmerc +lat_0=49 +lon_0=-2 +k=0.999601 +x_0=400000 +y_0=-100000 +ellps=airy +units=m +no_defs +nadgrids=./src/OSTN02_NTv2.gsb" -t_srs 'EPSG:4326' "build/geojson/$LAYER.json" "build/$LAYER.shp"
tippecanoe --no-feature-limit --no-tile-size-limit --exclude-all --minimum-zoom=5 --maximum-zoom=g --output-to-directory "build/www/tiles" `find ./build/geojson -type f | grep .json`
```

```
cat << EOF > "build/www/index.html"
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Map</title>
  <meta name="viewport" content="initial-scale=1,maximum-scale=1,user-scalable=no" />
  <script src="https://api.mapbox.com/mapbox-gl-js/v0.35.1/mapbox-gl.js"></script>
  <link href="https://api.mapbox.com/mapbox-gl-js/v0.35.1/mapbox-gl.css" rel="stylesheet" />
  <style>
    body { margin:0; padding:0; }
    #map { position:absolute; top:0; bottom:0; width:100%; }
  </style>
</head>
<body>
  <div id="map"></div>
  <script>
    mapboxgl.accessToken = "not-needed-unless-using-mapbox-styles";
    var map = new mapboxgl.Map({
        container: "map",
        style: "http://localhost:8000/style.json",
        center: [-0.1047, 51.5236],
        // These affect the hard limits of the zoom controls
        minZoom: 5,
        zoom: 11,
        maxZoom: 22
    });
    map.addControl(new mapboxgl.NavigationControl());
  </script>
</body>
</html>
EOF
cat << EOF > "build/www/style.json"
{
  "version": 8,
  "name": "Custom",
  "metadata": {
    "mapbox:autocomposite": true
  },
  "glyphs": "mapbox://fonts/mapbox/{fontstack}/{range}.pbf",
  "sources": {
    "composite": {
      "type": "vector",
      "tiles": ["http://localhost:8000/tiles/{z}/{x}/{y}.pbf"],
      "minzoom": 0,
      "maxzoom": 15
    }
  },
  "layers": [
    {
      "id": "background",
      "type": "background",
      "paint": {
        "background-color": "#e3decb"
      }
    },
    {
      "id": "Road",
      "type": "line",
      "source": "composite",
      "source-layer": "Road",
      "minzoom": 0,
      "maxzoom": 22,
      "paint": {
          "line-color": "#FFFFFF",
          "line-width": 0.5
      }
    }
  ]
}
EOF
```


# Start the server

```
npm install live-server

export PATH=$PWD/node_modules/.bin:$PATH
cat << EOF > "build/gzip.js"
module.exports = function(req, res, next) {
  if (req.url.endsWith('.pbf')) {
    next();
    res.setHeader('Content-Encoding', 'gzip');
  } else {
    next();
  }
}
EOF

live-server --port=8000 --middleware="${PWD}/build/gzip.js" --host=localhost build/www
```
