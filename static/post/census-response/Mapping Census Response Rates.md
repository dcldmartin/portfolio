
# Mapping Census Response Rates

I'm exploring different tools for mapping. I like [QGIS](http://www.qgis.org/en/site/), but it's not particularly reproducible. I'm also interested in working in D3 to produce good visualizations for the web but don't want to get too deep into the weeds of building my own custom tools.

Enter [a series of Medium posts](https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c) from [Mike Bostock](https://medium.com/@mbostock), walking through a set of tools he built for working with D3 from the command line.

The basic idea: abstract most of the operations you'd want into a small set of tools that you can use from the command line. You end up with a bash script that you can recombine, automate and reproduce - especially with [make files](https://bost.ocks.org/mike/make/).

The main innovation is to introduce 'new-line delimited JSON'. Rather than spitting out a super long blob, you can split it into discrete meaningful elements, making the code more human readable, coding and trobleshooting more interactively.

The below is just my effort to walk through this working with my own dataset. I'm using TIGER shapefiles from the census (New York state census tracts, circa 2010) and visualizing tract-level response rates to the 2010 census. Why does this matter? That's another story.

There's a bunch of dependencies here. Look at Bostock's [original post](https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c) for a list - the tools rely on Node under hood.


```bash
# Set the notebook-level working directory
%cd FILEPATH

# Download and unzip NY State shapefile from TIGER
curl 'ftp://ftp2.census.gov/geo/tiger/GENZ2010/gz_2010_36_140_00_500k.zip' -o 'ny_counties.zip'

unzip -o ny_counties.zip
```



* 1) ```shp2json``` converts the SHP file to JSON. In this case, the ```-n``` flag specifies newline delimited output with each feature as its own line, since I want to pipe it into the next command.

* 2) I use ```d3-geo-projection``` to specify a projection

* 3) ```ndjson-map``` maps the given function to each line. In this case, we're just generating a new id property that gives a unique ID for each tract in the shapefile.

NOTE: At any point, you can visualize the SHP or JSON file (but not NDJSON) by dropping it into [Mapshaper](http://mapshaper.org/)


```bash

shp2json gz_2010_36_140_00_500k.shp \
| geoproject 'd3.geoTransverseMercator()
  .rotate([74 + 30 / 60, -38 - 50 / 60]).fitSize([960, 960], d)' \
| ndjson-split 'd.features' \
| ndjson-map 'd.id = d.properties.GEO_ID.slice(11), d' \
  > ny-projected-id.ndjson
```

* 4) Pull down the Census Planning Database data. There may be a cleaner way to automate this, but this is how I implemented it, given the way the census API organizes this data. Every county has its own file. I download them all separately.

* 5) Using ```find```, I grab the JSON files in the new directory. ```ndjson-cat``` concatenates each of these into a single file. I want to clean this up a bit, so in order to use these command-line tools, so I use ```ndjson-split``` to create a newline-delimited file, with each line corresponding to a county. And for each of these, I create new features, pulling out an id, county name, population, census response rate (technically, the Mail Return Rate), among other things and output the edited JSON into a file. 

In the end we have a new-line delimited file with the features we want (identifiers, county labels and useful data to visualize on the map).


Note: I've included CENSUS_API_KEY as an environment variable. You'll obviously need your own.


```bash

mkdir pdb
curl 'http://api.census.gov/data/2015/pdb/tract?get=County,County_name,Tract,Tot_Population_CEN_2010,Mail_Return_Rate_CEN_2010&for=tract:*&in=state:36+county:[1-123]&key='${CENSUS_API_KEY} -o pdb/pdb_#1.json

cd pdb 

find . -type f -name "pdb_*.json" -size +1c -print0 \
| xargs -0 ndjson-cat > pdb_split.json

ndjson-split 'd.slice(1)' < pdb_split.json \
| ndjson-map '{id: d[0] + d[2], county: d[1], Tot_Population_CEN_2010: + d[3], Mail_Return_Rate_CEN_2010: + d[4], univ_id: +d[5]+d[6]+d[7]}'
> ny_pdb_data.ndjson

```

Now to join these together

