<template>
  <div id="map"></div>
</template>

<script>
import mapboxgl from "mapbox-gl";
import "mapbox-gl/dist/mapbox-gl.css";
import _ from "lodash";
import { categories } from "./data";

export default {
  mounted() {
    mapboxgl.accessToken =
      "pk.eyJ1Ijoibmljb3ByYXQiLCJhIjoiY2lxOTd2YnBwMDA0Mmhza3FhdjNwMHJ0biJ9.DN7SuYgw4j0i0Pw4K-ZZqg";

    const map = new mapboxgl.Map({
      container: "map",
      zoom: 1,
      maxZoom: 15,
      renderWorldCopies: false,
      maxBounds: [[-150, -60], [150, 60]],
      style: {
        version: 8,
        name: "Empty",
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

    const { clientWidth, clientHeight } = map.getContainer();
    const ZOOM_PER_LEVEL = 2;
    let maxLevel = 1; // Current max level within parent category

    window.categories = categories;

    const setup = ({ category, level, parentColor, parentCategoryName }) => {
      const color = parentColor || category.color;
      const bounds = new mapboxgl.LngLatBounds();
      maxLevel = Math.max(maxLevel, level);

      // First setup children and extend bounds from them
      (category.categories || []).forEach(child => {
        const { childrenBounds } = setup({
          category: child,
          level: level + 1,
          parentColor: color,
          parentCategoryName: [parentCategoryName, category.name]
            .filter(Boolean)
            .join("  /  ")
        });
        bounds.extend(childrenBounds);
      });

      const skills = category.skills.map(skill => ({
        ...skill,
        occurrences: Math.ceil(Math.random() * 100)
      }));
      const features = _.sortBy(skills, "occurrences")
        .reverse()
        .map(skill => ({
          type: "Feature",
          geometry: {
            type: "Point",
            coordinates: map
              .unproject([
                clientWidth * 0.1 + skill.x * (clientWidth * 0.8),
                clientHeight * 0.1 + skill.y * (clientHeight * 0.8)
              ])
              .toArray()
          },
          properties: {
            ...skill
          }
        }));

      map.addSource(`points-${category.name}`, {
        type: "geojson",
        data: {
          type: "FeatureCollection",
          features
        }
      });

      // Heatmap
      map.addLayer(
        {
          id: `heatmap-${category.name}`,
          source: `points-${category.name}`,
          type: "heatmap",
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
              color.replace("1.0", "0.0"),
              0.5,
              color.replace("1.0", "0.3"),
              1,
              color.replace("1.0", "0.8")
            ],
            // increase radius as zoom increases
            "heatmap-radius": {
              property: "occurrences",
              stops: [
                [{ zoom: 0, value: 0 }, 10],
                [{ zoom: 5, value: 100 }, 200]
              ]
            },
            // decrease opacity to transition into the circle layer
            "heatmap-opacity": {
              default: 1,
              stops: [[3, 1], [5, 0.5]]
            }
          }
        },
        "background"
      );

      // Circles
      map.addLayer({
        id: `circles-${category.name}`,
        source: `points-${category.name}`,
        type: "circle",
        paint: {
          "circle-color": color,
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

      // Counts
      map.addLayer({
        id: `count-${category.name}`,
        source: `points-${category.name}`,
        type: "symbol",
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

      // Labels
      map.addLayer({
        id: `labels-${category.name}`,
        source: `points-${category.name}`,
        type: "symbol",
        layout: {
          "text-field": ["get", "name"],
          "text-font": ["Open Sans Semibold", "Arial Unicode MS Bold"],
          "text-anchor": "top",
          "text-size": {
            property: "occurrences",
            stops: [[0, 10], [100, 22]]
          }
        },
        paint: {
          "text-color": color,
          "text-halo-color": "#fff",
          "text-halo-width": 1.5,
          "text-halo-blur": 0,
          "text-translate": [0, 15]
        }
      });

      // Popup
      const popup = new mapboxgl.Popup({
        closeButton: false,
        closeOnClick: false,
        anchor: "bottom",
        offset: {
          bottom: [0, 10]
        }
      });
      map.on("mouseenter", `labels-${category.name}`, ({ features }) => {
        map.getCanvas().style.cursor = "pointer";
        popup
          .setLngLat(features[0].geometry.coordinates)
          .setHTML(
            `Compétence <strong>${
              features[0].properties.name
            }</strong> présente chez <strong>${
              features[0].properties.occurrences
            } collaborateurs</strong>. Cliquez pour plus de détails.`
          )
          .addTo(map);
      });
      map.on("click", `labels-${category.name}`, ({ features }) => {
        map.flyTo({
          center: features[0].geometry.coordinates,
          zoom: 5
        });
      });
      map.on("mouseleave", `labels-${category.name}`, () => {
        map.getCanvas().style.cursor = "";
        popup.remove();
      });

      // Categories names & bounds
      features.forEach(feature => bounds.extend(feature.geometry.coordinates));
      map.addSource(`category-${category.name}`, {
        type: "geojson",
        data: {
          type: "FeatureCollection",
          features: [
            {
              type: "Feature",
              properties: {
                name: category.name
              },
              geometry: {
                type: "Point",
                coordinates: bounds.getCenter().toArray()
              }
            }
          ]
        }
      });

      // With ZOOM_PER_LEVEL = 2, results = [zoom, opacity]
      // Level 1:
      // [1, 1] --> first level, so it's visible at first
      // [2, 1] --> visible at target zoom level (1*2)
      // [3, 1] --> start fade out when next level fade in starts
      // [4, 0] --> end fade out when next level fade in ends
      // Level 2:
      // [3, 0] --> invisible at first, start fade in
      // [4, 1] --> end fade in at target zoom level (2*2)
      // [7, 1] --> start fade out when next level fade in starts
      // [8, 0] --> end fade out when next level fade in ends, or leave visible if max level

      const opacityStops = [
        [ZOOM_PER_LEVEL * level - 1, level === 1 ? 1 : 0],
        [ZOOM_PER_LEVEL * level, 1],
        [ZOOM_PER_LEVEL * (level + 1) - 1, 1],
        [ZOOM_PER_LEVEL * (level + 1), level === maxLevel ? 1 : 0]
      ];

      const MIN_FONT_SIZE = 10;
      const MAX_FONT_SIZE = 45;

      const fontSizeStops = [
        [ZOOM_PER_LEVEL * level - 1, MIN_FONT_SIZE],
        [ZOOM_PER_LEVEL * (level + 1), MAX_FONT_SIZE]
      ];

      // Add subtitle above label if child category
      const textField = parentCategoryName
        ? [
            "format",
            parentCategoryName,
            {
              "font-scale": 0.5,
              "text-translate": [0, 20]
            },
            "\n",
            {},
            ["get", "name"],
            {}
          ]
        : ["get", "name"];

      map.addLayer({
        id: `category-${category.name}`,
        source: `category-${category.name}`,
        type: "symbol",
        layout: {
          "text-transform": "uppercase",
          "text-field": textField,
          "text-font": ["Open Sans Bold"],
          "text-anchor": "top",
          "text-size": {
            stops: fontSizeStops
          },
          "text-line-height": 1,
          "text-ignore-placement": true, // Allow child category labels to bo visible even if colliding
          "text-letter-spacing": -0.03 // Prevent spaces from creating halo gaps
        },
        paint: {
          "text-color": color,
          "text-halo-color": "#fff",
          "text-halo-width": 10,
          "text-opacity": {
            stops: opacityStops
          }
        }
      });

      map.on("mouseenter", `category-${category.name}`, () => {
        map.getCanvas().style.cursor = "pointer";
      });

      map.on("click", `category-${category.name}`, () => {
        map.fitBounds(bounds, { padding: 50 });
      });

      map.on("mouseleave", `category-${category.name}`, () => {
        map.getCanvas().style.cursor = "";
      });

      return {
        childrenBounds: bounds
      };
    };

    map.on("load", () =>
      categories.forEach(category => setup({ category, level: 1 }))
    );
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