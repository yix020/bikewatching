
<script>
    import mapboxgl from 'mapbox-gl'; // Import Mapbox JavaScript library
    import '../../node_modules/mapbox-gl/dist/mapbox-gl.css'; // Import Mapbox CSS
    import * as d3 from 'd3'; // Import D3 for data handling
  
    import { onMount } from 'svelte';
  
    // Set your Mapbox access token
    mapboxgl.accessToken = 'pk.eyJ1IjoieWl4MDIwIiwiYSI6ImNtM3FlNGsxcjAyMmYydnBub2RjajdlazYifQ.BQvyeoAZOyZmmghXx-LdMA'; // Replace with your actual Mapbox token
    function minutesSinceMidnight(date) {
        return date.getHours() * 60 + date.getMinutes();
    }
    
    let map;
    let stations = []; // Variable to hold the fetched station data
    let mapViewChanged = 0; 
    let trips = [];
    let departures, arrivals;
    let maxTraffic = 0;
    let timeFilter = -1;
    let stationFlow = d3.scaleQuantize().domain([0, 1]).range([0, 0.5, 1]);

      // Reactive variable for displaying the time label
    $: timeFilterLabel = timeFilter === -1
        ? "(any time)"
        : new Date(0, 0, 0, Math.floor(timeFilter / 60), timeFilter % 60).toLocaleString('en', {
            timeStyle: 'short',
        });

    function formatTime(minutes) {
        const hours = Math.floor(minutes / 60);
        const mins = minutes % 60;
        return `${hours.toString().padStart(2, '0')}:${mins
        .toString()
        .padStart(2, '0')}`;
    }
    onMount(async () => {
      const TRIP_DATA_URL = 'https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv';

      // Initialize the Mapbox map

      map = new mapboxgl.Map({
        container: 'map', // ID of the container
        style: 'mapbox://styles/mapbox/streets-v11', // Map style
        center: [-71.1097, 42.3736], // Center on Boston and Cambridge
        zoom: 13 // Adjusted zoom level to include both areas
      });
  
      await new Promise((resolve) => map.on('load', resolve));
  
      // Add Boston bike lanes
      map.addSource('boston_route', {
        type: 'geojson',
        data: 'https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D',
      });
  
      map.addLayer({
        id: 'boston_bike_routes',
        type: 'line',
        source: 'boston_route',
        layout: {
          'line-join': 'round',
          'line-cap': 'round',
        },
        paint: {
          'line-color': '#FF5733', // Boston route color
          'line-width': 4,
          'line-opacity': 0.6,
        },
      });
  
      // Add Cambridge bike lanes
      map.addSource('cambridge_route', {
        type: 'geojson',
        data: 'https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson', // Cambridge bike lane GeoJSON file
      });
  
      map.addLayer({
        id: 'cambridge_bike_routes',
        type: 'line',
        source: 'cambridge_route',
        layout: {
          'line-join': 'round',
          'line-cap': 'round',
        },
        paint: {
          'line-color': '#FFFF00', // Yellow for Cambridge bike lanes
          'line-width': 4,
          'line-opacity': 0.6,
        },
      });
  
      // Fetch Bluebikes station data
      stations = await d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-stations.csv');
      console.log('Fetched station data:', stations); // Log station data to console for exploration
      // Fetch traffic data
      trips = await d3.csv(TRIP_DATA_URL).then((trips) => {
        for (let trip of trips) {
            trip.started_at = new Date(trip.started_at); // Convert started_at to Date object
            trip.ended_at = new Date(trip.ended_at); // Convert ended_at to Date object
        }
        return trips; // Return processed trips
       });      
        console.log('Fetched traffic data:', trips);

          // Calculate departures and arrivals
        departures = d3.rollup(
            trips,
            (v) => v.length,
            (d) => d.start_station_id
        );

        arrivals = d3.rollup(
            trips,
            (v) => v.length,
            (d) => d.end_station_id
        );

            // Add arrivals, departures, and totalTraffic to each station
        stations = stations.map((station) => {
            const id = station.Number; // Assuming 'Number' is the station ID field
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            maxTraffic = Math.max(maxTraffic, station.totalTraffic);
            return station;
        });


        console.log(stations);
        // Append circles to the SVG for station markers
        svg = d3.select('.map-overlay'); // Select the existing SVG

        const renderMarkers = () => {
        svg.selectAll('circle').remove(); // Clear previous markers

        svg.selectAll('circle')
            .data(stations)
            .enter()
            .append('circle')
            .attr('cx', (d) => map.project([+d.Long, +d.Lat]).x) // Project longitude to x
            .attr('cy', (d) => map.project([+d.Long, +d.Lat]).y) // Project latitude to y
            .attr('r', 5) // Circle radius
            .attr('fill', 'blue') // Circle color
            .attr('opacity', 0.8); // Semi-transparent for better visibility
        };

        // Initial render
        renderMarkers();

        // Reposition markers on map move or zoom
   

        // Increment mapViewChanged on map move
        
    });
    // Helper function to calculate circle center coordinates
    function getCoords(station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let { x, y } = map.project(point);
        return { cx: x, cy: y };
    }

    $: map?.on('move', (evt) => mapViewChanged++);
    // $: maxTraffic = Math.max(stations.map(station => station.totalTraffic));
    $: radiusScale = d3
            .scaleSqrt()
            .domain([0, maxTraffic])
            .range(timeFilter === -1 ? [0, 25] : [3, 50]);
    $: filteredTrips =
      timeFilter === -1
        ? trips // No filtering
        : trips.filter((trip) => {
            let startedMinutes = minutesSinceMidnight(trip.started_at);
            let endedMinutes = minutesSinceMidnight(trip.ended_at);
            return (
            Math.abs(startedMinutes - timeFilter) <= 60 ||
            Math.abs(endedMinutes - timeFilter) <= 60
            );
        });

    $: filteredDepartures = d3.rollup(
        filteredTrips,
        (v) => v.length,
        (d) => d.start_station_id
    );

    $: filteredArrivals = d3.rollup(
        filteredTrips,
        (v) => v.length,
        (d) => d.end_station_id
    );

    $: fileredStations = stations.map((station) => {
            station = { ...station };
            const id = station.Number; // Assuming 'Number' is the station ID field
            station.arrivals = filteredArrivals.get(id) ?? 0;
            station.departures = filteredDepartures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            // maxTraffic = Math.max(maxTraffic, station.totalTraffic);
            return station;
        })
  </script>
  
  <style>
    @import url('$lib/global.css');
    header {
        display: flex;
        gap: 1em;
        align-items: baseline;
    }

    label {
        margin-left: auto; /* Push slider to the right */
    }

    label input[type='range'] {
        margin-left: 1em; /* Space between label text and slider */
    }

    time {
        display: block; /* Place time on its own line */
        margin-top: 0.5em;
    }

    time em {
        color: #888; /* Lighter color for (any time) */
        font-style: italic; /* Italicize (any time) */
    }
    #map-container {
      display: flex;
      justify-content: center; /* Centers the map horizontally */
      margin: 2rem 0; /* Adds vertical spacing */
    }
  
    #map {
      flex: 0 0 80%; /* Makes the map take up 80% of the width */
      height: 600px; /* Increases the height of the map */
      background-color: lightgray; /* Temporary background to ensure visibility */
      border: 2px solid #ddd; /* Optional: Add a border for visual clarity */
      border-radius: 8px; /* Optional: Add rounded corners */
    }
  
    #map svg {
        position: absolute; /* Ensure the SVG overlays the map */
        z-index: 1; /* Place the SVG above the map */
        width: 100%; /* Make it fill the map container */
        height: 100%; /* Match the map's height */
        pointer-events: none; /* Allow interactions with the map */
        /*background: yellow; /* Set a yellow background for visibility */
        /*opacity: 0.5; /* Semi-transparent for debugging */
     }
  
    h1 {
      text-align: center;
      color: #007bff;
    }
  
    h2 {
      text-align: center;
      color: #555;
    }
  
    p {
      margin: 0 auto;
      max-width: 600px;
      line-height: 1.6;
    }

    circle, .legend {
        fill: steelblue;
        fill-opacity: 60%;
        stroke: white;
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
        fill: var(--color);
    }

    .legend {
        display: flex;
        margin-block: 3, 3, 3, 3;
    }

    .legend-div {
        flex: 1;
        margin: 1px;
        color: white;
        padding: 10px;
    }
  </style>
  
  <!-- Title -->
  <header>
    <h1>🚴 BIKEWATCHING</h1>
    <label>
      Filter by time:
      <input type="range" min="-1" max="1440" bind:value={timeFilter} />
      <time>{timeFilterLabel}</time>
    </label>
  </header>
  
  <!-- Map Container -->
  <div id="map-container">
    <div id="map">
      <svg class="map-overlay">
        {#key mapViewChanged}
          {#each fileredStations as station}
            <circle
              cx={getCoords(station).cx}
              cy={getCoords(station).cy}
              r={radiusScale(station.totalTraffic)}
              fill="steelblue"
              style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }"
            />
          {/each}
        {/key}
      </svg>
    </div>
  </div>

  <div class="legend">
    <div class="legend-div" style="--departure-ratio: 1; background-color: steelblue; text-align: left">More departures</div>
    <div class="legend-div" style="--departure-ratio: 0.5; background-color: rgb(203, 195, 227); text-align: center">Balanced</div>
    <div class="legend-div" style="--departure-ratio: 0; background-color: darkorange; text-align: right">More arrivals</div>
  </div>
  
  <!-- Subtitle -->
  <h2>Interactive Boston and Cambridge Bike Traffic Map Visualization</h2>
  
  <!-- Description -->
  <p>
      This map visualizes bike traffic in both Boston and Cambridge. The map includes:
  </p>    
  <p>
      <strong>Boston Bike Lanes:</strong> Represented by vivid orange-red lines.<br />
      <strong>Cambridge Bike Lanes:</strong> Represented by yellow lines.<br />
      Users can pan and zoom to explore the bike network in greater detail.
  </p>
  