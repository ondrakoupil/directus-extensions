<template>
	<div class="superwrapper">

		<div class="map" :id="idForHtml">

		</div>

		<div class="geocoder" :class="{'geocoder--unsuccessful': geocoderUnsuccessful}">
			<form @submit.stop.prevent="geocode">
				<input class="geocoder-input" v-model="geocodingQuery" placeholder="Najít adresu nebo místo" />
				<button type="submit" v-if="!geocoderRunning" class="geocoder-button material-icons">search</button>
				<div class="progress-spinner" v-if="geocoderRunning" ></div>
			</form>
		</div>

		<div class="coordinates">
			<span class="coordinate-pair" :class="{'coordinate-pair--invalid': latInvalid}" >
				<input
						class="coordinate-input"
						@input="coordinatesChangedByUser('lat', $event)"
						@blur="coordinateFieldBlurred('lat', $event.target)"
						:value="coords ? coords.lat : ''"
				/><span class="input-affix">&deg; s. š.</span>
			</span>
			<span class="coordinate-pair"  :class="{'coordinate-pair--invalid': lngInvalid}" >
				<input
						class="coordinate-input"
						@input="coordinatesChangedByUser('lng', $event)"
						@blur="coordinateFieldBlurred('lng', $event.target)"
						:value="coords ? coords.lng : ''"
				/><span class="input-affix">&deg; v. d.</span>
			</span>
		</div>
	</div>
</template>

