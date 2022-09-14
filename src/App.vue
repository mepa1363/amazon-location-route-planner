<template>
  <div id="map"></div>
  <n-card>
    <n-space vertical>
      <!-- Travel mode options -->
      <n-radio-group v-model:value="travelModeOptions">
        <n-radio-button value="Car"> Car </n-radio-button>
        <n-radio-button value="Truck"> Truck </n-radio-button>
        <n-radio-button value="Walking"> Walking </n-radio-button>
      </n-radio-group>

      <n-space style="margin: 20px 0 20px 0">
        Click on the map to choose your starting point and destination(s).
      </n-space>

      <!-- Route summary -->
      <n-h3 v-if="route.features.length > 0">
        <n-text type="primary">
          {{ Math.round(route.features[0].properties.DurationSeconds / 60) }}
          Minutes
        </n-text>
        <n-text depth="3">
          ({{ route.features[0].properties.Distance.toFixed(1) }}
          {{ route.features[0].properties.DistanceUnit }})
        </n-text>
      </n-h3>

      <!-- Starting point and destinations -->
      <n-space>
        <n-timeline>
          <n-timeline-item
            v-for="position in positions.features"
            type="info"
            :content="position.geometry.coordinates.join(', ')"
            :title="position.properties.title"
            line-type="dashed"
          />
        </n-timeline>
      </n-space>
    </n-space>
    <n-collapse style="margin-top: 20px">
      <!-- Unit of measurement options -->
      <n-collapse-item title="Unit of measurement" name="1">
        <n-radio-group v-model:value="unitOfMeasurementOptions">
          <div>
            <n-radio value="metric"> Metric </n-radio>
          </div>
          <div>
            <n-radio value="imperial"> Imperial </n-radio>
          </div>
        </n-radio-group>
        <template #header-extra>
          <n-icon :component="RulerAlt" />
        </template>
      </n-collapse-item>

      <!-- Avoid options -->
      <n-collapse-item
        v-if="travelModeOptions === 'Car' || travelModeOptions === 'Truck'"
        title="Avoid options"
        name="2"
      >
        <n-checkbox-group v-model:value="avoidOptions">
          <div>
            <n-checkbox value="AvoidTolls" label="Tolls" />
          </div>
          <div>
            <n-checkbox value="AvoidFerries" label="Ferries" />
          </div>
        </n-checkbox-group>
        <template #header-extra>
          <n-icon :component="DirectionFork" />
        </template>
      </n-collapse-item>

      <!-- Truck options -->
      <n-collapse-item
        v-if="travelModeOptions === 'Truck'"
        title="Truck options"
        name="3"
      >
        <n-input
          clearable
          :placeholder="`Height (${units[unitOfMeasurementOptions]['truck']['dimension']})`"
          v-model:value="truckHeight"
        >
          <template #prefix>
            <n-icon :component="FitToHeight" />
          </template>
        </n-input>
        <n-input
          clearable
          :placeholder="`Width (${units[unitOfMeasurementOptions]['truck']['dimension']})`"
          v-model:value="truckWidth"
          style="margin-top: 10px"
        >
          <template #prefix>
            <n-icon :component="CenterToFit" />
          </template>
        </n-input>
        <n-input
          clearable
          :placeholder="`Length (${units[unitOfMeasurementOptions]['truck']['dimension']})`"
          v-model:value="truckLength"
          style="margin-top: 10px"
        >
          <template #prefix>
            <n-icon :component="FitToWidth" />
          </template>
        </n-input>
        <n-input
          clearable
          :placeholder="`Weight (${units[unitOfMeasurementOptions]['truck']['weight']})`"
          v-model:value="truckWeight"
          style="margin-top: 10px"
        >
          <template #prefix>
            <n-icon :component="Scales" />
          </template>
        </n-input>
        <template #header-extra>
          <n-icon :component="DeliveryTruck" />
        </template>
      </n-collapse-item>

      <!-- Departure time options -->
      <n-collapse-item
        v-if="travelModeOptions === 'Car' || travelModeOptions === 'Truck'"
        title="Departure time"
        name="4"
      >
        <n-radio-group v-model:value="departureTimeOptions">
          <div>
            <n-radio value="default"> Optimal traffic conditions </n-radio>
          </div>
          <div>
            <n-radio value="now"> Now </n-radio>
          </div>
          <div>
            <n-radio value="custom"> Leave at </n-radio>
          </div>
        </n-radio-group>
        <n-input-group
          v-if="departureTimeOptions === 'custom'"
          style="margin-top: 15px"
        >
          <n-date-picker v-model:value="timestamp" type="datetime" clearable />
        </n-input-group>
        <template #header-extra> <n-icon :component="Time" /> </template>
      </n-collapse-item>
    </n-collapse>
  </n-card>
