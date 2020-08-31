<template>
  <div id="map"></div>
</template>

<script>
import mapboxgl from "mapbox-gl";
import "mapbox-gl/dist/mapbox-gl.css";
import _ from "lodash";
import data from "./data";

export default {
  mounted() {
    mapboxgl.accessToken =
      "pk.eyJ1Ijoibmljb3ByYXQiLCJhIjoiY2lxOTd2YnBwMDA0Mmhza3FhdjNwMHJ0biJ9.DN7SuYgw4j0i0Pw4K-ZZqg";

    var map = new mapboxgl.Map({
      container: "map",
      zoom: 1,
      maxZoom: 15,
      renderWorldCopies: false,
      maxBounds: [[-200, -80], [200, 80]],
      style: {
        version: 8,
        name: "Empty",
        metadata: {
          "mapbox:autocomposite": true,
          "mapbox:type": "template"
        },
        glyphs: "mapbox://fonts/mapbox/{fontstack}/{range}.pbf",
        sources: {},
        layers: [
          {
            id: "background",
            type: "background",
            paint: {
              "background-color": "rgba(0,0,0,0)"
            }
          }
        ]
      }
    });

    map.on("load", function() {
      const features = _.sortBy(data, "occurrences")
        .reverse()
        .map(skill => ({
          type: "Feature",
          geometry: {
            type: "Point",
            coordinates: [skill.x * 340 - 170, skill.y * 140 - 70]
          },
          properties: {
            ...skill
          }
        }));

      map.addSource("points", {
        type: "geojson",
        data: {
          type: "FeatureCollection",
          features
        }
      });

      map.addLayer({
        id: "heatmap",
        type: "heatmap",
        source: "points",
        maxzoom: 9,
        paint: {
          // increase weight as diameter breast height increases
          "heatmap-weight": {
            property: "occurrences",
            type: "exponential",
            stops: [[0, 0], [100, 1]]
          },
          // increase intensity as zoom level increases
          "heatmap-intensity": {
            stops: [[11, 1], [15, 10]]
          },
          // use sequential color palette to use exponentially as the weight increases
          "heatmap-color": [
            "interpolate",
            ["linear"],
            ["heatmap-density"],
            0,
            "rgba(89, 152, 219, 0)",
            0.1,
            "rgba(89, 152, 219, .1)",
            0.3,
            "rgba(89, 152, 219, .3)",
            0.5,
            "rgba(89, 152, 219, .5)",
            0.7,
            "rgba(89, 152, 219, .7)",
            1,
            "rgba(89, 152, 219, 1)"
          ],
          // increase radius as zoom increases
          "heatmap-radius": {
            property: "occurrences",
            stops: [[{ zoom: 0, value: 0 }, 10], [{ zoom: 5, value: 100 }, 200]]
          },
          // decrease opacity to transition into the circle layer
          "heatmap-opacity": {
            default: 1,
            stops: [[3, 1], [5, 0.5]]
          }
        }
      });

      map.addLayer({
        id: "circles",
        source: "points",
        type: "circle",
        paint: {
          "circle-color": "#2A4365",
          "circle-radius": {
            property: "occurrences",
            stops: [
              [{ zoom: 0, value: 0 }, 1],
              [{ zoom: 0, value: 100 }, 1],
              [{ zoom: 5, value: 0 }, 10],
              [{ zoom: 5, value: 100 }, 20]
            ]
          },
          "circle-opacity": {
            default: 1,
            stops: [[3, 0.25], [4, 1]]
          }
        }
      });

      map.addLayer({
        id: "count",
        type: "symbol",
        source: "points",
        layout: {
          "text-field": ["get", "occurrences"],
          "text-font": ["Open Sans Semibold", "Arial Unicode MS Bold"],
          "text-size": {
            property: "occurrences",
            stops: [[0, 10], [100, 16]]
          },
          "text-allow-overlap": true
        },
        paint: {
          "text-color": "#fff",
          "text-opacity": {
            stops: [[3, 0], [4, 1]]
          }
        }
      });

      map.addLayer({
        id: "labels",
        type: "symbol",
        source: "points",
        layout: {
          "text-field": ["get", "name"],
          "text-font": ["Open Sans Semibold", "Arial Unicode MS Bold"],
          "text-offset": {
            stops: [[0, [0, 0]], [5, [0, 1]]]
          },
          "text-anchor": "top",
          "text-size": {
            property: "occurrences",
            stops: [[0, 10], [100, 22]]
          }
        },
        paint: {
          "text-color": "#2B6CB0",
          "text-halo-color": "#fff",
          "text-halo-width": 1,
          "text-halo-blur": 1
        }
      });

      const popup = new mapboxgl.Popup({
        closeButton: false,
        closeOnClick: false,
        anchor: "bottom",
        offset: {
          bottom: [0, 10]
        }
      });

      map.on("mouseenter", "labels", function(e) {
        // Change the cursor style as a UI indicator.
        map.getCanvas().style.cursor = "pointer";

        var coordinates = e.features[0].geometry.coordinates.slice();
        var description = `
      Compétence <strong>${
        e.features[0].properties.name
      }</strong> présente chez <strong>${
          e.features[0].properties.occurrences
        } collaborateurs</strong>. Cliquez pour plus de détails.
    `;

        // Ensure that if the map is zoomed out such that multiple
        // copies of the feature are visible, the popup appears
        // over the copy being pointed to.
        while (Math.abs(e.lngLat.lng - coordinates[0]) > 180) {
          coordinates[0] += e.lngLat.lng > coordinates[0] ? 360 : -360;
        }

        // Populate the popup and set its coordinates
        // based on the feature found.
        popup
          .setLngLat(coordinates)
          .setHTML(description)
          .addTo(map);
      });

      map.on("click", "labels", function(e) {
        // alert(`ID: ${e.features[0].properties.id}`)
        map.flyTo({
          center: e.features[0].geometry.coordinates,
          zoom: 5
        });
      });

      map.on("mouseleave", "labels", function() {
        map.getCanvas().style.cursor = "";
        popup.remove();
      });
      /*
  const coordinates = _.mapValues(map.getSource('points')._data.features(0,10), (d) => {
    return [d[0], d[1]]
  }).reduce((d) => {
    
  })
  */
    });
  }
};
</script>

<style>
html,
body,
#map {
  display: flex;
  flex-direction: column;
  min-height: 100%;
  flex-grow: 1;
  padding: 0;
  margin: 0;
}

canvas:focus {
  outline: none;
}
</style>