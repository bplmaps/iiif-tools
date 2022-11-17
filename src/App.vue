<template>
  <section id="main">
    <div class="container is-fluid">
      <div class="box">
        <div class="field has-addons">
          <div class="control is-expanded">
            <input
              class="input"
              type="text"
              v-model="enteredUrl"
              placeholder="Enter a LMEC Digital Collections or Digital Commonwealth URL, or an external Manifest"
            />
          </div>
          <div class="control">
            <a class="button is-primary" @click="resolveEnteredUrl">
              Open this URL
            </a>
          </div>
        </div>
      </div>
    </div>

    <div class="container is-fluid" v-if="error" id="error">
      <div class="notification is-warning is-light">{{ errorMessage }}</div>
    </div>

    <div
      class="container is-fluid"
      v-if="manifestLoaded && sourceType === 'commonwealth'"
    >
      <div class="box">
        <h2 class="is-size-4 mb-3">Object Information</h2>

        <div class="field has-addons">
          <p class="control">
            <a class="button is-link" :href="dcObjectUrl" target="_blank">
              Digital Commonwealth URL
            </a>
          </p>
          <p class="control is-expanded">
            <input
              class="input"
              type="text"
              readonly
              :value="dcObjectUrl"
              id="dcObjectUrl"
            />
          </p>
          <p class="control">
            <button class="button is-secondary" v-clipboard="() => dcObjectUrl">
              📋
            </button>
          </p>
        </div>

        <div class="field has-addons">
          <p class="control">
            <a class="button is-link" :href="lmecObjectUrl" target="_blank">
              LMEC Collections URL
            </a>
          </p>
          <p class="control is-expanded">
            <input class="input" type="text" readonly :value="lmecObjectUrl" />
          </p>

          <p class="control">
            <button
              class="button is-secondary"
              v-clipboard="() => lmecObjectUrl"
            >
              📋
            </button>
          </p>
        </div>

        <div class="field has-addons">
          <p class="control">
            <a class="button is-link" :href="arkLink" target="_blank">
              ARK Permalink
            </a>
          </p>
          <p class="control is-expanded">
            <input class="input" type="text" readonly :value="arkLink" />
          </p>

          <p class="control">
            <button class="button is-secondary" v-clipboard="() => arkLink">
              📋
            </button>
          </p>
        </div>

        <div class="field has-addons">
          <p class="control">
            <a class="button is-link" :href="manifestUrl" target="_blank">
              IIIF Manifest
            </a>
          </p>
          <p class="control is-expanded">
            <input class="input" type="text" readonly :value="manifestUrl" />
          </p>

          <p class="control">
            <button class="button is-secondary" v-clipboard="() => manifestUrl">
              📋
            </button>
          </p>
        </div>
      </div>
    </div>

    <div
      class="container is-fluid"
      :class="[manifestLoaded ? '' : 'is-hidden']"
    >
      <div class="box">
        <h2 class="is-size-4">Images</h2>
        <ul>
          <li v-for="(sequence, seqIndex) in manifest.sequences">
            Sequence {{ seqIndex }}
            <ul>
              <li v-for="(canvas, canIndex) in sequence.canvases">
                Canvas {{ canIndex }} ({{ canvas.height }} ×{{ canvas.width }})

                <ul>
                  <li v-for="(image, imgIndex) in canvas.images">
                    Image {{ imgIndex }}
                    <button
                      class="button is-small is-primary"
                      @click="loadImage(seqIndex, canIndex, imgIndex)"
                    >
                      Load into viewer
                    </button>
                  </li>
                </ul>
              </li>
            </ul>
          </li>
        </ul>
      </div>
    </div>

    <div
      class="container is-fluid"
      :class="[state === 'imageLoaded' ? '' : 'is-hidden']"
    >
      <div class="box">
        <h2 class="is-size-4">
          Sequence {{ this.loadedImg[0] }}, Canvas {{ this.loadedImg[1] }},
          Image {{ this.loadedImg[2] }}
        </h2>

        <div class="field has-addons">
          <p class="control">
            <a class="button is-link" :href="loadedImgIIIF" target="_blank">
              IIIF Image Endpoint
            </a>
          </p>
          <p class="control is-expanded">
            <input class="input" type="text" readonly :value="loadedImgIIIF" />
          </p>

          <p class="control">
            <button
              class="button is-secondary"
              v-clipboard="() => loadedImgIIIF"
            >
              📋
            </button>
          </p>
        </div>

        <div class="field is-horizontal">
          <div class="field-label is-normal">
            <label class="label">Image Quality</label>
          </div>
          <div class="field-body">
            <div class="field is-narrow">
              <div class="control">
                <div class="select is-fullwidth">
                  <select v-model="imgType">
                    <option>default</option>
                    <option>color</option>
                    <option>gray</option>
                    <option>bitonal</option>
                  </select>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="field is-horizontal">
          <div class="field-label is-normal">
            <label class="label">Image Resolution</label>
          </div>
          <div class="field-body">
            <div class="field is-narrow">
              <div class="control">
                <div class="select is-fullwidth">
                  <select v-model="imgResolutionType">
                    <option>full</option>
                    <option>Maximum height</option>
                    <option>Maximum width</option>
                    <option>Percent</option>
                  </select>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="field is-horizontal" v-if="imgResolutionType != 'full'">
          <div class="field-label is-normal">
            <label class="label">{{ imgResolutionType }}</label>
          </div>
          <div class="field-body">
            <div class="field is-narrow">
              <div class="control">
                <input
                  class="input"
                  type="text"
                  v-model="imgResolutionVariable"
                />
              </div>
            </div>
          </div>
        </div>

        <div class="field has-addons">
          <p class="control">
            <a class="button is-link" :href="uncroppedImage" target="_blank">
              Uncropped Image
            </a>
          </p>
          <p class="control is-expanded">
            <input class="input" type="text" readonly :value="uncroppedImage" />
          </p>

          <p class="control">
            <button
              class="button is-secondary"
              v-clipboard="() => uncroppedImage"
            >
              📋
            </button>
          </p>
        </div>
      </div>
    </div>
    <div
      class="container is-fluid"
      :class="[state === 'imageLoaded' ? '' : 'is-invisible']"
    >
      <div class="box">
        <div class="field has-addons">
          <p class="control">
            <a class="button is-static">
              Extent Coordinate Array (Map Format)
            </a>
          </p>
          <p class="control is-expanded">
            <input class="input" type="text" readonly :value="extentCoords" />
          </p>
          <p class="control">
            <button
              class="button is-secondary"
              v-clipboard="() => extentCoords"
            >
              📋
            </button>
          </p>
        </div>

        <div class="field has-addons">
          <p class="control">
            <a class="button is-static">
              Extent Coordinate Array (Map Format, Rounded)
            </a>
          </p>
          <p class="control is-expanded">
            <input
              class="input"
              type="text"
              readonly
              :value="extentCoordsRounded"
            />
          </p>
          <p class="control">
            <button
              class="button is-secondary"
              v-clipboard="() => extentCoordsRounded"
            >
              📋
            </button>
          </p>
        </div>

        <div class="field has-addons">
          <p class="control">
            <a class="button is-static">
              Extent Coordinate Array (IIIF Format)
            </a>
          </p>
          <p class="control is-expanded">
            <input
              class="input"
              type="text"
              readonly
              :value="extentCoordsIIIF"
            />
          </p>
          <p class="control">
            <button
              class="button is-secondary"
              v-clipboard="() => extentCoordsIIIF"
            >
              📋
            </button>
          </p>
        </div>

        <div class="field has-addons">
          <p class="control">
            <a class="button is-link" :href="croppedImage" target="_blank">
              Cropped Image
            </a>
          </p>
          <p class="control is-expanded">
            <input class="input" type="text" readonly :value="croppedImage" />
          </p>
          <p class="control">
            <button
              class="button is-secondary"
              v-clipboard="() => croppedImage"
            >
              📋
            </button>
          </p>
        </div>
        <div class="notification is-info is-light">
          Shift and drag to select an extent
        </div>
        <div id="ol-map"></div>
      </div>
    </div>
  </section>
