<template>
  <div id="mapwrapper">
    <h1>{{ data }}</h1>
    <div id="osmmap"></div>
  </div>
</template>

<script>
var leaflet = require("leaflet");
export default {
  name: "maposm",
  data() {
    return {
      data: "",
      mymap: null,
      currentNodeList: {},
      currentEdgeList: {}
    };
  },
  methods: {
    updateOSMData: function() {
      let request = new XMLHttpRequest();
      let bounds = this.mymap.getBounds();
      let apiUrl =
        "http://overpass-api.de/api/interpreter?data=[bbox][out:json];(node;way;);out;&bbox=" +
        bounds.getWest() +
        "," +
        bounds.getSouth() +
        "," +
        bounds.getEast() +
        "," +
        bounds.getNorth();
      let nodeList = {};
      let edgeList = {};
      request.open("GET", apiUrl, true);
      request.onload = function(vueMap) {
        let osmResponse = JSON.parse(this.response).elements;
        console.log(osmResponse);
        for (let i = 0; i < osmResponse.length; i++) {
          if (osmResponse[i].type == "node") {
            nodeList[osmResponse[i].id] = {
              lat: osmResponse[i].lat,
              lon: osmResponse[i].lon
            };
          }
        }
        for (let i = 0; i < osmResponse.length; i++) {
          if (osmResponse[i].type == "way" && osmResponse[i].tags && !osmResponse[i].tags.building && osmResponse[i].tags.highway && osmResponse[i].tags.access != 'private') {
            edgeList[osmResponse[i].id] = osmResponse[i].nodes;
            for (let j = 1; j < osmResponse[i].nodes.length; j++) {
              if (osmResponse[i].nodes[j] in nodeList && osmResponse[i].nodes[j-1] in nodeList){
                let polyline = leaflet.polyline([
                [
                  nodeList[osmResponse[i].nodes[j]].lat,
                  nodeList[osmResponse[i].nodes[j]].lon
                ],
                [
                  nodeList[osmResponse[i].nodes[j - 1]].lat,
                  nodeList[osmResponse[i].nodes[j - 1]].lon
                ]
              ]).addTo(vueMap.mymap);
              polyline.addEventListener('click', () => console.log(osmResponse[i]));
              }
            }
          }
        }
        vueMap.currentNodeList = nodeList;
        vueMap.currentEdgeList = edgeList;
      }.bind(request, this);

      request.send();
    }
  },
  mounted() {
    this.mymap = leaflet.map("osmmap").setView([50.7, 7.15], 15);
    leaflet
      .tileLayer(
        "https://api.tiles.mapbox.com/v4/{id}/{z}/{x}/{y}.png?access_token={accessToken}",
        {
          attribution:
            'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors, <a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
          maxZoom: 18,
          id: "mapbox.streets",
          accessToken:
            "pk.eyJ1Ijoic3BpaWxncmlpbSIsImEiOiJjanppZWgxN2gwODNzM2NvOHlmNTl4NDB6In0.7gxi13vVRZI5UMJR8ktV6A"
        }
      )
      .addTo(this.mymap);

    this.updateOSMData();
  }
};
</script>

<style scoped>
#osmmap {
  height: 80vh;
}
</style>