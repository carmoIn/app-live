<template>
    <div id="app">
        <div
            v-if="loading"
            class="loading-overlay">
            <div class="lds-dual-ring"></div>
        </div>
        <l-map
            ref="map"
            :zoom="zoom"
            :center="center"
            style="height: 100vh; width: 100%"
        >
            <l-control-layers
                position="topright"
                :collapsed="false"
                :sort-layers="true"
            >
            </l-control-layers>
            <l-control
                class="legend-control"
                position="bottomright">
                <div class="legend-content">
                    <h3 class="legend-title">Legenda</h3>
                    <div class="legend-item">
                        <i>
                            <img src="icons/ambulance.png" style="left: 8px; top: 0px;" width="24">
                        </i>
                        <span>Ambulância</span>
                    </div>
                    <div class="legend-item">
                        <i>
                            <img src="icons/reperfusion.png" style="left: 8px; top: 0px;" width="24">
                        </i>
                        <span>Reperfusão Química</span>
                    </div>
                    <div class="legend-item">
                        <i>
                            <img src="icons/hemodynamic.png" style="left: 8px; top: 0px;" width="24">
                        </i>
                        <span>Hemodinâmica</span>
                    </div>
                    <div class="legend-item">
                        <i style="width: 24px; height: 18px;">
                            <canvas id="legend30"
                                    height="18" width="24"
                                    style="border: 1px solid black;background: #6060f4;">
                            </canvas>
                        </i>
                        <span>até 30 minutos</span>
                    </div>
                    <div class="legend-item">
                        <i style="width: 24px; height: 18px;">
                            <canvas id="legend60"
                                    height="18" width="24"
                                    style="border: 1px solid black;background: #aaa7ee;">
                            </canvas>
                        </i>
                        <span>até 60 minutos</span>
                    </div>
                </div>
            </l-control>
            <l-tile-layer
                :url="url"
                :attribution="attribution"
            />
            <l-layer-group
                layer-type="overlay"
                name="Ambulânicas"
                v-if="geojsonAmbulancia !== null">
                <l-marker
                    v-for="(item, index) in geojsonAmbulancia"
                    :key="index"
                    :lat-lng="item.posicao">
                    <l-icon
                        :icon-size=[24,24]
                        :iconUrl= "'icons/ambulance.png'">
                    </l-icon>
                    <l-tooltip :content="item.nome_municipio"></l-tooltip>
                </l-marker>
            </l-layer-group>
            <l-layer-group
                layer-type="overlay"
                name="Hemodinâmicas"
                v-if="geojsonHemodinamicas !== null">
                <l-marker
                    v-for="(item, index) in geojsonHemodinamicas"
                    :key="index"
                    :lat-lng="item.posicao">
                    <l-icon
                        :icon-size=[24,24]
                        :iconUrl= "'icons/hemodynamic.png'">
                    </l-icon>
                    <l-tooltip :content="item.nome_municipio"></l-tooltip>
                </l-marker>
            </l-layer-group>
            <l-layer-group
                layer-type="overlay"
                name="Reperfusão Química"
                v-if="geojsonReperfusao !== null">
                <l-marker
                    v-for="(item, index) in geojsonReperfusao"
                    :key="index"
                    :lat-lng="item.posicao">
                    <l-icon
                        :icon-size=[24,24]
                        :iconUrl= "'icons/reperfusion.png'">
                    </l-icon>
                    <l-tooltip :content="item.nome_municipio"></l-tooltip>
                </l-marker>
            </l-layer-group>
            <l-geo-json
                ref="isochromy60"
                :geojson="geojsonIsochrone60"
                :options-style="styleLayer60">
            </l-geo-json>
            <l-geo-json
                :geojson="geojsonIsochrone30"
                :options-style="styleLayer30">
            </l-geo-json>
            <l-layer-group
                layer-type="overlay"
                name="Municípios">
                <l-geo-json :geojson="counties"
                            :options="optionsMunicipios"
                            :options-style="styleMunicipio"
                            @update:ready="zoomToCounty">
                </l-geo-json>
            </l-layer-group>

        </l-map>
    </div>
</template>

<script>
import "leaflet/dist/leaflet.css"

//import L from "leaflet";
import axios from 'axios';
import { latLng, Icon } from 'leaflet';
import { LMap, LTileLayer, LGeoJson, LControlLayers, LLayerGroup, LControl, LMarker, LIcon, LTooltip } from 'vue2-leaflet';

delete Icon.Default.prototype._getIconUrl;
Icon.Default.mergeOptions({
    iconRetinaUrl: require('leaflet/dist/images/marker-icon-2x.png'),
    iconUrl: require('leaflet/dist/images/marker-icon.png'),
    shadowUrl: require('leaflet/dist/images/marker-shadow.png'),
});