* 6) The ```ndjson-join``` tool works like a SQL table join. I've created a property in each file labeled id to match on so that I can merge in the data I just re-formatted.
* 7) The data was joined as its own node in each newline. We'd like to join in the data we created as properties of the main node. ```ndjson-map``` comes in handy for that, grabbing the data from the second node (aka, d[1]).
* 8) ```geo2topo``` converts the GeoJSON into TopoJSON. This accomplishes a few things - for one, it's a more efficient way to render the graphics, for faster load times. For another, we're able to create object types that, later on in the process we can edit separately. Here I specify the input as 'tracts', which is the level of aggregation for the data, but I'd also like to show county lines, which will let the reader locate specific areas somewhat more easily.
* 9) ```toposimplify``` and ```topoquantize``` provide further efficiency gains (Bostock gives [a good explanation of the difference](https://stackoverflow.com/questions/18900022/topojson-quantization-vs-simplification)). Quantization removes some precision on individual points, stripping some of the decimal places from each coordinate and simplification removes some of the points, the result being arcs that can be described more simply, without too much loss in fidelity. tl;dr: these are just compression algorithms, trading off some precision and fidelity to the original image for something that takes much less time to load and still looks pretty good for our purposes.
* 10) ```topomerge``` lets me do a couple things: In the first case, the ```-k``` flag lets me specify a new way to group objects and create a key using an arbitrary expression. Below, I'm taking all the objects that I'd specified as tracts when I converted to TopoJSON and aggregating them into counties. The key I create is just the digits in the ID that identify what county the tract is in.
* 11) Second, I can create a mesh of lines instead of merging the polygons. Below I've specified the ```-f``` flag to filter some lines out based on the expression, and I'm pulling the lines from the 'counties' polygons I just created. The expression indicates that I'm only showing internal borders - not those on the coasts. In terms of the expression, arcs that aren't adjacent to others.


```bash

ndjson-join 'd.id' \
  ny-projected-id.ndjson \
  pdb/ny_pdb_data.ndjson \
| ndjson-map 'd[0].properties = {Tot_Population_CEN: d[1].Tot_Population_CEN_2010, Mail_Return_Rate_CEN: d[1].Mail_Return_Rate_CEN_2010, county: d[1].county}, d[0]' \
> pdb_joined.ndjson

# As an interim check - just reduce this here and map it to see if what you've got looks ok
    ndjson-reduce \
      < pdb_joined.ndjson \
      | ndjson-map '{type: "FeatureCollection", features: d}' \
      > ny_pdb_map.json

geo2topo -n \
  tracts=pdb_joined.ndjson \
| toposimplify -p 1 -f \
| topoquantize 1e5 \
| topomerge -k 'd.id.slice(0, 3)' counties=tracts \
| topomerge --mesh -f 'a !== b' counties=counties \
> ny_pdb_topo.json
```

* 12) Here I add the fill. I've got the data aggregated at the tract level so I filter to include just the 'tracts' objects in ```topo2geo```. 
* 13) I use ```ndjson-map``` to specify a fill value for each tract object. This takes a more complex d3 function. I specify that the function requires d3 and d3-scale-chromatic modules and I create a color ramp across discrete values of the variable I'm displaying - the census response rates.
* 14) ```ndjson-split``` splits the file back into separate lines, where each line is a distinct feature.
* 15) Likewise, I take the counties separately - the mesh of internal county borders - and specify the line properties.
* 16) Finally, the ```geo2svg``` tool converts the geojson into an SVG.


```bash
(topo2geo tracts=- \
    < ny_pdb_topo.json \
    | ndjson-map -r d3 -r d3=d3-scale-chromatic 'z = d3.scaleThreshold().domain([0, 1, 60, 70, 75, 80, 85, 90]).range(d3.schemeOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.Mail_Return_Rate_CEN)), d' \
    | ndjson-split 'd.features'; \
topo2geo counties=- \
    < ny_pdb_topo.json \
    | ndjson-map 'd.properties = {"stroke": "#000", "stroke-opacity": 0.3}, d')\
| geo2svg -n --stroke none -p 1 -w 960 -h 960 \
> ny_pdb.svg
```

The result:
https://cdn.rawgit.com/dcldmartin/CensusResponse_SocialCapital/47e9a619/ny_pdb.svg
