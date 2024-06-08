---
title: Exploring Mapbox - Evolving Notes
description:  Exploring Mapbox and Mapbox GL
permalink: posts/{{ title | slug }}/index.html
date: '2024-06-06'
tags: [Mapbox, Javascript, Web]
---

I have been exploring Mapbox over the last year in an attempt to improve my tooling when adventuring into the backcountry. These notes are ever evolving and may contain errors - both spelling and logical.

## Key Components
- Mapbox GL: A graphics library that renders both 2D and 3D maps with OpenGL.
- Client-side Rendering: Mapbox GL JS is the client-side Javascript library for building applications with Mapbox. Maps are dynamically rendered with vector tiles and styling on the client side.
- The Map Class: The basis of every Mapbox GL JS project.

```javascript
mapboxgl.accessToken = <TOKEN>;
const map = new mapboxgl.map({
    container: 'map', // container id
    style: '', // add a style URL
    center: [], // focal point in lng, lat
    zoom: 9 // starting zoom of the map
});
```

Components:
- Container: The HTML element where the map is to be held. In the sample code above, the maps being stored in a div with an ID of map.
- Style: The style URL allows for referencing a style created with Mapbox Studio. The URL is made up of three main components: the Mapbox Styles API, our Mapbox username, and the styles unique ID.
- Center: The coordinates of the maps starting focal point in longitude and latitude. NOTE: Mapbox uses decimal degress however the formatting is not always consistent. For example, Mapbox GL JS use long, lat while Mapbox Maps SDK for iOS uses lat, long.
- Zoom: Sets the zoom level we want the map to be set at when initialized. Can be a whole number or decimals with the following:
  - 0: The Earth
  - 3: A continent
  - 4: Large islands
  - 6: Large rivers
  - 7: Large roads
  - 15: Buildings

Map tiles are stored in a quadtree data structure.

### Understanding quadtrees
Quadtrees are a tree data structure where each internal node has exactly four children and most commonly used to partition a two-dimensional space by recursively subdividing it into four quadrants or regions. The leaf cell can change depending on the use case but traditionally each leaf cell represents a unit of spacial information.

The subregions are not required to be square and can be of any arbitrary shape. 

All quadtrees share the following features:
- They decompose space into adaptable cells.
- Each cell (or bucket) has a maximum capacity. When that capacity is reached, the bucket splits.
- The tree directory follows the spacial decomposition of the quadtree.
- The tree-pyramid (T-pyramid) is a complete tree as every node has four child nodes except leaf nodes, all leaves are on the same level, and that level corresponds to individual pixels in the image.

### Tiles
Mapbox tiles based on Mapbox GL display in 512x512 pixels by default. The only product from Mapbox that uses a different default is Mapbox Raster Tiles API which defaults to 256x256. However, the defaults can be changed by defining the pixels in the maps source definition and ensuring the TileJSON is correct.

```json
"mapbox-satellite": {
    "type": "raster",
    "url": "mapbox://mapbox.satellite",
    "tileSize": 256
}
```

## Layers
As with most tooling, Mapbox allows for several layers to be used which provide additional visuals and information on a map.

Mapbox relies on the GL JS addLayer methods which relies on, at minimum, a Mapbox style layer object. One important component is the optional before parameter.

The before parameter is the ID of an existing layer that is used to insert the new layer before. If the parameter is left out, then the renderer will draw the layer on top of the map. This can be improved upon by using Mapbox Standard.

### Mapbox standard
Mapbox standard uses the concept of slots to make adding layers easier. Slots are simply predetermined locations where the new layer will be added - such as as the top layer but below any labels. The ordering of these layers at runtime can be defined as follows:
- bottom: Above any polygons such as land, landuse, water, etc.
- middle: Above lines such as roads but behind 3d buildings. (think 3D riving directions on Apple or Google Maps)
- top: Above POI labels and behind Place and Transit labels.

If no label is specified, then the layer will sit above all existing layers in the style.

```js
map.addLayer({ type: 'line', slot: 'middle' /* ... */ });
```

Use slot instead of id when inserting a custom layer into the Standard basemap. If a layer has the same slot, they can be rearranged using the beforeId argument. If the beforeId argument is used but the layers are of different slots, then it's ignored.

Note:
" During 3D globe and terrain rendering, GL JS aims to batch multiple layers together for optimal performance. This process might lead to a rearrangement of layers. Layers draped over globe and terrain, such as fill, line, background, hillshade, and raster, are rendered first. These layers are rendered underneath symbols, regardless of whether they are placed in the middle or top slots or without a designated slot."

## Event bindings and layers
Mapbox GL JS layers are remote so code that connects to it often uses event binding to change the map at the right time.

```js
map.on('load', () => {
  map.addLayer({
    id: 'terrain-data',
    type: 'line',
    source: {
      type: 'vector',
      url: 'mapbox://mapbox.mapbox-terrain-v2'
    },
    'source-layer': 'contour'
  });
});
```

The above example uses function to call map.addLayer only after the maps resources have been loaded. If we attempted to load the layer first, it would throw an error as the layer wouldn't exist yet.

When adding a new layer, the following source types are available:
- Vector Tiles: Must be in the Mapbox vector tile format and must specify a source-layer value.
- Raster Tiles:
- Raster-DEM: Only supports Mapbox Terrain-DEM
- GeoJSON: Data must be provided via a "data" property which can be either a URL or an inline GeoJSON.
- Image: A URL that contains the image location and a coordinates array.
- Video: A video source where the "urls" value is an array to support multiple browsers. Contains an array of coordinates listed in clockwise order from the top for video corners.

### Data styling
Layers feature two properties for data styling that define how data will be rendered on the map:
- paint: Fine-grained attributes such as opacity, color, and translation.
- layout: Applied early in the rendering process and refers to placement and visibility.

Sample code:
```js
map.on('load', () => {
  map.addLayer({
    id: 'rpd_parks',
    type: 'fill',
    source: {
      type: 'vector',
      url: 'mapbox://mapbox.3o7ubwm8'
    },
    'source-layer': 'RPD_Parks',
    layout: {
      visibility: 'visible'
    },
    paint: {
      'fill-color': 'rgba(61,153,80,0.55)'
    }
  });
});
```

## Camera
You can adjust the maps field of view with the following parameters:
- center
- zoom
- bearing
- pitch