</template>

<script setup>
import { ref, watch, computed, reactive } from "vue";
import {
  LocationClient,
  CalculateRouteCommand,
} from "@aws-sdk/client-location";
import maplibregl from "maplibre-gl";
import {
  NCard,
  NRadioGroup,
  NRadioButton,
  NRadio,
  NCheckboxGroup,
  NCheckbox,
  NInput,
  NInputGroup,
  NDatePicker,
  NIcon,
  NSpace,
  NCollapse,
  NCollapseItem,
  NTimeline,
  NTimelineItem,
  NH3,
  NText,
} from "naive-ui";
import {
  RulerAlt,
  DirectionFork,
  Time,
  DeliveryTruck,
  FitToHeight,
  FitToWidth,
  CenterToFit,
  Scales,
} from "@vicons/carbon";
import { point, lineString, featureCollection } from "@turf/helpers";
import {
  region,
  credentials,
  refreshCredentials,
  transformRequest,
} from "./auth";
import { mapName, routeCalculatorName } from "./config";

//  Variables ////////////////////////////////////////////////////////////////////////////////
let map = null;

let travelModeOptions = ref("Car");
const positions = reactive(featureCollection([]));
const route = reactive(featureCollection([]));

let unitOfMeasurementOptions = ref("metric");
const units = {
  metric: {
    route: { distance: "Kilometers" },
    truck: { dimension: "Meters", weight: "Kilograms" },
  },
  imperial: {
    route: { distance: "Miles" },
    truck: { dimension: "Feet", weight: "Pounds" },
  },
};

const avoidOptions = ref([]);
let departureTimeOptions = ref("default");
let timestamp = ref(Date.now());

let truckHeight = ref(null);
let truckWidth = ref(null);
let truckLength = ref(null);
let truckWeight = ref(null);

// Build API request body (computed properties) ///////////////////////////////////////////////////
const requestParams = computed(() => {
  const params = {
    CalculatorName: routeCalculatorName,
    TravelMode: travelModeOptions.value,
    DistanceUnit: units[unitOfMeasurementOptions.value]["route"]["distance"],
    IncludeLegGeometry: true,
  };

  // Positions (DeparturePosition, WaypointPositions, and DestinationPosition)
  if (positions.features.length > 1) {
    params.DeparturePosition = positions.features[0].geometry.coordinates;

    params.DestinationPosition =
      positions.features[positions.features.length - 1].geometry.coordinates;

    params.WaypointPositions = positions.features
      .slice(1, positions.features.length - 1)
      .map((feature) => feature.geometry.coordinates);
  }

  // DepartureTimeOptions
  if (departureTimeOptions.value === "default") {
    delete params.DepartNow;
    delete params.DepartureTime;
  }

  if (departureTimeOptions.value === "now") {
    delete params.DepartureTime;
    params.DepartNow = true;
  }

  if (departureTimeOptions.value === "custom") {
    delete params.DepartNow;
    params.DepartureTime = new Date(timestamp.value);
  }

  // CarModeOptions
  if (travelModeOptions.value === "Car") {
    params.CarModeOptions = {
      ...params.CarModeOptions,
      AvoidFerries:
        avoidOptions.value.findIndex((option) => option === "AvoidFerries") >
        -1,
    };
    params.CarModeOptions = {
      ...params.CarModeOptions,
      AvoidTolls:
        avoidOptions.value.findIndex((option) => option === "AvoidTolls") > -1,
    };
  }

  // TruckModeOptions
  if (travelModeOptions.value === "Truck") {
    params.TruckModeOptions = {
      ...params.TruckModeOptions,
      AvoidFerries:
        avoidOptions.value.findIndex((option) => option === "AvoidFerries") >
        -1,
    };
    params.TruckModeOptions = {
      ...params.TruckModeOptions,
      AvoidTolls:
        avoidOptions.value.findIndex((option) => option === "AvoidTolls") > -1,
    };

    if (
      truckHeight.value != null ||
      truckLength.value != null ||
      truckWidth.value != null
    ) {
      params.TruckModeOptions.Dimensions = {
        Height: truckHeight.value,
        Length: truckLength.value,
        Width: truckWidth.value,
        Unit: units[unitOfMeasurementOptions.value]["truck"]["dimension"],
      };
    }

    if (truckWeight.value != null) {
      params.TruckModeOptions.Weight = {
        Total: truckWeight.value,
        Unit: units[unitOfMeasurementOptions.value]["truck"]["weight"],
      };
    }
  }

  return params;
});

