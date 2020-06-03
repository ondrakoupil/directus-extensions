<template>
	<div v-if="coords" class="wrapper" :class="{'wrapper--hidden': hiddenMap}">
		<button class="coord-button" v-on:click.prevent.stop="toggleMap">
			<span class="the-icon">{{ (loadedMap && !hiddenMap) ? 'close' : 'location_on' }}</span>
			<span class="the-label">
				N {{ coords.lat | coord }}, E {{ coords.lng | coord }}
			</span>
		</button>
		<div class="map-window" v-on:click.prevent.stop="noop">
			<div class="map-window-js" :id="idForHtml">
				<span class="map-placeholder">Načítám mapu...</span>
			</div>
		</div>
	</div>
</template>

<script>
	import mixin from "@directus/extension-toolkit/mixins/interface";

	let initializedComponents = [];

	export default {
		mixins: [mixin],
		data: function() {
			return {
				hiddenMap: false,
				loadedMap: false,
			}
		},
		methods: {
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
			noop() {

			},
			removeOverflowHiddenFromCell() {
				let wrapper = document.querySelector('#' + this.idForHtml);
				if (wrapper) {
					let cell = wrapper.closest('.cell');
					cell.setAttribute('data-orig-overflow', cell.style.overflow);
					cell.style.overflow = 'visible';
					let body = wrapper.closest('.body');
					body.setAttribute('data-orig-overflow', body.style.overflow);
					body.style.overflow = 'visible';
					let vTable = wrapper.closest('.v-table');
					vTable.setAttribute('data-orig-overflow', vTable.style.overflow);
					vTable.style.overflow = 'visible';
				}
			},
			async toggleMap() {

				this.hideAllOtherMaps();

				if (this.loadedMap) {
					this.hiddenMap = !this.hiddenMap;
				} else {


					this.removeOverflowHiddenFromCell();
					await this.appendMapyApiIfNeeded();
					let center = SMap.Coords.fromWGS84(this.coords.lng, this.coords.lat);
					let map = new SMap(document.getElementById(this.idForHtml), center, 17);
					map.addDefaultLayer(SMap.DEF_BASE).enable();
					map.addControl(new SMap.Control.Zoom());

					let layer = new SMap.Layer.Marker();
					map.addLayer(layer);
					layer.enable();

					let marker = new SMap.Marker(center, null, {});
					layer.addMarker(marker);

					this.loadedMap = true;

					initializedComponents.push(this);

				}
			},
			closeMap() {

				if (this.loadedMap) {
					this.hiddenMap = true;
				}

			},
			hideAllOtherMaps() {
				initializedComponents.map(
					(vueComponent) => {
						if (vueComponent !== this) {
							vueComponent.closeMap();
						}
					}
				)
			}
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
		filters: {
			coord(value) {
				return Math.round(value * 1000) / 1000;
			}
		}
	}
</script>

<style lang="scss" scoped>
	.coord-button {
		cursor: pointer;
		transition: color 0.3s;

		&:hover {
			color: black;
		}
	}
	.map-window {
		position: absolute;
		z-index: 100;
		right: 0;
		width: 300px;
		height: 300px;
		top: 35px;
		padding: 3px;
		border-radius: 4px;
		border: solid 1px #CCC;
		box-shadow: rgba(0, 0, 0, 0.1) 0 1px 2px;
		transition: opacity 0.3s, visibility 0.3s;

		&::after {
			content: '';
			display: block;
			position: absolute;
			left: 50%;
			top: -10px;
			width: 0;
			height: 0;
			border-left: 10px solid transparent;
			border-right: 10px solid transparent;
			border-bottom: 10px solid #CCC;
		}
	}

	.wrapper--hidden .map-window {
		opacity: 0;
		visibility: hidden;
	}

	.map-window-js {
		width: 100%;
		height: 100%;
	}
	.wrapper {
		position: relative;
		transition: opacity 0.3s, visibility 0.3s;
	}
	.the-icon {
		font-family: Material Icons;
		display: inline-block;
		font-size: 20px;
		vertical-align: middle;
	}
	.the-label {
		text-decoration: underline;
	}
	.map-placeholder {
		background-color: #DDD;
		color: rgba(0, 0, 0, 0.5);
		width: 100%;
		height: 100%;
		display: flex;
		align-items: center;
		justify-content: center;
	}
</style>
