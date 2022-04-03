<template>
    <div id="app">
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
            />
            <l-tile-layer
                :url="url"
                :attribution="attribution"
            />
            <!--<l-layer-group v-if="geojsonHemodinamicas !== null">
                <l-marker
                    v-for="(item, index) in geojsonHemodinamicas.features"
                    :key="index"
                    :lat-lng.sync="item.geometry.coordinates">
                </l-marker>
            </l-layer-group>-->
            <l-geo-json
                ref="isochromy60"
                :geojson="geojsonIsochrone60"
                :options-style="layer60">
            </l-geo-json>
            <l-layer-group
                layer-type="overlay"
                name="Hemodinâmicas">
                <l-geo-json :geojson="geojsonHemodinamicas"></l-geo-json>
            </l-layer-group>
            <l-geo-json :geojson="geojsonIsochrone30" :options-style="styleFunction"></l-geo-json>
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
import { LMap, LTileLayer, LGeoJson, LControlLayers, LLayerGroup } from 'vue2-leaflet';

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
        LLayerGroup
    },
    data: function() {
        return {
            zoom: 8,
            path: "/images/",
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
        // showing the name of the province
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
                ];

                Promise.all(
                    requests.map((request) => axios.get(request))
                ).then((
                    [
                        {data: response60},
                    ])=> {
                        this.setIsochrone60(response60);
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
                        fillColor: "#3388ff",
                        weight: 1,
                        color: "#3388ff",
                        fillOpacity: 0.2
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
                    fillColor: "#3388ff",
                    weight: 1,
                    color: "#3388ff",
                    fillOpacity: 0.2
                };
            };
        },
        styleFunction() {
            return () => {
                return {
                    weight: 2,
                    color: "#ECEFF1",
                    opacity: 1,
                    fillColor: "#FF00FF",
                    fillOpacity: 0.5
                };
            };
        },
        layer60 () {
            return () => {
                return {
                    weight: 2,
                    color: "#ECEFF1",
                    opacity: 1,
                    fillColor: "#FFFF00",
                    fillOpacity: 0.5
                };
            };
        },
    },
    async created() {
        let requests = [
            '/pontos/ambulancias.geojson',
            '/pontos/hemodinamicas.geojson',
            '/pontos/reperfusao_quimica.geojson',
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
            }
        );
    }
};
</script>

<style>
body {
    margin: 0;
}
</style>