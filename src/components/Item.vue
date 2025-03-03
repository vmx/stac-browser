<template>
  <div>
    <b-container :class="loaded && 'loaded'">
      <b-row>
        <b-col md="12">
          <header>
            <!-- <a href="https://www.planet.com/disasterdata/">
              <img
                id="header_logo"
                src="https://planet-pulse-assets-production.s3.amazonaws.com/uploads/2016/06/blog-logo.jpg"
                alt="Powered by Planet Labs"
                class="float-right">
            </a>-->
            <div>
              <b-breadcrumb :items="breadcrumbs" />
            </div>
          </header>
        </b-col>
      </b-row>
      <b-row>
        <b-col md="8">
          <h1 class="scroll">{{ title }}</h1>
          <p class="scroll">
            <small>
              <b-button
                v-clipboard="url"
                variant="link"
                size="sm"
                class="clipboard"
              >
                <span
                  v-if="validationErrors"
                  title="Validation errors present; please check the JavaScript Console"
                  >⚠️</span
                >
                <i class="far fa-copy" />&nbsp;
                <code>{{ url }}</code>
              </b-button>
            </small>
          </p>
          <b-tabs v-model="tabIndex">
            <b-tab
              v-if="visibleTabs.includes('preview')"
              title="Preview"
              active
            >
              <div id="map-container">
                <div id="map"></div>
              </div>
              <multiselect
                v-if="cogs.length > 1"
                v-model="selectedImage"
                :options="cogs"
                placeholder="Select an image"
                track-by="href"
                label="title"
              />
            </b-tab>
            <b-tab
              v-if="visibleTabs.includes('thumbnail')"
              title="Thumbnail"
              :active="!visibleTabs.includes('preview')"
            >
              <a :href="thumbnail">
                <img id="thumbnail" align="center" :src="thumbnail" />
              </a>
            </b-tab>
            <b-tab
              v-if="visibleTabs.includes('assets')"
              title="Assets"
              :active="
                !visibleTabs.includes('preview') &&
                  !visibleTabs.includes('thumbnail')
              "
            >
              <div class="table-responsive assets">
                <table class="table table-striped">
                  <thead>
                    <tr>
                      <th>Name</th>
                      <th v-if="bands.length > 0">Band(s)</th>
                      <th>Content-Type</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="asset in assets" :key="asset.key">
                      <td>
                        <!-- eslint-disable-next-line vue/max-attributes-per-line vue/no-v-html -->
                        <a
                          :href="asset.href"
                          :title="asset.key"
                          v-html="asset.label"
                        />
                      </td>
                      <td v-if="bands.length > 0">{{ asset.bandNames }}</td>
                      <td>
                        <code>{{ asset.type }}</code>
                      </td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </b-tab>
            <b-tab
              v-if="visibleTabs.includes('bands')"
              title="Bands"
              :active="
                !visibleTabs.includes('preview') &&
                  !visibleTabs.includes('thumbnail') &&
                  !visibleTabs.includes('assets')
              "
            >
              <b-table
                :items="bands"
                :fields="bandFields"
                responsive
                small
                striped
              />
            </b-tab>
          </b-tabs>
        </b-col>

        <b-col md="4">
          <b-card bg-variant="light">
            <div id="locator-map" />
            <div class="table-responsive metadata">
              <table class="table">
                <tbody>
                  <tr v-if="collection">
                    <td class="title">Collection</td>
                    <td>
                      <router-link :to="linkToCollection">
                        {{ collection.title || "Untitled" }}
                      </router-link>
                    </td>
                  </tr>
                  <tr v-if="license">
                    <td class="title">License</td>
                    <td>
                      <!-- eslint-disable-next-line vue/no-v-html -->
                      <span v-html="license"></span>
                      <template v-if="licensor"
                        >by
                        <!-- eslint-disable-next-line vue/no-v-html -->
                        <span v-html="licensor"></span>
                      </template>
                    </td>
                  </tr>
                  <tr v-for="prop in propertyList" :key="prop.key">
                    <td class="title">
                      <!-- eslint-disable-next-line vue/no-v-html -->
                      <span :title="prop.key" v-html="prop.label" />
                    </td>
                    <!-- eslint-disable-next-line vue/no-v-html -->
                    <td v-html="prop.value" />
                  </tr>
                </tbody>
              </table>
            </div>
          </b-card>
        </b-col>
      </b-row>
    </b-container>
    <footer class="footer">
      <b-container>
        <span class="poweredby text-muted">
          Powered by
          <a href="https://github.com/radiantearth/stac-browser"
            >STAC Browser</a
          >
        </span>
      </b-container>
    </footer>
  </div>
