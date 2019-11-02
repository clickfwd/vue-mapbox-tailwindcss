<template>
	<div class="text-sm flex flex-col sm:block">
		<h1 class="w-full text-center p-4 text-lg text-gray-700 font-medium">
			Vue Mapbox Tailwind CSS Demo
		</h1>
		<div class="flex flex-col flex-1 md:flex-row justify-center w-full">
			<div class="w-full md:w-1/2 p-4">
				<button
					@click="addMarkers"
					class="m-1 btn btn-blue w-full lg:w-auto"
					:disabled="buttonDisabled"
				>
					Add Markers
				</button>

				<button
					@click="removeMarkers"
					class="m-1 btn btn-blue w-full lg:w-auto"
					:disabled="buttonDisabled"
				>
					Remove Markers
				</button>

				<button
					@click="changeMapStyle"
					class="m-1 btn btn-blue w-full lg:w-auto"
					:disabled="buttonDisabled"
				>
					Change Map Style
				</button>

				<button
					@click="toggleMarkerColor"
					class="m-1 btn btn-blue w-full lg:w-auto"
					:disabled="buttonDisabled"
				>
					Change Marker Color
				</button>

				<button
					@click="useCustomMarker = !useCustomMarker"
					class="m-1 btn btn-blue w-full lg:w-auto"
				>
					Use Custom Marker
				</button>
			</div>
			<div
				class="px-4 w-full md:w-1/2 flex items-center mb-6 justify-end flex-col lg:flex-row lg:mb-0 lg:justify-between"
			>
				<div ref="mapSearch" class="w-full lg:w-1/2 order-last lg:order-first"></div>
				<div class="mb-4 w-full lg:w-auto lg:mt-0 flex justify-center lg:justify-end">
					There are {{ payload.length }} markers
				</div>
			</div>
		</div>
		<div class="map flex flex-col flex-1 md:flex-row justify-center w-full">
			<div
				class="flex flex-row w-full h-24 py-4 px-2 mb-4 bg-gray-100 flex-grow flex-shrink-0 overflow-x-auto h-6 md:h-auto md:overflow-y-auto md:flex-col md:w-1/2 md:mb-0 "
			>
				<div
					:class="{ 'bg-teal-200': selectedMarker == marker.id }"
					class="flex items-center p-2 md:mb-3 w-40 bg-white shadow cursor-pointer md:w-full"
					v-for="(marker, index) in payload"
					:key="marker.id"
					@click="itemClicked(index)"
					ref="items"
				>
					<span
						class="text-xs inline-flex items-center justify-center p-4 mr-2 w-6 h-6 rounded-full bg-teal-800 text-teal-100"
					>
						{{ marker.id }}
					</span>
					<span>{{ marker.label }}</span>
				</div>
			</div>
			<MglMap
				class="w-full md:w-1/2 flex-grow h-full min-h-full"
				:accessToken="accessToken"
				:mapStyle.sync="mapStyle"
				:scrollZoom="false"
				:center="mapCenter"
				:zoom="defaultZoom"
				@load="onMapLoaded"
			>
				<MglNavigationControl :showZoom="true" :showCompass="false" />
				<MglMarker
					v-for="marker in payload"
					:coordinates="[marker.lng, marker.lat]"
					:draggable="true"
					:color.sync="markerColor"
					:markerId="marker.id"
					:key="marker.id + '' + markerColor + (useCustomMarker ? 1 : 0)"
					@click="markerClicked"
					ref="markers"
				>
					<template slot="marker" v-if="useCustomMarker">
						<span
							class="text-xs inline-flex items-center justify-center p-3 mr-2 w-4 h-4 rounded-full"
							:class="{
								'bg-teal-800 text-teal-100': markerColor == 'blue',
								'bg-red-600 text-red-100': markerColor == 'red',
							}"
						>
							{{ marker.id }}
						</span>
					</template>
					<MglPopup>
						<div class="flex items-center">
							<span
								class="text-xs inline-flex items-center justify-center p-3 mr-2 w-4 h-4 rounded-full"
								:class="{
									'bg-teal-800 text-teal-100': markerColor == 'blue',
									'bg-red-800 text-red-100': markerColor == 'red',
								}"
							>
								{{ marker.id }}
							</span>
							<span class="text-sm">{{ marker.label }}</span>
						</div>
					</MglPopup>
				</MglMarker>
			</MglMap>
		</div>
	</div>