<script>
	import mixin from "@directus/extension-toolkit/mixins/interface";

	export default {
		mixins: [mixin],
		data() {
			return {
				geocoderRunning: false,
				geocodingQuery: '',
				geocoderUnsuccessful: false,
				markerInMap: null,
				markerLayerInMap: null,
				map: null,
				latInvalid: false,
				lngInvalid: false,
				defaultCoords: null,
			}
		},
		methods: {
			setCoordinates(coordinates) {
				this.value = JSON.stringify(coordinates);
				this.$emit('input', this.value);
				this.movePointerToCurrentValue();
			},
			movePointerToCurrentValue() {

				let coordinates;
				try {
					coordinates = JSON.parse(this.value);
				} catch (e) {
					coordinates = null;
				}
				if (coordinates) {
					let mapyCoordinates = SMap.Coords.fromWGS84(coordinates.lng, coordinates.lat);
					this.markerInMap.setCoords(mapyCoordinates);
					this.map.setCenter(mapyCoordinates, true);
					this.markerLayerInMap.enable();
				} else {
					this.markerLayerInMap.disable();
				}
			},
			coordinatesChangedByUser(type, event) {
				let number = parseFloat(event.target.value.replace(',', '.').trim());

				let coords = this.coords;
				let changed = false;
				if (type === 'lat') {
					if (number) {
						if (number !== coords.lat) {
							changed = true;
						}
						coords.lat = number;
						this.latInvalid = false;
					} else {
						this.latInvalid = true;
					}
				}
				if (type === 'lng') {
					if (number) {
						if (number !== coords.lng) {
							changed = true;
						}
						coords.lng = number;
						this.lngInvalid = false;
					} else {
						this.lngInvalid = true;
					}
				}

				if (changed) {
					this.setCoordinates(coords);
				}
			},
			coordinateFieldBlurred(type, field) {
				field.value = this.coords[type];
			},
			appendMapyApiIfNeeded() {
				if (window.mapyApiLoaded) {
					return Promise.resolve();
				}
				window.mapyApiLoaded = true;
				return new Promise(
					(resolve, reject) => {
						let s = document.createElement('script');
						s.src = "https://api.mapy.cz/loader.js";
						let finished = () => {
							console.log('Seznam Mapy.cz API loaded');
							resolve();
						};
						s.onload = () => {
							Loader.async = true;
							Loader.load(null, null, finished);
						};
						document.head.appendChild(s);
					}
				);
			},
			initializeMap() {
				let center;
				if (this.coords) {
					center = SMap.Coords.fromWGS84(this.coords.lng, this.coords.lat);
				} else {
					center = SMap.Coords.fromWGS84(+this.options.defaultLng || 15, +this.options.defaultLat || 50);
				}
				this.defaultCoords = center;
				let map = new SMap(document.getElementById(this.idForHtml), center, this.options.defaultZoom || 15);
				this.map = map;
				this.map.addDefaultLayer(SMap.DEF_BASE).enable();
				this.map.addDefaultControls();

				this.map.setCursor('crosshair');

				this.markerLayerInMap = new SMap.Layer.Marker();
				this.map.addLayer(this.markerLayerInMap);

				this.markerInMap = new SMap.Marker(center);
				this.markerLayerInMap.addMarker(this.markerInMap);

				if (this.coords) {
					this.movePointerToCurrentValue();
				}

				this.map.getSignals().addListener(window, 'map-click', (e) => {
					let coordsSeznam = SMap.Coords.fromEvent(e.data.event, this.map);
					let coordsString = coordsSeznam.toWGS84(0);
					let floatCoords = {
						lat: parseFloat(coordsString[1]),
						lng: parseFloat(coordsString[0]),
					};
					this.setCoordinates(floatCoords);
				});

			},

			geocode() {
				if (this.geocodingQuery && !this.geocoderRunning) {
					this.geocoderRunning = true;
					let geocoder = new SMap.Geocoder(this.geocodingQuery, () => {
						this.geocoderRunning = false;
						let coords = geocoder.getResults()[0].results.map((x) => x.coords);
						if (!coords.length) {
							this.geocoderUnsuccessful = true;
							setTimeout(() => {
								this.geocoderUnsuccessful = false;
							}, 500);
						} else {

							let currentCoords = this.coords ? SMap.Coords.fromWGS84(this.coords.lng, this.coords.lat) : this.defaultCoords;
							coords = coords.map((c) => {
								return {
									lat: c.y,
									lng: c.x,
									distance: c.distance(currentCoords)
								}
							});

							coords.sort((c1, c2) => {
								return c1.distance - c2.distance;
							});

							this.setCoordinates(coords[0]);

						}

					}, {
						url: 'api4.mapy.cz/geocode',
						count: 50,
					})
				}
			},

		},
		computed: {
			idForHtml() {
				return this.id + '-map-' + this._uid;
			},
			coords() {
				if (!this.value) {
					return null;
				}
				try {
					let parsed = JSON.parse(this.value);
					if (parsed) {
						return {
							lat: +parsed.lat || 0,
							lng: +parsed.lng || 0,
						}
					}
					return null;
				} catch (e) {
					return null;
				}
			}
		},
		updated() {
		},
		async mounted() {
			await this.appendMapyApiIfNeeded();
			this.initializeMap();
		}
	}
</script>