export default {
    name: "CustomPath",
    components: {
        LMap,
        LTileLayer,
        LGeoJson,
        LControlLayers,
        LControl,
        LLayerGroup,
        LMarker,
        LIcon,
        LTooltip
    },
    data: function() {
        return {
            loading: true,
            zoom: 8,
            path: "/icons/",
            center: [-24.637, -51.927],
            url: 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
            attribution:
                '&copy; <a href="http://osm.org/copyright">OpenStreetMap</a> contributors',
            marker: latLng(47.41322, -1.219482),
            geojsonAmbulancia: null,
            geojsonHemodinamicas: null,
            geojsonReperfusao: null,
            geojsonIsochrone30: null,
            geojsonIsochrone60: null,
            counties: null
        };
    },
    methods: {
        setIsochrone30: function (geojson) {
            this.geojsonIsochrone30 = geojson;
        },
        setIsochrone60: function (geojson) {
            this.geojsonIsochrone60 = geojson;
            this.zoomToCounty();
        },
        showPopupCounty: function(feature, layer) {
            if (feature.properties && feature.properties.NM_MUN) {
                layer.bindPopup(feature.properties.NM_MUN);
            }
        },
        zoomToCounty: function () {
            setTimeout(() => {
                const map = this.$refs.map.mapObject;
                const isochromy60 = this.$refs.isochromy60.mapObject;
                if (map) {
                    map.fitBounds(isochromy60.getBounds());
                }
            }, 200);
        },
        laodLayers: async function (feature) {
            const geoid = feature.properties.geoid;
            if (geoid) {
                let requests = [
                    `/60/${geoid}.geojson`,
                    `/30/${geoid}.geojson`,
                ];

                Promise.all(
                    requests.map((request) => axios.get(request))
                ).then((
                    [
                        {data: response60},
                        {data: response30}
                    ])=> {
                        this.setIsochrone60(response60);
                        this.setIsochrone30(response30);
                    }
                );
            }
        }
    },
    computed: {
        optionsMunicipios() {
            return {
                onEachFeature: this.onEachFeatureFunction
            };
        },
        onEachFeatureFunction() {
            return (feature, layer) => {
                let self = this;

                layer.on("mouseover", function () {
                    // bindPopup
                    self.showPopupCounty(feature, layer);
                    this.openPopup();
                    // style
                    this.setStyle({
                        fillColor: "#eb4034",
                        weight: 1,
                        color: "#eb4034",
                        fillOpacity: 0.7
                    });
                });
                layer.on("mouseout", function () {
                    this.closePopup();
                    // style
                    this.setStyle({
                        fillColor: "#1d1e1f",
                        weight: 1,
                        color: "#1d1e1f",
                        fillOpacity: 0.1
                    });
                });
                layer.on("click", function () {
                    self.laodLayers(feature);
                });
            }
        },
        styleMunicipio() {
            return () => {
                return {
                    fillColor: "#1d1e1f",
                    weight: 1,
                    color: "#1d1e1f",
                    fillOpacity: 0.1
                };
            };
        },
        styleLayer30() {
            return () => {
                return {
                    weight: 2,
                    color: "#000",
                    opacity: 1,
                    fillColor: "#6060f4",
                    fillOpacity: 0.5
                };
            };
        },
        styleLayer60 () {
            return () => {
                return {
                    weight: 2,
                    color: "#000",
                    opacity: 1,
                    fillColor: "#aaa7ee",
                    fillOpacity: 0.5
                };
            };
        },
    },
    createLegend: function () {
    },
    async created() {
        let requests = [
            '/pontos/ambulancias.json',
            '/pontos/hemodinamicas.json',
            '/pontos/reperfusao_quimica.json',
            '/municipios.geojson'
        ];

        Promise.all(
            requests.map((request) => axios.get(request))
        ).then((
            [
                {data: ambulances},
                {data: hemodynamics},
                {data: reperfusion},
                {data: counties}
            ])=> {
                this.geojsonAmbulancia = ambulances;
                this.geojsonHemodinamicas = hemodynamics;
                this.geojsonReperfusao = reperfusion;
                this.counties = counties;
                this.loading = false;
            }
        );
    }
};
</script>

<style>
body {
    margin: 0;
}
.legend-control {
    display: flex;
    background: white;
    border: 2px solid rgba(0,0,0,0.2);
    background-clip: padding-box;
}
.legend-content {
    display: block;
    padding: 1rem;
}
.legend-title {
    margin: 3px;
    padding-bottom: 5px;
}
.legend-item {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 0.25rem;
}
.loading-overlay {
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
    background: rgba(0, 0, 0, 0.8);
    z-index: 1000;
    display: flex;
    justify-content: center;
    align-items: center;
}
.lds-dual-ring {
    display: inline-block;
    width: 80px;
    height: 80px;
}
.lds-dual-ring:after {
    content: " ";
    display: block;
    width: 64px;
    height: 64px;
    margin: 8px;
    border-radius: 50%;
    border: 6px solid #fff;
    border-color: #fff transparent #fff transparent;
    animation: lds-dual-ring 1.2s linear infinite;
}
@keyframes lds-dual-ring {
    0% {
        transform: rotate(0deg);
    }
    100% {
        transform: rotate(360deg);
    }
}

</style>