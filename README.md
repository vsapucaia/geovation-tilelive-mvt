Setup a local server to serve and consume vector tiles.


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
