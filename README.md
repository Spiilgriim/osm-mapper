# osm-mapper

> OSM Mapper is part of the rando-senior project. This tool allows you to to give roads difficulties that can be exported as csv.  
> You can also import correctly formated CSV (see below) to add data to the current map.

## Map

The map is obtained from [Open Street Map](https://wiki.openstreetmap.org/wiki/Main_Page) data. The search bar requests are made with [nominatim](https://wiki.openstreetmap.org/wiki/Nominatim). OSM's data are filtered before being rendered: are not displayed:
* buildings
* Highways with access tagged as private

## CSV

The correct format for CSV is `,` or `;` separated elements and `\n` separated rows. The upload will run and display what it can extract from the given CSV without to find errors so if the input format is not correct, the parsing might not work and will display no error. A CSV generated with this toll will be in the right format to be read by this tool.

## Build Setup

``` bash
# install dependencies
npm install

# serve with hot reload at localhost:8080
npm run dev

# build for production with minification
npm run build
```

For detailed explanation on how things work, consult the [docs for vue-loader](http://vuejs.github.io/vue-loader).

## Usage

To go to a specific place simply write the name of the place in the search bar and if the place exists in OSM's data the map will display it. The refresh button loads OSM data corresponding to the current map view (I'd advice not to use it for a too big zone, it might be very long). Upload and download buttons allow you to use CSV. You can pick color from the bottom of the page (just click on it and all the edges you will click on afterwards will take this color). Reloading the page will loose you data, use the refresh button and don't panick retrieving data can be a bit long sometimes.