</template>

<script>
import path from "path";
import url from "url";

import escape from "lodash.escape";
import isEqual from "lodash.isequal";
import Leaflet from "leaflet";
import "leaflet-easybutton";
import { mapActions, mapGetters } from "vuex";

import common from "./common";

const COG_TYPES = [
  "image/vnd.stac.geotiff; cloud-optimized=true",
  "image/x.geotiff" // generated by sat-api
];

export default {
  ...common,
  name: "ItemDetail",
  props: {
    ancestors: {
      type: Array,
      required: true
    },
    center: {
      type: Array,
      default: null
    },
    path: {
      type: String,
      required: true
    },
    resolve: {
      type: Function,
      required: true
    },
    slugify: {
      type: Function,
      required: true
    },
    url: {
      type: String,
      required: true
    },
    validate: {
      type: Function,
      required: true
    }
  },
  data() {
    return {
      fullscreen: false,
      locatorMap: null,
      map: null,
      tileLayer: null,
      overlayLayer: null,
      locatorOverlayLayer: null,
      geojsonOptions: {
        style: function() {
          return {
            weight: 2,
            color: "#333",
            opacity: 1,
            fillOpacity: 0
          };
        }
      },
      selectedImage: null,
      selectedTab: null,
      validationErrors: null
    };
  },
  computed: {
    ...common.computed,
    ...mapGetters(["getEntity"]),
    _collectionLinks() {
      return this.links.filter(x => x.rel === "collection");
    },
    _description() {
      return this._properties.description;
    },
    _keywords() {
      return (
        (this.collection && this.collection.keywords) ||
        common.computed._keywords.apply(this)
      );
    },
    _license() {
      return (
        this._properties["item:license"] ||
        (this.collection && this.collection.license) ||
        (this.rootCatalog && this.rootCatalog.license)
      );
    },
    _providers() {
      return (
        this._properties["item:providers"] ||
        (this.collection && this.collection.providers) ||
        common.computed._providers.apply(this)
      );
    },
    _temporalCoverage() {
      if (this._properties["dtr:start_datetime"] != null) {
        return [
          this._properties["dtr:start_datetime"],
          this._properties["dtr:end_datetime"]
        ]
          .map(x => x || "..")
          .join("/");
      }

      return this._properties.datetime;
    },
    _title() {
      return this._properties.title;
    },
    assets() {
      return (
        Object.keys(this.item.assets)
          .map(key => ({
            ...this.item.assets[key],
            key
          }))
          .map(x => ({
            ...x,
            bands: (x["eo:bands"] || []).map(idx => this.bands[idx]),
            title: x.title || path.basename(x.href),
            label:
              escape(x.title) ||
              `<code>${escape(path.basename(x.href))}</code>`,
            href: this.resolve(x.href, this.url)
          }))
          .map(x => ({
            ...x,
            bandNames: x.bands
              .map(band =>
                band != null
                  ? band.description || band.common_name || band.name
                  : null
              )
              .filter(x => x != null)
              .join(", ")
          }))
          // prioritize assets w/ a format set
          .sort((a, b) => {
            const formatA = a.format || "zzz";
            const formatB = b.format || "zzz";

            if (formatA < formatB) {
              return -1;
            }

            if (formatA > formatB) {
              return 1;
            }

            return 0;
          })
      );
    },
    attribution() {
      if (this.license != null || this.licensor != null) {
        return `Imagery ${this.license || ""} ${this.licensor || ""}`;
      }

      return null;
    },
    bands() {
      return (
        this._properties["eo:bands"] ||
        (this.collection &&
          this.collection.properties &&
          this.collection.properties["eo:bands"]) ||
        (this.rootCatalog &&
          this.rootCatalog.properties &&
          this.rootCatalog.properties["eo:bands"]) ||
        []
      );
    },
    cog() {
      return (
        this.cogs
          .slice()
          // prefer COGs with "visual" key
          .sort((a, b) => b.key.indexOf("visual") - a.key.indexOf("visual"))
          .shift()
      );
    },
    cogs() {
      return this.assets
        .filter(x => COG_TYPES.includes(x.type))
        .map(cog => ({
          ...cog,
          title:
            cog.bandNames.length > 0
              ? `${cog.title} (${cog.bandNames})`
              : cog.title
        }));
    },
    collection() {
      if (this.collectionLink != null) {
        this.load(this.collectionLink.href);

        return this.getEntity(this.collectionLink.href);
      }

      return null;
    },
    collectionLink() {
      return this._collectionLinks
        .map(x => ({
          ...x,
          href: this.resolve(x.href, this.url),
          slug: this.slugify(this.resolve(x.href, this.url))
        }))
        .pop();
    },
    entity() {
      return this.item;
    },
    item() {
      if (this._entity.type === "FeatureCollection") {
        const { hash } = url.parse(this.url);
        const idx = hash.slice(1);

        return this._entity.features[idx];
      }

      return this._entity;
    },
    jsonLD() {
      const dataset = {
        "@context": "https://schema.org/",
        "@type": "Dataset",
        // required
        name: this.title,
        description: this.description || `${this.title} STAC Item`,
        // recommended
        citation: this._properties["sci:citation"],
        identifier: this._properties["sci:doi"] || this.item.id,
        keywords: this._keywords,
        license: this.licenseUrl,
        isBasedOn: this.url,
        url: this.path,
        workExample: (this._properties["sci:publications"] || []).map(p => ({
          identifier: p.doi,
          citation: p.citation
        })),
        includedInDataCatalog: [this.collectionLink, this.parentLink]
          .filter(x => !!x)
          .map(l => ({
            isBasedOn: l.href,
            url: l.slug
          })),
        spatialCoverage: {
          "@type": "Place",
          geo: {
            "@type": "GeoShape",
            box: (this.item.bbox || []).join(" ")
          }
        },
        temporalCoverage: this._temporalCoverage,
        distribution: this.assets.map(a => ({
          contentUrl: a.href,
          fileFormat: a.type,
          name: a.title
        })),
        image: this.thumbnail
      };

      return dataset;
    },
    licensor() {
      return this.providers
        .filter(x => (x.roles || []).includes("licensor"))
        .map(x => {
          if (x.url != null) {
            return `<a href=${x.url}>${x.name}</a>`;
          }

          return x.name;
        })
        .pop();
    },
    linkToCollection() {
      if (this.collectionLink.href != null) {
        return `/collection/${this.slugify(this.collectionLink.href)}`;
      }

      return null;
    },
    parentLink() {
      return this.links
        .filter(x => x.rel === "parent")
        .map(x => ({
          ...x,
          href: this.resolve(x.href, this.url),
          slug: this.slugify(this.resolve(x.href, this.url))
        }))
        .pop();
    },
    tabIndex: {
      get: function() {
        return this.visibleTabs.indexOf(this.selectedTab);
      },
      set: function(tabIndex) {
        // wait for the DOM to update
        this.$nextTick(() => {
          this.selectedTab = this.visibleTabs[tabIndex];
        });
      }
    },
    thumbnail() {
      const thumbnail = this.assets.find(x => x.key === "thumbnail");

      if (thumbnail != null) {
        return this.resolve(thumbnail.href, this.url);
      }

      return null;
    },
    tileSource() {
      if (this.selectedImage == null) {
        return "";
      }

      // TODO global config
      return `https://tiles.rdnt.io/tiles/{z}/{x}/{y}@2x?url=${encodeURIComponent(
        this.selectedImage.href
      )}`;
      // return `http://localhost:8000/tiles/{z}/{x}/{y}@2x?url=${encodeURIComponent(
      //   this.cog
      // )}`;
    },
    visibleTabs() {
      return [
        this.cogs.length > 0 && "preview",
        this.thumbnail != null && "thumbnail",
        this.assets.length > 0 && "assets",
        this.bands.length > 0 && "bands"
      ].filter(x => x != null && x !== false);
    }
  },
  watch: {
    ...common.watch,
    fullscreen(to, from) {
      if (to !== from) {
        if (to) {
          this.map.getContainer().classList.add("leaflet-pseudo-fullscreen");
        } else {
          this.map.getContainer().classList.remove("leaflet-pseudo-fullscreen");
        }

        this.updateState({
          fullscreen: to
        });

        this.map.invalidateSize();
      }
    },
    selectedImage(to, from) {
      if (!isEqual(to, from)) {
        const idx = this.cogs.indexOf(to);

        if (idx >= 0) {
          this.updateState({
            si: idx
          });
        } else {
          this.updateState({
            si: null
          });
        }
      }
    },
    selectedTab(to, from) {
      if (to !== from) {
        this.updateState({
          t: to
        });

        if (to === "preview") {
          this.map && this.map.invalidateSize();
        }
      }
    },
    tileSource(to, from) {
      if (to !== from) {
        if (this.map != null) {
          if (this.tileLayer != null) {
            this.tileLayer.removeFrom(this.map);
          }

          this.tileLayer = Leaflet.tileLayer(to, {
            attribution: this.attribution
          }).addTo(this.map);
        }
      }
    }
  },
  methods: {
    ...common.methods,
    ...mapActions(["load"]),
    initialize() {
      this.syncWithQueryState(this.$route.query);

      this.$nextTick(() => {
        if (this.cog != null) {
          this.selectedImage = this.selectedImage || this.cog;

          this.initializePreviewMap();
        }
        this.initializeLocatorMap();
      });
    },
    initializeLocatorMap() {
      if (this.locatorMap == null) {
        this.locatorMap = Leaflet.map("locator-map", {
          attributionControl: false,
          zoomControl: false,
          boxZoom: false,
          doubleClickZoom: false,
          dragging: false,
          scrollWheelZoom: false,
          touchZoom: false
        });

        Leaflet.tileLayer(
          "https://{s}.basemaps.cartocdn.com/dark_nolabels/{z}/{x}/{y}@2x.png",
          {
            attribution: `Map data <a href="https://www.openstreetmap.org/copyright">&copy; OpenStreetMap contributors</a>, &copy; <a href="https://carto.com/attributions">CARTO</a>`
          }
        ).addTo(this.locatorMap);
        Leaflet.tileLayer(
          "https://{s}.basemaps.cartocdn.com/dark_only_labels/{z}/{x}/{y}@2x.png",
          {
            opacity: 0.6,
            zIndex: 1000
          }
        ).addTo(this.locatorMap);
      } else {
        this.locatorOverlayLayer.removeFrom(this.locatorMap);
      }

      this.locatorOverlayLayer = Leaflet.geoJSON(this.item, {
        pane: "tilePane",
        style: {
          weight: 1,
          color: "#ffd65d",
          opacity: 1,
          fillOpacity: 1
        }
      }).addTo(this.locatorMap);

      this.locatorMap.fitBounds(this.locatorOverlayLayer.getBounds(), {
        padding: [95, 95]
      });
    },
    initializePreviewMap() {
      if (this.map == null) {
        this.map = Leaflet.map("map", {
          scrollWheelZoom: false
        });

        this.map.on("moveend", this.updateHash);
        this.map.on("zoomend", this.updateHash);

        this.map.attributionControl.setPrefix("");

        this.button = Leaflet.easyButton(
          "fas fa-expand fa-2x",
          () => (this.fullscreen = !this.fullscreen),
          {
            position: "topright"
          }
        ).addTo(this.map);

        if (this.fullscreen) {
          this.map.getContainer().classList.add("leaflet-pseudo-fullscreen");
        }

        Leaflet.tileLayer(
          "https://{s}.basemaps.cartocdn.com/dark_nolabels/{z}/{x}/{y}@2x.png",
          {
            attribution: `Map data <a href="https://www.openstreetmap.org/copyright">&copy; OpenStreetMap contributors</a>, &copy; <a href="https://carto.com/attributions">CARTO</a>`
          }
        ).addTo(this.map);
        Leaflet.tileLayer(
          "https://{s}.basemaps.cartocdn.com/dark_only_labels/{z}/{x}/{y}@2x.png",
          {
            zIndex: 1000
          }
        ).addTo(this.map);
      } else {
        if (this.tileLayer != null) {
          this.tileLayer.removeFrom(this.map);
        }

        this.overlayLayer.removeFrom(this.map);
      }

      if (this.tileSource) {
        this.tileLayer = Leaflet.tileLayer(this.tileSource, {
          attribution: this.attribution
        }).addTo(this.map);
      }

      this.overlayLayer = Leaflet.geoJSON(this.item, {
        ...this.geojsonOptions,
        pane: "tilePane"
      }).addTo(this.map);

      if (this.center != null) {
        const [zoom, lat, lng] = this.center;

        this.map.setView([lat, lng], zoom);
      } else {
        this.map.fitBounds(this.overlayLayer.getBounds());
      }
    },
    syncWithQueryState(query) {
      this.selectedTab = query.t;

      if (query.si != null) {
        this.selectedImage = this.cogs[query.si];
      } else {
        this.selectedImage = this.cog;
      }

      this.fullscreen = [true, "true"].includes(query.fullscreen);
    },
    updateHash() {
      const center = this.map.getCenter();
      const zoom = this.map.getZoom();

      this.$router.replace({
        ...this.$route,
        hash: `${zoom}/${center.lat.toFixed(6)}/${center.lng.toFixed(6)}`
      });
    }
  }
};
</script>