</template>

<script lang="ts">
import Vue from "vue";

import Map from "ol/Map.js";
import View from "ol/View.js";
import TileLayer from "ol/layer/Tile.js";
import IIIF from "ol/source/IIIF";
import IIIFInfo from "ol/format/IIIFInfo";
import ExtentInteraction from "ol/interaction/Extent";

import Clipboard from "v-clipboard";
Vue.use(Clipboard);

export default Vue.extend({
  mounted() {
    this.layer = new TileLayer();
    this.map = new Map({
      target: "ol-map",
      layers: [this.layer],
      view: new View({
        center: [0, 0],
        zoom: 2,
      }),
    });
    this.extent = new ExtentInteraction();
    this.map.addInteraction(this.extent);
    this.extent.setActive(false);

    window.addEventListener("keydown", (event) => {
      if (event.keyCode == 16) {
        this.extent.setActive(true);
      }

      window.addEventListener("keyup", (event) => {
        if (event.keyCode == 16) {
          this.extent.setActive(false);
        }

        this.extent.on("extentchanged", (e) => {
          this.$data.extentCoords = this.extent.getExtent();
        });
      });
    });
  },

  data: function() {
    return {
      state: "initial",
      error: false,
      errorMessage: null,
      manifestLoaded: false,
      sourceType: null,
      manifestUrl: null,
      enteredUrl: "",
      commonwealthObjectId: null,
      manifest: "",
      loadedImg: [],
      imgType: "default",
      extentCoords: [0, 0, 0, 0],
      imgResolutionType: "full",
      imgResolutionVariable: 1200,
    };
  },

  computed: {
    lmecObjectUrl: function() {
      return `https://collections.leventhalmap.org/search/${this.commonwealthObjectId}`;
    },
    dcObjectUrl: function() {
      return `https://www.digitalcommonwealth.org/search/${this.commonwealthObjectId}`;
    },
    arkLink: function() {
      if (!this.manifest.metadata) {
        return "";
      } else {
        var idf = this.manifest.metadata.filter(
          (d) => d.label === "Identifier"
        )[0].value;
        if (typeof idf === "string" || idf instanceof String) {
          return idf;
        } else {
          return idf[0];
        }
      }
    },

    extentCoordsRounded: function() {
      return this.extentCoords.map((d) => Math.round(d));
    },

    extentCoordsIIIF: function() {
      let c = [
        this.extentCoords[0],
        this.extentCoords[3] * -1,
        this.extentCoords[2] - this.extentCoords[0],
        this.extentCoords[1] * -1 - this.extentCoords[3] * -1,
      ];
      return c.map((d) => Math.round(d));
    },

    loadedImgIIIF: function() {
      return this.manifest.sequences && this.loadedImg.length > 0
        ? this.manifest.sequences[this.loadedImg[0]].canvases[this.loadedImg[1]]
            .images[this.loadedImg[2]].resource.service["@id"]
        : "";
    },
    resolution: function() {
      if (this.imgResolutionType === "full") {
        return "full";
      } else if (this.imgResolutionType === "Maximum height") {
        return "," + this.imgResolutionVariable;
      } else if (this.imgResolutionType === "Maximum width") {
        return this.imgResolutionVariable + ",";
      } else if (this.imgResolutionType === "Percent") {
        return "pct:" + this.imgResolutionVariable;
      }
    },
    uncroppedImage: function() {
      return (
        this.loadedImgIIIF +
        "/" +
        "full" +
        "/" +
        this.resolution +
        "/0/" +
        this.imgType +
        ".jpg"
      );
    },
    croppedImage: function() {
      return (
        this.loadedImgIIIF +
        "/" +
        Math.round(this.extentCoords[0]) +
        "," +
        Math.round(this.extentCoords[3] * -1) +
        "," +
        Math.round(this.extentCoords[2] - this.extentCoords[0]) +
        "," +
        Math.round(this.extentCoords[1] * -1 - this.extentCoords[3] * -1) +
        "/" +
        this.resolution +
        "/0/" +
        this.imgType +
        ".jpg"
      );
    },
  },

  methods: {
    resolveEnteredUrl: function() {
      this.state = "loading";
      this.error = false;

      const re = /search\/commonwealth:[^//]+/;
      const match = re.exec(this.enteredUrl);
      const re2 = /archive.org\/details\/[^//]+/;
      const match2 = re2.exec(this.enteredUrl);
      const re3 = /2\/commonwealth:[^//]+/;
      const match3 = re3.exec(this.enteredUrl);

      if (match) {
        this.commonwealthObjectId = match[0];
        this.sourceType = "commonwealth";
        this.manifestUrl = `https://www.digitalcommonwealth.org/${match[0]}/manifest.json`;
        this.getManifest();
      }
      else if (match2) {
        this.sourceType = "external";
        this.manifestUrl = `https://iiif.archivelab.org/iiif/${match2[0].split("/")[2]}/manifest.json`;
        this.getManifest();          
      } else if (match3) {
        fetch(`https://collections.leventhalmap.org/search/${match3[0].slice(2)}.json`)
              .then((response) => response.json())
              .then((data) => {
                console.log(data)
                const uri =
                  data["response"]["document"]["is_file_set_of_ssim"][0];
                this.sourceType = "commonwealth";
                this.manifestUrl = `https://www.digitalcommonwealth.org/search/${uri}/manifest.json`;
                this.commonwealthObjectId = uri;
                this.getManifest();
            });            
      } else {
        this.sourceType = "external";
        this.manifestUrl = this.enteredUrl;
      }
    },
   
    getManifest: function() {
      //  Edit this pattern for different mappings from collections ID to Manifest
      fetch(this.manifestUrl)
        .then((r) => r.json())
        .then((d) => {
          console.log(d)
          this.manifest = d;
          this.manifestLoaded = true;
        })
        .catch((e) => {
          console.log(e)
          this.state = "error";
          this.error = true;
          this.errorMessage =
            "There is something wrong with the Manifest for this object. Check the URL to make sure it’s valid.";
        });
    },

    loadImage: function(s, c, i) {
      this.loadedImg = [s, c, i];
      this.state = "imageLoaded";
      this.loadIIIFLayer(this.loadedImgIIIF + "/info.json");
    },

    loadIIIFLayer(endpoint) {
      fetch(endpoint)
        .then((response) => {
          response
            .json()
            .then((imageInfo) => {
              var options = new IIIFInfo(imageInfo).getTileSourceOptions();
              if (options === undefined || options.version === undefined) {
                window.alert(
                  "Data seems to be no valid IIIF image information."
                );
                return;
              }
              options.zDirection = -1;
              var iiifTileSource = new IIIF(options);
              this.layer.setSource(iiifTileSource);
              this.map.setView(
                new View({
                  resolutions: iiifTileSource.getTileGrid().getResolutions(),
                  extent: iiifTileSource.getTileGrid().getExtent(),
                  constrainOnlyCenter: true,
                })
              );
              this.map.getView().fit(iiifTileSource.getTileGrid().getExtent());
              this.map.render();
            })
            .catch(() => {
              window.alert("Could not read image info json.");
            });
        })
        .catch(() => {
          window.alert("Could not read data from URL.");
        });
    },
  },
});
</script>

<style>
@import "ol/ol.css";

#ol-map {
  height: 500px;
  width: 100%;
  margin-top: 20px;
  background-color: #222;
}

#main > .container {
  margin-top: 1rem;
}

li > ul {
  margin-left: 15px;
}
</style>
