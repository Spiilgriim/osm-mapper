<template>
  <div id="mapwrapper">
    <div id="navbar">
      <div id="searchbar">
        <input type="text" v-model="locationSearch" />
        <button id="search-button" @click="updateView">search</button>
      </div>
      <button id="refresh-button" @click="updateOSMData">refresh</button>
      <button id="download-button" @click="downloadCSV">Download</button>
    </div>
    <div id="osmmap"></div>
    <div id="color-picker">
      <div id="green-color" @click="changeColorToGreen">
        <div id="green-color-show"></div>
        <p id="green-color-value">Praticable par tous</p>
      </div>
      <div id="orange-color">
        <div id="orange-color-show" @click="changeColorToOrange"></div>
        <p id="orange-color-value">Impraticable en fauteuil</p>
      </div>
      <div id="red-color">
        <div id="red-color-show" @click="changeColorToRed"></div>
        <p id="red-color-value">Impraticable</p>
      </div>
    </div>
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
      currentEdgeList: {},
      locationSearch: "Default Location",
      currentColor: { color: "red", value: "test" },
      savedEdges: {},
      defaultColor: "#3388ff"
    };
  },
  methods: {
    updateOSMData: function() {
      this.clearMap();
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
        for (let i = 0; i < osmResponse.length; i++) {
          if (osmResponse[i].type == "node") {
            nodeList[osmResponse[i].id] = {
              lat: osmResponse[i].lat,
              lon: osmResponse[i].lon
            };
          }
        }
        for (let i = 0; i < osmResponse.length; i++) {
          if (
            osmResponse[i].type == "way" &&
            osmResponse[i].tags &&
            !osmResponse[i].tags.building &&
            osmResponse[i].tags.highway &&
            osmResponse[i].tags.access != "private"
          ) {
            edgeList[osmResponse[i].id] = osmResponse[i].nodes;
            let edgePolyline = [];
            for (let j = 1; j < osmResponse[i].nodes.length; j++) {
              if (
                osmResponse[i].nodes[j] in nodeList &&
                osmResponse[i].nodes[j - 1] in nodeList
              ) {
                let polyline = leaflet
                  .polyline([
                    [
                      nodeList[osmResponse[i].nodes[j]].lat,
                      nodeList[osmResponse[i].nodes[j]].lon
                    ],
                    [
                      nodeList[osmResponse[i].nodes[j - 1]].lat,
                      nodeList[osmResponse[i].nodes[j - 1]].lon
                    ]
                  ])
                  .addTo(vueMap.mymap);
                polyline.on("click", e => {
                  if (
                    [osmResponse[i].nodes[j], osmResponse[i].nodes[j - 1]] in
                    vueMap.savedEdges
                  ) {
                    delete vueMap.savedEdges[
                      [osmResponse[i].nodes[j], osmResponse[i].nodes[j - 1]]
                    ];
                    e.target.setStyle({ color: vueMap.defaultColor });
                  } else {
                    vueMap.savedEdges[
                      [osmResponse[i].nodes[j], osmResponse[i].nodes[j - 1]]
                    ] = vueMap.currentColor.value;
                    e.target.setStyle({ color: vueMap.currentColor.color });
                  }
                });
              }
            }
          }
        }
        vueMap.currentNodeList = nodeList;
        vueMap.currentEdgeList = edgeList;
      }.bind(request, this);

      request.send();
    },
    updateView() {
      let request = new XMLHttpRequest();
      this.clearMap();
      let apiUrl =
        "https://nominatim.openstreetmap.org/search/" +
        this.locationSearch +
        "?format=json";
      request.open("GET", apiUrl, true);
      request.onload = function(vueMap) {
        let osmRequest = JSON.parse(this.response);
        if (osmRequest.length > 0) {
          vueMap.mymap.setView([osmRequest[0].lat, osmRequest[0].lon], 16);
          vueMap.updateOSMData();
        } else {
          vueMap.locationSearch = "Unable to find location";
        }
      }.bind(request, this);
      request.send();
    },
    clearMap() {
      for (let i in this.mymap._layers) {
        if (this.mymap._layers[i].options.format == undefined) {
          try {
            this.mymap.removeLayer(this.mymap._layers[i]);
          } catch (e) {
            console.log("problem with " + e + this.mymap._layers[i]);
          }
        }
      }
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
    },
    changeColorToGreen() {
      this.currentColor.color = "green";
      this.currentColor.value = 0;
    },
    changeColorToOrange() {
      this.currentColor.color = "orange";
      this.currentColor.value = 1;
    },
    changeColorToRed() {
      this.currentColor.color = "red";
      this.currentColor.value = 2;
    },
    getIdFromString(idString) {
      let index = idString.indexOf(",");
      return [idString.slice(0, index), idString.slice(index + 1)];
    },
    downloadCSV() {
      var csv = 'Node1,Node2,Difficulty\n';
      for (let edge in this.savedEdges) {
        let edgeId = this.getIdFromString(edge);
        csv += [edgeId[0], edgeId[1], this.savedEdges[edge]].join(";");
        csv += "\n";
      }

      var downloadLink = document.createElement("a");
      var blob = new Blob(["\ufeff", csv]);
      var url = URL.createObjectURL(blob);
      downloadLink.href = url;
      downloadLink.download = this.locationSearch+".csv";
      document.body.appendChild(downloadLink);
      downloadLink.click();
      document.body.removeChild(downloadLink);
    }
  },
  mounted() {
    this.mymap = leaflet.map("osmmap").setView([50.7, 7.15], 18);
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

#searchbar {
  display: inline-block;
}

#refresh-button {
  float: right;
}

#download-button {
  float: right;
}

#navbar {
  margin-bottom: 10px;
}

#color-picker {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  column-gap: 20px;
  margin-top: 20px;
}

#color-picker p {
  display: inline-block;
}

#green-color {
  grid-column: 1/2;
}

#green-color-show {
  background-color: green;
  width: 30px;
  height: 30px;
  display: inline-block;
}

#orange-color {
  grid-column: 2/3;
}

#orange-color-show {
  background-color: orange;
  width: 30px;
  height: 30px;
  display: inline-block;
}

#red-color {
  grid-column: 3/4;
}

#red-color-show {
  background-color: red;
  width: 30px;
  height: 30px;
  display: inline-block;
}
</style>