<style lang="css">
html {
  position: relative;
  min-height: 100%;
}

body {
  line-height: 24px;
  color: #111;
  font-family: Arial, sans-serif;
  margin-bottom: 40px;
}

blockquote,
body {
  font-size: 16px;
}
table {
  font-size: 80%;
}
h1,
h2,
h3,
h4,
h5,
h6 {
  padding: 0;
  margin: 0;
}
h1,
h2,
h3,
h4 {
  font-family: Arial, sans-serif;
  text-rendering: optimizeLegibility;
  padding-bottom: 4px;
}
h1:last-child,
h2:last-child,
h3:last-child,
h4:last-child {
  padding-bottom: 0;
}
h1 {
  font-weight: 400;
  font-size: 28px;
  line-height: 1.2;
}
h2 {
  font-weight: 700;
  font-size: 21px;
  line-height: 1.3;
}
h3 {
  font-weight: 700;
}
h3,
h4 {
  font-size: 17px;
  line-height: 1.255;
}
h4 {
  font-weight: 400;
}
h5 {
  font-size: 13px;
  line-height: 19px;
}
h5,
h6 {
  font-weight: 700;
}
h6 {
  text-transform: uppercase;
  font-size: 11px;
  line-height: 1.465;
  padding-bottom: 1px;
}
p {
  padding: 0;
  margin: 0 0 14px;
}
p:last-child {
  margin-bottom: 0;
}
p + p {
  margin-top: -4px;
}
b,
strong {
  font-weight: 700;
}
em,
i {
  font-style: italic;
}
blockquote {
  margin: 13px;
}
small {
  font-size: 12px;
}
a,
a:active,
a:link,
a:visited {
  color: #0066c0;
}
header {
  padding: 1.5em 0 0.5em;
}
code {
  color: #555;
  white-space: nowrap;
}