<style lang="scss" scoped>

	.superwrapper {
		position: relative;
	}

	.map {
		width: 100%;
		height: 600px;
		border: var(--input-border-width) solid var(--lighter-gray);
		border-radius: var(--border-radius);
		resize: both;
		overflow: hidden;
		position: relative;
		z-index: 5;
	}

	.coordinates {
		padding-top: 0.5rem;
	}

	.coordinate-pair {
		margin-right: 0.5rem;
	}

	.coordinate-input {
		width: 120px;
		border: var(--input-border-width) solid var(--lighter-gray);
		border-right: none;
		border-radius: var(--border-radius) 0 0 var(--border-radius);
		color: var(--gray);
		padding: 10px;
		font-size: 1rem;
		line-height: 1.5;
		text-transform: none;
		transition: var(--fast) var(--transition);
		transition-property: color, border-color, padding;
		height: var(--input-height);
		vertical-align: bottom;
	}

	.coordinate-pair--invalid {

		.coordinate-input {
			border-color: var(--red-600)
		}

		.input-affix {
			border-color: var(--red-600);
			background-color: var(--red-600);
			color: white;
		}

	}

	.input-affix {
		background-color: var(--grey-200);
		padding: 10px;
		display: inline-block;
		border: var(--input-border-width) solid var(--lighter-gray);
		border-left: none;
		border-radius: 0 var(--border-radius) var(--border-radius) 0;
		height: var(--input-height);
		vertical-align: bottom;
	}

	.geocoder {
		position: absolute;
		left: 10px;
		top: 10px;
		z-index: 10;
	}

	.geocoder--unsuccessful {
		animation: shakeGeocoderField 0.5s;

		.geocoder-input {
			border-color: var(--red-500)
		}
	}

	.geocoder-input {
		width: 300px;
		border: var(--input-border-width) solid var(--lighter-gray);
		border-radius: var(--border-radius) 0 0 var(--border-radius);
		color: var(--gray);
		padding: 10px;
		font-size: 1rem;
		transition: var(--fast) var(--transition);
		transition-property: color, border-color, padding;
		height: var(--input-height);
		vertical-align: bottom;

		&::placeholder {
			opacity: 0.5;
		}
	}

	.geocoder-button {
		font-size: 24px;
		display: inline-block;
		position: absolute;
		right: 0;
		top: 0;
		padding: 10px;
		transition: all 0.3s;

		&:hover {
			color: black;
		}
	}

	.progress-spinner {
		position: absolute;
		right: 10px;
		top: 10px;
		width: 24px;
		height: 24px;
		transition: all 0.3s;
		background: no-repeat center center url('data:image/svg+xml;base64,PD94bWwgdmVyc2lvbj0iMS4wIiBlbmNvZGluZz0idXRmLTgiPz4KPHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHhtbG5zOnhsaW5rPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5L3hsaW5rIiBzdHlsZT0ibWFyZ2luOiBhdXRvOyBiYWNrZ3JvdW5kOiB3aGl0ZTsgZGlzcGxheTogYmxvY2s7IHNoYXBlLXJlbmRlcmluZzogYXV0bzsiIHdpZHRoPSIyMDBweCIgaGVpZ2h0PSIyMDBweCIgdmlld0JveD0iMCAwIDEwMCAxMDAiIHByZXNlcnZlQXNwZWN0UmF0aW89InhNaWRZTWlkIj4KPGNpcmNsZSBjeD0iNTAiIGN5PSI1MCIgZmlsbD0ibm9uZSIgc3Ryb2tlPSIjODVhMmI2IiBzdHJva2Utd2lkdGg9IjEwIiByPSIzNSIgc3Ryb2tlLWRhc2hhcnJheT0iMTY0LjkzMzYxNDMxMzQ2NDE1IDU2Ljk3Nzg3MTQzNzgyMTM4IiB0cmFuc2Zvcm09InJvdGF0ZSgxNzcuOTg5IDUwIDUwKSI+CiAgPGFuaW1hdGVUcmFuc2Zvcm0gYXR0cmlidXRlTmFtZT0idHJhbnNmb3JtIiB0eXBlPSJyb3RhdGUiIHJlcGVhdENvdW50PSJpbmRlZmluaXRlIiBkdXI9IjAuNjQxMDI1NjQxMDI1NjQxcyIgdmFsdWVzPSIwIDUwIDUwOzM2MCA1MCA1MCIga2V5VGltZXM9IjA7MSI+PC9hbmltYXRlVHJhbnNmb3JtPgo8L2NpcmNsZT4KPCEtLSBbbGRpb10gZ2VuZXJhdGVkIGJ5IGh0dHBzOi8vbG9hZGluZy5pby8gLS0+PC9zdmc+');
		background-size: 24px 24px;
	}

	@keyframes shakeGeocoderField {
		0% {
			transform: translateX(0);
		}
		12% {
			transform: translateX(-10px);
		}
		37% {
			transform: translateX(8px);
		}
		62% {
			transform: translateX(-5px);
		}
		88% {
			transform: translateX(2px);
		}
		100% {
			transform: translateX(0);
		}
	}

</style>