// Watchers, keeping track of changes in routing parameters ///////////////////////////////////////
watch(route, (value) => {
  map.getSource("route").setData(value);
});

watch(travelModeOptions, async () => {
  await calculateRoute();
});

watch(positions, async (value) => {
  map.getSource("positions").setData(value);
  map.getSource("positions-label").setData(value);
  await calculateRoute();
});

watch(unitOfMeasurementOptions, async () => {
  await calculateRoute();
});

watch(departureTimeOptions, async () => {
  await calculateRoute();
});

watch(timestamp, async () => {
  await calculateRoute();
});

watch(avoidOptions, async () => {
  await calculateRoute();
});

watch(truckHeight, async (value) => {
  if (value === "") {
    truckHeight.value = null;
  }
  await calculateRoute();
});

watch(truckWidth, async (value) => {
  if (value === "") {
    truckWidth.value = null;
  }
  await calculateRoute();
});

watch(truckLength, async (value) => {
  if (value === "") {
    truckLength.value = null;
  }
  await calculateRoute();
});

watch(truckWeight, async (value) => {
  if (value === "") {
    truckWeight.value = null;
  }
  await calculateRoute();
});

// Initialize app /////////////////////////////////////////////////////////////////////////////////
const initializeApp = async () => {
  // Refresh auth credentials
  await refreshCredentials();

  // Initialize a new map instance
  map = new maplibregl.Map({
    container: "map",
    center: [-114.067375, 51.046333],
    zoom: 16,
    style: mapName,
    hash: true,
    transformRequest,
  });

  // Add navigation controls for the map
  map.addControl(new maplibregl.NavigationControl(), "bottom-right");

  map.on("load", () => {
    // Add a layer for rendering departure, waypoint, and destination positions on the map
    map.addLayer({
      id: "positions",
      type: "circle",
      source: { type: "geojson", data: positions },
      paint: {
        "circle-radius": 5,
        "circle-color": "#ffffff",
        "circle-stroke-color": "#00b0ff",
        "circle-stroke-width": 3,
      },
    });

    // Add a layer for rendering position labels on the map
    map.addLayer({
      id: "positions-label",
      type: "symbol",
      source: { type: "geojson", data: positions },
      layout: {
        "text-field": ["get", "title"],
        "text-variable-anchor": ["left"],
        "text-radial-offset": 0.5,
        "text-justify": "auto",
        "text-font": ["Noto Sans Regular"],
      },
    });

    // Add a layer for rendering routes on the map
    map.addLayer(
      {
        id: "route",
        type: "line",
        source: { type: "geojson", data: route },
        layout: {
          "line-join": "round",
          "line-cap": "round",
        },
        paint: {
          "line-color": "#00b0ff",
          "line-width": 5,
          "line-opacity": 0.7,
        },
      },
      "positions"
    );
  });

  // Add an event handler to capture departure, waypoint, and destination positions
  let counter = 0;
  map.on("click", async (e) => {
    counter++;
    const { lngLat } = e;
    const p = point([lngLat.lng.toFixed(5), lngLat.lat.toFixed(5)], {
      title: `Point ${counter}`,
    });
    positions.features.push(p);
  });
};

// Calculate route
const calculateRoute = async () => {
  const client = new LocationClient({
    credentials: credentials,
    region: region,
  });

  if (
    requestParams.value.DeparturePosition &&
    requestParams.value.DestinationPosition
  ) {
    const command = new CalculateRouteCommand(requestParams.value);
    const response = await client.send(command);
    const routeFeature = lineString(
      response.Legs.flatMap((leg) => leg.Geometry.LineString),
      response.Summary
    );
    route.features.length = 0;
    route.features.push(routeFeature);
  }
};

initializeApp();
</script>

<style>
@import "maplibre-gl/dist/maplibre-gl.css";

body {
  margin: 0;
  padding: 0;
}
#map {
  position: absolute;
  top: 0;
  bottom: 0;
  width: 100%;
}
.n-card {
  max-width: 400px;
  margin: 10px 0 0 10px;
  box-shadow: 0 1px 2px -2px rgba(0, 0, 0, 0.08),
    0 3px 6px 0 rgba(0, 0, 0, 0.06), 0 5px 12px 4px rgba(0, 0, 0, 0.04);
}
</style>