.scroll {
  overflow-x: scroll;
  -ms-overflow-style: none;
  overflow: -moz-scrollbars-none;
  scrollbar-width: none;
}

.scroll::-webkit-scrollbar {
  display: none;
}

.btn code {
  font-size: 10.5px;
}

.btn.clipboard {
  white-space: nowrap;
}

.btn.clipboard:hover {
  text-decoration: none;
}

.footer {
  position: absolute;
  bottom: 0;
  width: 100%;
  height: 40px;
  line-height: 40px;
  text-align: right;
  font-size: 0.75em;
}

.poweredby {
  border: 1px dotted hotpink;
  border-radius: 4px;
  padding: 5px 10px;
  background-color: #f9f9f9;
}

.tabs {
  margin-top: 25px;
}
</style>

<style scoped lang="css">
.leaflet-pseudo-fullscreen {
  position: fixed !important;
  width: 100% !important;
  height: 100% !important;
  top: 0 !important;
  left: 0 !important;
  z-index: 99999;
}

.leaflet-container {
  background-color: #262626;
}

#locator-map {
  height: 200px;
  width: 100%;
  margin-bottom: 10px;
}

#map-container {
  height: 500px;
}

#map {
  height: 100%;
  width: 100%;
}

#thumbnail {
  display: block;
  margin-left: auto;
  margin-right: auto;
  max-height: 500px;
}

#header_logo {
  max-width: 100px;
}

.table th,
.table td {
  border: none;
  padding: 0.25rem;
}

.table th {
  border-top: none;
  border-bottom: 1px solid #dee2e6;
}

.metadata td.title {
  font-weight: bold;
  width: 33%;
}

.table-responsive.assets {
  padding: 15px;
}

.table-responsibe.metadata {
  background-color: #f9f9f9;
  border: 1px solid #dee2e6;
  padding: 10px;
}

.multiselect {
  margin-top: 5px;
}
</style>