</template>

<script>
import Mapbox from 'mapbox-gl';
import { MglMap, MglNavigationControl, MglMarker, MglPopup } from 'vue-mapbox';
import MapboxGeocoder from '@mapbox/mapbox-gl-geocoder';

export default {
	components: {
		MglMap,
		MglNavigationControl,
		MglMarker,
		MglPopup,
	},
	data() {
		return {
			accessToken: process.env.VUE_APP_MAPBOX_ACCESS_TOKEN,
			defaultZoom: 12,
			mapCenter: [-71.038887, 42.364506],
			mapStyle: 'mapbox://styles/mapbox/light-v9',
			markerColor: 'blue',
			useCustomMarker: false,
			buttonDisabled: false,
			counter: 0,
			selectedMarker: null,
			payload: [],
			// Geocoder
			address: '',
		};
	},
	methods: {
		onMapLoaded(event) {
			this.map = event.map;
			this.addMarkers();
			this.addGeocoder();
		},
		toggleMarkerColor() {
			this.markerColor = this.markerColor == 'blue' ? 'red' : 'blue';
		},
		changeMapStyle() {
			if (this.mapStyle == 'mapbox://styles/mapbox/light-v9') {
				this.mapStyle = 'mapbox://styles/mapbox/streets-v11';
			} else {
				this.mapStyle = 'mapbox://styles/mapbox/light-v9';
			}
		},
		itemClicked(index) {
			this.$refs['markers'][index].togglePopup();
			this.selectedMarker = index;
			this.$refs.markers.forEach((marker, i) => {
				if (i != index) {
					marker.marker._popup.remove();
				}
			});

			this.$nextTick(() => {
				this.map.panTo(this.$refs['markers'][index].coordinates);
			});
		},
		addMarkers() {
			this.buttonDisabled = true;
			const bounds = this.map.getBounds();
			let lat_min = bounds.getSouthWest().lat,
				lat_range = bounds.getNorthEast().lat - lat_min,
				lng_min = bounds.getSouthWest().lng,
				lng_range = bounds.getNorthEast().lng - lng_min;
			let limit = this.counter + 20;
			for (this.counter; this.counter < limit; this.counter++) {
				this.payload.push({
					id: this.counter,
					lat: lat_min + Math.random() * lat_range,
					lng: lng_min + Math.random() * lng_range,
					label: Math.random()
						.toString(36)
						.substring(2, 15),
				});
			}
			this.buttonDisabled = false;
		},
		removeMarkers() {
			this.payload = [];
			this.counter = 0;
		},
		markerClicked(event) {
			let markerId = event.component.$attrs.markerId;
			this.selectedMarker = markerId;
			this.$refs.items[markerId].scrollIntoView({
				behavior: 'smooth',
				block: 'center',
				inline: 'center',
			});
		},
		addGeocoder() {
			this.geocoder = new MapboxGeocoder({
				accessToken: this.mapbox.accessToken,
				maboxgl: this.mapbox,
				marker: false,
			});
			this.$refs.mapSearch.appendChild(this.geocoder.onAdd(this.map));

			let $this = this;
			this.geocoder.on('result', function() {
				$this.$nextTick(() => {
					$this.addMarkers();
				});
			});
		},
	},
	created() {
		this.map = null;
		this.mapbox = Mapbox;
	},
};
</script>

<style>
.map {
	height: 500px;
}
@media screen and (min-width: 640px) {
	.mapboxgl-ctrl-geocoder {
		width: 100% !important;
		max-width: 100% !important;
	}
}
</style>
