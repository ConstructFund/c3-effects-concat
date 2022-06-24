<template>
  <v-app>
    <v-app-bar absolute dense flat>
      <v-layout row justify-center align-center style="margin: 0px 5px">
        <v-toolbar-title class="d-none-xs">
          C3 Effects Concatenator
        </v-toolbar-title>
        <v-spacer></v-spacer>
        <v-btn color="primary" @click="onIssue" outlined>Report an issue</v-btn>
        <v-divider vertical style="margin-right: 15px"></v-divider>
        <v-switch
          style="align-self: center"
          v-model="dark"
          inset
          hide-details
          label="Dark mode"
        ></v-switch>
      </v-layout>
    </v-app-bar>
    <v-main>
      <!-- file upload button -->
      <v-card
        @drop.prevent="onDrop($event)"
        @dragover.prevent="onDrag($event)"
        @dragenter.prevent="onDrag($event)"
        @dragleave.prevent="dragover = false"
        :class="`dragzone` + (dragover ? ` dragover` : ``)"
      >
        <v-layout column wrap>
          <v-layout
            row
            style="
              width: 100%;
              align-items: center;
              justify-content: center;
              margin: 0 !important;
              margin-bottom: 30px;
            "
          >
            <v-card
              style="
                height: 286px;
                flex: 8;
                display: flex;
                flex-direction: column;
              "
              class="c3effectslist"
            >
              <v-card-title
                style="
                  word-break: initial;
                  background-color: rgba(0, 0, 0, 0.1);
                  border-bottom: 1px solid rgba(150, 150, 150, 0.5);
                  margin-bottom: 10px;
                  padding-bottom: 6px;
                  padding-top: 13px;
                "
              >
                Add a vanilla C3 effect
              </v-card-title>

              <v-text-field
                v-model="search"
                label="Search"
                single-line
                dense
                hide-details
                style="margin: 0px 0px 0px 10px; flex: 0 0 26px"
              ></v-text-field>
              <v-list
                style="bottom: 0; overflow-y: auto; flex: 1"
                subheader
                dense
                class="transparent"
              >
                <v-list-item
                  v-for="element in filteredEffects"
                  :key="element.json.id"
                  @click="addC3Effect(element)"
                >
                  <v-list-item-content>
                    <v-list-item-title>{{
                      element.lang.text.effects[element.json.id.toLowerCase()]
                        .name
                    }}</v-list-item-title>
                  </v-list-item-content>
                </v-list-item>
              </v-list>
            </v-card>
            <span
              style="
                flex: 1.2;
                text-align: center;
                display: flex;
                margin: 20px;
                flex-direction: column;
              "
            >
              <v-divider></v-divider>
              OR
              <v-divider></v-divider>
            </span>
            <v-btn
              style="
                flex: 5.7;
                height: 286px;
                border-radius: 12px;
                border-top-left-radius: 0;
                border-bottom-left-radius: 0;
              "
              color="primary"
              outlined
              :loading="isSelecting"
              @click="handleFileImport"
            >
              <span style="white-space: normal; width: 100%">
                Add 3rd party effects
              </span>
            </v-btn>
          </v-layout>

          <!-- Create a File Input that will be hidden but triggered with JavaScript -->
          <input
            ref="uploader"
            class="d-none"
            type="file"
            multiple
            @change="onFileChanged"
          />
          <draggable v-model="addons" class="list-group" ghost-class="ghost">
            <transition-group>
              <v-layout
                row
                style="flex-wrap: nowrap; margin-left: 0; margin-right: 0"
                v-for="element in addons"
                :key="`${element.data.id}${element.number}`"
                class="list-group-item"
              >
                <v-icon style="margin-right: 10px">
                  mdi-drag-horizontal-variant
                </v-icon>
                <v-tooltip v-if="element.nbTextureReads > 5" color="red" bottom>
                  <template v-slot:activator="{ on, attrs }">
                    <v-icon
                      color="red"
                      v-bind="attrs"
                      v-on="on"
                      style="margin-right: 10px"
                    >
                      mdi-alert
                    </v-icon>
                  </template>
                  This effect might cause performance issues when concatenated
                </v-tooltip>

                <div style="width: calc(100% - 48px)">{{ element.name }}</div>

                <v-icon
                  style="align-self: flex-end"
                  @click="remove(element)"
                  big
                  >mdi-close</v-icon
                >
              </v-layout>
            </transition-group>
          </draggable>

          <!-- dropdown for category, a checkbox to remove duplicate authors, and a text field for custom name -->

          <details>
            <summary>Addon properties</summary>

            <div class="content">
              <v-layout
                row
                wrap
                style="
                  margin: 0px 10px;
                  align-items: center;
                  justify-content: flex-start;
                "
              >
                <v-switch
                  label="Force category"
                  v-model="forceCategory"
                  hide-details
                  style="margin-top: 8px !important; margin-bottom: 8px"
                ></v-switch>
                <v-select
                  v-model="customCategory"
                  :items="categories"
                  label="Category"
                  outlined
                  :disabled="!forceCategory"
                  dense
                  hide-details
                  style="margin: 0px 15px 0px 10px"
                ></v-select>
              </v-layout>
              <v-layout
                row
                wrap
                style="
                  margin: 0px 10px;
                  align-items: center;
                  justify-content: flex-start;
                "
              >
                <v-switch
                  v-model="removeDuplicates"
                  label="Remove duplicate authors"
                  hide-details
                  style="margin-top: 8px !important; margin-bottom: 8px"
                ></v-switch>
              </v-layout>
              <v-layout
                row
                wrap
                style="
                  margin: 0px 10px;
                  align-items: center;
                  justify-content: flex-start;
                "
              >
                <v-switch
                  label="Force name"
                  v-model="forceName"
                  hide-details
                  style="margin-top: 8px !important; margin-bottom: 8px"
                ></v-switch>
                <v-text-field
                  v-model="customName"
                  :disabled="!forceName"
                  label="Name"
                  dense
                  hide-details
                  style="margin: 0px 10px"
                ></v-text-field>
              </v-layout>
              <v-layout
                row
                wrap
                style="
                  margin: 0px 10px;
                  align-items: center;
                  justify-content: flex-start;
                "
              >
                <v-switch
                  label="Force Description"
                  v-model="forceDescription"
                  hide-details
                  style="margin-top: 8px !important; margin-bottom: 8px"
                ></v-switch>
                <v-text-field
                  v-model="customDescription"
                  :disabled="!forceDescription"
                  label="Description"
                  dense
                  hide-details
                  style="margin: 0px 10px"
                ></v-text-field>
              </v-layout>
              <details>
                <summary>Advanced</summary>

                <div class="content">
                  <v-layout
                    row
                    wrap
                    style="
                      margin: 0px 10px;
                      align-items: center;
                      justify-content: flex-start;
                    "
                  >
                    <v-switch
                      label="Force ID"
                      v-model="forceId"
                      hide-details
                      style="margin-top: 8px !important; margin-bottom: 8px"
                    ></v-switch>
                    <v-text-field
                      v-model="customId"
                      :disabled="!forceId"
                      label="ID"
                      dense
                      hide-details
                      style="margin: 0px 10px"
                    ></v-text-field>
                  </v-layout>
                </div>
              </details>
            </div>
          </details>
          <!-- generate button -->
          <v-btn
            color="primary"
            @click="onGenerate"
            :disabled="addons.length < 2"
            style="margin-top: 20px"
          >
            Concatenate
          </v-btn>

          <!-- searchable list with all C3Effects -->
        </v-layout>
      </v-card>
      <v-dialog v-model="dialog" persistent>
        <v-card>
          <v-card-title class="text-h5"> Something went wrong </v-card-title>
          <v-card-text>
            An error occurred while trying to concatenate the effects.
            <v-expansion-panels v-model="panel" focusable multiple>
              <v-expansion-panel>
                <v-expansion-panel-header>Error</v-expansion-panel-header>
                <v-expansion-panel-content style="padding-top: 20px">
                  {{ error }}
                </v-expansion-panel-content>
              </v-expansion-panel>
            </v-expansion-panels>
          </v-card-text>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="error" text @click="copyError">
              Copy error data
            </v-btn>
            <v-btn color="primary" text @click="onIssue">
              Report an issue
            </v-btn>
            <v-btn color="grey" text @click="dialog = false"> Close </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
      <v-card v-if="animation" class="concatenatedAddon">
        <v-card-title style="justify-content: center" class="text-h5">
          Effects concatenated!
        </v-card-title>
        <v-card-text style="margin-bottom: 20px; width: 100%" class="text-h5">
          {{ animationName }}
        </v-card-text>
      </v-card>
    </v-main>
  </v-app>
</template>

<script>
import JSZip from "jszip";
import draggable from "vuedraggable";
import { saveAs } from "file-saver";
import allEffects from "./assets/allEffects.json";
import allEffectsLang from "./assets/precompiled-en-US.json";
import Accordion from "./Accordion.js";

var TokenString = require("glsl-tokenizer/string");
var ParseTokens = require("@sicmutils/glsl-parser/direct");
let deparser = require("@andrewray/glsl-deparser");

function getDarkPreference() {
  let dark = false;
  if (localStorage.getItem("dark") === "true") {
    dark = true;
  } else if (localStorage.getItem("dark") === "false") {
    dark = false;
  } else if (window.matchMedia("(prefers-color-scheme: dark)").matches) {
    dark = true;
  }
  return dark;
}

export default {
  name: "App",

  components: { draggable },

  data: () => ({
    panel: [0],
    animation: false,
    animationName: "mes couilles",
    search: "",
    error: "",
    C3Effects: [],
    dragover: false,
    dialog: false,
    loadingC3Effects: true,
    dark: getDarkPreference(),
    isSelecting: false,
    addons: [],
    files: [],
    processedEffects: {},
    categories: ["blend", "color", "distortion", "normal-mapping", "other"],
    forceName: false,
    forceCategory: false,
    customName: "",
    customCategory: "other",
    removeDuplicates: true,
    customId: "",
    forceId: false,
    customDescription: "",
    forceDescription: false,
    commonUniforms: [
      "varying mediump vec2 vTex;",
      "uniform lowp sampler2D samplerFront;",
      "uniform mediump vec2 srcStart;",
      "uniform mediump vec2 srcEnd;",
      "uniform mediump vec2 srcOriginStart;",
      "uniform mediump vec2 srcOriginEnd;",
      "uniform mediump vec2 layoutStart;",
      "uniform mediump vec2 layoutEnd;",
      "uniform lowp sampler2D samplerBack;",
      "uniform lowp sampler2D samplerDepth;",
      "uniform mediump vec2 destStart;",
      "uniform mediump vec2 destEnd;",
      "uniform highp float seconds;",
      "uniform mediump vec2 pixelSize;",
      "uniform mediump float pixelWidth;",
      "uniform mediump float pixelHeight;",
      "uniform mediump float layerScale;",
      "uniform mediump float layerAngle;",
      "uniform mediump float devicePixelRatio;",
    ],
    commonUniformsToRemove: [
      "varying mediump vec2 vTex;",
      "uniform lowp sampler2D samplerFront;",
      "uniform mediump vec2 srcStart;",
      "uniform mediump vec2 srcEnd;",
      "uniform mediump vec2 srcOriginStart;",
      "uniform mediump vec2 srcOriginEnd;",
      "uniform mediump vec2 layoutStart;",
      "uniform mediump vec2 layoutEnd;",
      "uniform lowp sampler2D samplerBack;",
      "uniform lowp sampler2D samplerDepth;",
      "uniform mediump vec2 destStart;",
      "uniform mediump vec2 destEnd;",
      "uniform highp float seconds;",
      "uniform mediump float seconds;",
      "uniform mediump vec2 pixelSize;",
      "uniform mediump float pixelWidth;",
      "uniform mediump float pixelHeight;",
      "uniform lowp float pixelWidth;",
      "uniform lowp float pixelHeight;",
      "uniform mediump float layerScale;",
      "uniform mediump float layerAngle;",
      "uniform mediump float devicePixelRatio;",
    ],
    commonUniformsToRemove2: [
      "vTex",
      "samplerFront",
      "srcStart",
      "srcEnd",
      "srcOriginStart",
      "srcOriginEnd",
      "layoutStart",
      "layoutEnd",
      "samplerBack",
      "samplerDepth",
      "destStart",
      "destEnd",
      "seconds",
      "pixelSize",
      "layerScale",
      "layerAngle",
      "devicePixelRatio",
    ],
  }),
  mounted() {
    this.$vuetify.theme.dark = this.dark;

    // fetch with no cors
    let effects = allEffects;
    let lang = allEffectsLang;
    effects.all.forEach((effect) => {
      let id = effect.json.id;
      effect.lang = {
        languageTag: lang.languageTag,
        fileDescription: lang.fileDescription,
        text: {
          effects: {
            [id.toLowerCase()]: lang.text.effects[id.toLowerCase()],
          },
        },
      };
    });
    this.C3Effects = effects.all;
    // this.C3Effects.forEach((effect) => {
    //   this.addC3Effect(effect);
    // });
    this.loadingC3Effects = false;

    document.querySelectorAll("details").forEach((el) => {
      new Accordion(el);
    });
  },
  watch: {
    dark() {
      this.$vuetify.theme.dark = this.dark;
      localStorage.setItem("dark", this.dark);
    },
  },
  computed: {
    filteredEffects() {
      let search = this.search.toLowerCase();
      return this.C3Effects.filter((effect) => {
        return (
          effect.json.id.toLowerCase().includes(search) ||
          effect.lang.text.effects[effect.json.id.toLowerCase()].name
            .toLowerCase()
            .includes(search)
        );
      });
    },
  },
  methods: {
    addC3Effect(effect) {
      let addon = {
        name: effect.lang.text.effects[effect.json.id.toLowerCase()].name,
        data: JSON.parse(JSON.stringify(effect.json)),
        id: effect.json.id,
        fx: effect.glsl,
        lang: JSON.parse(JSON.stringify(effect.lang)),
        langId: effect.json.id.toLowerCase(),
        number: 0,
      };

      // find number of matches in fx of texture2D
      let texture2DRegex = /texture2D *\( *samplerFront *, */g;
      let matches = addon.fx.match(texture2DRegex);
      if (matches) {
        addon.nbTextureReads = matches.length;
      } else {
        addon.nbTextureReads = 0;
      }

      let readInForLoopRegex =
        /for *\([^{]*{[^{}]*texture2D\( *samplerFront *,[^}]*}/g;
      matches = addon.fx.match(readInForLoopRegex);
      if (matches) {
        addon.nbTextureReads = 999;
      }

      let index = this.addons.findIndex((element) => element.id === addon.id);
      if (index !== -1) {
        addon.number = this.addons.filter(
          (element) => element.id === addon.id
        ).length;
        this.addons.push(addon);
        this.processEffect(addon);
      } else {
        this.addons.push(addon);
        this.processEffect(addon);
      }
    },
    processAST(addon) {
      let id = addon.id;
      let langId = addon.langId;
      let processedEffects = this.processedEffects;
      processedEffects[id] = processedEffects[id] || [];
      let number = processedEffects[id].length;
      let dataContent = addon.data;
      let langContent = addon.lang;
      let fxContent = addon.fx;

      console.log("Processing AST for " + id);
      //HACK to fix bug with macros being used instead of precision keywords
      let precisionMacroRegex =
        /#ifdef GL_FRAGMENT_PRECISION_HIGH[^#]*#define +(\w+) +\w+(?:[^#]*#)*endif/g;

      // remove precision macros
      let match;
      while ((match = precisionMacroRegex.exec(fxContent)) !== null) {
        console.log("matched ", match);
        let name = match[1];
        fxContent = fxContent.replace(match[0], "");
        fxContent = fxContent.replaceAll(name, "mediump");
      }

      console.log(fxContent);
      var tokens = TokenString(fxContent);
      var ast = ParseTokens(tokens);
      console.log(ast);
      window.ast = ast;

      Object.keys(ast.scope)
        .filter((key) => this.commonUniformsToRemove2.includes(key))
        .forEach((key) => {
          console.log("removing", key);
          // find children of ast that is that variable
          var child = ast.scope[key];
          while (child.parent !== ast) {
            child = child.parent;
          }
          // remove the variable
          ast.children = ast.children.filter((x) => x !== child);
          // remove the variable from the scope
          if (key !== "vTex") {
            delete ast.scope[key];
          }
        });

      // rename all remaining variables to avoid conflicts
      Object.keys(ast.scope).forEach((key) => {
        if (key === "main" || key === "vTex") return;
        var child = ast.scope[key];
        child.data = id + number.toString() + "_" + child.data;
        let uniformParam = dataContent.parameters.find(
          (x) => x.uniform === key
        );

        if (uniformParam && langId) {
          let lang =
            langContent.text.effects[langId].parameters[uniformParam.id];
          // remove that lang from the lang file
          delete langContent.text.effects[langId].parameters[uniformParam.id];
          if (uniformParam) {
            uniformParam.id = `${id}${number.toString()}_${uniformParam.id}`;
            uniformParam.uniform = child.data;
          }
          // add it back to lang with the correct id
          langContent.text.effects[langId].parameters[uniformParam.id] = lang;
        } else {
          console.log(
            `${id}${number.toString()}_${key} not found in data.json`
          );
        }
      });

      let preprocessors = [];

      function findPreprocessorsRecursively(node) {
        if (
          node.type === "preprocessor" &&
          node.token.data.startsWith("#define")
        ) {
          preprocessors.push(node);
        }
        if (node.children) {
          node.children.forEach((child) => {
            findPreprocessorsRecursively(child);
          });
        }
      }
      findPreprocessorsRecursively(ast);
      console.log(preprocessors);

      function renamePreprocessorRecursively(node, name, newName) {
        if (node.type === "ident" && node.data === name) {
          node.data = newName;
        }
        if (node.children) {
          node.children.forEach((child) => {
            renamePreprocessorRecursively(child, name, newName);
          });
        }
      }

      preprocessors.forEach((preprocessor) => {
        let data = preprocessor.token.data;
        let defineRefex = /#define +([^( ]+)(\([^)]*\))? +(.*)$/;
        // get the name of the define
        let match = defineRefex.exec(data);
        let name = match[1];
        let args = match[2];
        let value = match[3];
        let newName = id + number.toString() + "_" + name;
        preprocessor.token.data = `#define ${newName}${
          args === undefined ? "" : args
        } ${value}`;
        // find all "ident" nodes that are the same name, and rename them to the new name
        renamePreprocessorRecursively(ast, name, newName);
      });

      processedEffects[id].push(ast);
    },
    onDrag(e) {
      // check that a file is being dragged
      var dt = e.dataTransfer;
      if (
        dt.types &&
        (dt.types.indexOf
          ? dt.types.indexOf("Files") != -1
          : dt.types.contains("Files"))
      ) {
        this.dragover = true;
      }
    },
    onDrop(e) {
      this.dragover = false;
      this.files = [...e.dataTransfer.files];
      this.onFileChange(this.files);
    },
    copyError() {
      navigator.clipboard.writeText(this.error);
    },
    handleFileImport() {
      this.isSelecting = true;

      // After obtaining the focus when closing the FilePicker, return the button state to normal
      window.addEventListener(
        "focus",
        () => {
          this.isSelecting = false;
        },
        { once: true }
      );

      // Trigger click on the FileInput
      this.$refs.uploader.click();
    },
    onFileChanged(e) {
      this.files = [...e.target.files];
      this.onFileChange(this.files);
      // clear the file input
      e.target.value = "";
    },
    remove(element) {
      this.addons = this.addons.filter((addon) => addon !== element);
    },
    onIssue() {
      window.open(
        "https://github.com/ConstructFund/c3-effects-concat/issues/new"
      );
    },
    onFileChange(files) {
      this.error = "";
      Promise.all(
        files.map((file) => {
          let zip = new JSZip();
          let addon = {
            name: file.name,
            data: null,
            id: "",
            fx: null,
            lang: null,
            number: 0,
          };
          let found = 0;
          return zip
            .loadAsync(file.arrayBuffer())
            .then(async (zip) => {
              // get all files
              let promises = [];
              zip.forEach((relativePath) => {
                // get all files
                // get addon.json, effect.fx and lang/en-US.json
                if (relativePath.includes("MACOSX")) return;
                if (relativePath.endsWith("addon.json")) {
                  promises.push(
                    zip
                      .file(relativePath)
                      .async("string")
                      .then((content) => {
                        // get addon.json and remove 1st char if it is \ufeff
                        let json = content.replaceAll(/\ufeff/g, "");
                        addon.data = JSON.parse(json);
                        addon.id = addon.data.id;
                        addon.name = addon.data.name;
                      })
                  );
                  found++;
                } else if (relativePath.endsWith("effect.fx")) {
                  promises.push(
                    zip
                      .file(relativePath)
                      .async("string")
                      .then((content) => {
                        addon.fx = content;
                      })
                  );
                  found++;
                } else if (relativePath.endsWith("lang/en-US.json")) {
                  promises.push(
                    zip
                      .file(relativePath)
                      .async("string")
                      .then(function (content) {
                        // get lang/en-US.json
                        let json = content.replaceAll(/\ufeff/g, "");
                        addon.lang = JSON.parse(json);
                      })
                  );
                  found++;
                }
              });

              if (found === 3) {
                await Promise.all(promises);
                // find number of matches in fx of texture2D
                let texture2DRegex = /texture2D *\( *samplerFront *, */g;
                let matches = addon.fx.match(texture2DRegex);
                if (matches) {
                  addon.nbTextureReads = matches.length;
                } else {
                  addon.nbTextureReads = 0;
                }

                let readInForLoopRegex =
                  /for *\([^{]*{[^{}]*texture2D\( *samplerFront *,[^}]*}/g;
                matches = addon.fx.match(readInForLoopRegex);
                if (matches) {
                  addon.nbTextureReads = 999;
                }

                let langId = null;
                let id = addon.id;
                if (addon.lang.text.effects[id.toLowerCase()]) {
                  langId = id.toLowerCase();
                } else if (addon.lang.text.effects[id]) {
                  langId = id;
                } else {
                  let keys = Object.keys(addon.lang.text.effects);
                  if (keys.length === 1) {
                    langId = keys[0];
                  }
                }
                addon.langId = langId;
                if (!langId) {
                  throw "Could not find language data for addon";
                }

                // if addons already contains addon, replace it
                let index = this.addons.findIndex(
                  (element) => element.id === addon.id
                );
                if (index !== -1) {
                  addon.number = this.addons.filter(
                    (element) => element.id === addon.id
                  ).length;
                  this.addons.push(addon);
                  this.processEffect(addon);
                } else {
                  this.addons.push(addon);
                  this.processEffect(addon);
                }
              } else {
                this.error += "Invalid effect addon file: " + file.name + "\n";
                this.dialog = true;
              }
            })
            .catch((err) => {
              console.log(err);
            });
        })
      ).then(() => {
        this.files = [];
      });
    },
    onGenerate() {
      let start = performance.now();
      try {
        let zip = new JSZip();
        // create effect.fx, addon.json and lang/en-US.json
        zip.file(
          "effect.fx",
          this.concatEffects(this.addons.map((addon) => addon.id))
        );
        let addonJson = this.concatData(this.addons.map((addon) => addon.id));
        zip.file("addon.json", JSON.stringify(addonJson, null, 2));
        let langJson = this.concatLang(this.addons.map((addon) => addon.id));
        zip.file("lang/en-US.json", JSON.stringify(langJson, null, 2));
        // download zip
        zip.generateAsync({ type: "blob" }).then((blob) => {
          saveAs(blob, addonJson.id + ".c3Addon");
          this.createCoolAddonAnimationAndClearList(
            langJson.text.effects[addonJson.id.toLowerCase()].name
          );
        });
      } catch (err) {
        console.error(err);
        this.error = err;
        this.dialog = true;
      }
      let end = performance.now();
      console.log("Time to generate: " + (end - start) + "ms");
    },
    createCoolAddonAnimationAndClearList(name) {
      this.animation = true;
      this.animationName = name;
      setTimeout(() => {
        this.animation = false;
      }, 1600);
      setTimeout(() => {
        this.addons = [];
        this.processedEffects = {};
      }, 200);
    },
    processEffect(addon) {
      this.processAST(addon);
    },

    concatEffects(list) {
      let resultArr = [];
      let commonUniforms = this.commonUniforms;
      // append common uniforms
      resultArr = resultArr.concat(commonUniforms);

      resultArr.push("");
      // append uniforms for all effects
      let prevEffectFunction = "";
      let texture2DRegex = /texture2D *\( *samplerFront *, */g;
      let numberPerId = {};
      list.forEach((id, index) => {
        // check if id exists in numberPerId
        if (numberPerId[id] === undefined) {
          numberPerId[id] = 0;
        } else {
          numberPerId[id]++;
        }
        let number = numberPerId[id];
        let effect = this.processedEffects[id][number];
        let isNotLastMain = index < list.length - 1;
        if (isNotLastMain) {
          effect.scope.vTex.data = id + number.toString() + "_vTex";
        }

        let fxSrc = deparser(effect);
        if (prevEffectFunction !== "") {
          // replace "texture2D(samplerFront" with a call to the previous function
          fxSrc = fxSrc.replaceAll(texture2DRegex, `${prevEffectFunction}(`);
        }
        if (isNotLastMain) {
          resultArr.push(`mediump vec2 ${id}${number.toString()}_vTex;`);
          // replace main function name with a new name
          prevEffectFunction = `${id}${number.toString()}_main`;
          fxSrc = fxSrc
            .replace(
              /void main\([^)]*\)\s*\{/g,
              `mediump vec4 ${prevEffectFunction}(mediump vec2 vTex) {
  ${id}${number.toString()}_vTex = vTex;`
            )
            .replaceAll(/gl_FragColor *= */g, `return `);
        }
        resultArr.push(fxSrc);
      });
      return resultArr.join("\n");
    },

    concatData(list) {
      let resultData = {
        "is-c3-addon": true,
        type: "effect",
        version: "1.0.0.0",
        website: "https://www.construct.net",
        documentation: "https://www.construct.net",
        "file-list": ["lang/en-US.json", "addon.json", "effect.fx"],
      };

      let numberPerId = {};

      let addons = list.map((id) => {
        if (numberPerId[id] === undefined) {
          numberPerId[id] = 0;
        } else {
          numberPerId[id]++;
        }
        return this.addons.find((x) => {
          return x.id === id && x.number === numberPerId[id];
        });
      });
      if (this.forceName) {
        resultData.name = this.customName;
      } else {
        resultData.name = addons.map((addon) => addon.name).join(" + ");
      }
      if (this.forceId) {
        resultData.id = this.customId;
      } else {
        resultData.id = addons.map((addon) => addon.id).join("_");
      }
      if (this.removeDuplicates) {
        let set = new Set([...addons.map((addon) => addon.data.author)]);
        resultData.author = Array.from(set).join(", ");
      } else {
        resultData.author = addons.map((addon) => addon.data.author).join(", ");
      }
      if (this.forceDescription) {
        resultData.description = this.customDescription;
      } else {
        resultData.description =
          "Concat effect of " + addons.map((addon) => addon.name).join(", ");
      }

      if (this.forceCategory) {
        resultData.category = this.customCategory;
      } else {
        // if all categories are the same, use that one, else "other"
        resultData.category = addons
          .map((addon) => addon.data.category)
          .reduce((acc, curr) => {
            if (acc === curr) {
              return acc;
            } else {
              return "other";
            }
          }, addons[0].data.category);
      }
      // or all the results
      resultData["blends-background"] = addons
        .map((addon) => addon.data["blends-background"])
        .reduce((acc, curr) => {
          return acc || curr;
        }, addons[0].data["blends-background"]);
      resultData["cross-sampling"] = addons
        .map((addon) => addon.data["cross-sampling"])
        .reduce((acc, curr) => {
          return acc || curr;
        }, addons[0].data["cross-sampling"]);
      resultData["preserves-opaqueness"] = addons
        .map((addon) => addon.data["preserves-opaqueness"])
        .reduce((acc, curr) => {
          return acc || curr;
        }, addons[0].data["preserves-opaqueness"]);
      resultData.animated = addons
        .map((addon) => addon.animated)
        .reduce((acc, curr) => {
          return acc || curr;
        }, addons[0].animated);
      resultData["extend-box"] = addons
        .map((addon) => addon.data["extend-box"])
        .reduce((acc, curr) => {
          return {
            horizontal: Math.max(acc.horizontal, curr.horizontal),
            vertical: Math.max(acc.vertical, curr.vertical),
          };
        }, addons[0].data["extend-box"]);
      resultData.parameters = addons
        .map((addon) => addon.data.parameters)
        .reduce((acc, curr) => {
          return acc.concat(curr);
        }, []);
      return resultData;
    },

    concatLang(list) {
      let resultData = {
        languageTag: "en-US",
        text: {
          effects: {},
        },
      };

      let numberPerId = {};

      let addons = list.map((id) => {
        if (numberPerId[id] === undefined) {
          numberPerId[id] = 0;
        } else {
          numberPerId[id]++;
        }
        return this.addons.find((x) => {
          return x.id === id && x.number === numberPerId[id];
        });
      });

      let id = addons.map((addon) => addon.id).join("_");
      if (this.forceId) {
        id = this.customId;
      }
      resultData.text.effects[id.toLowerCase()] = {
        name: this.forceName
          ? this.customName
          : addons
              .map((addon) => addon.lang.text.effects[addon.langId].name)
              .join(" + "),
        description: this.forceDescription
          ? this.customDescription
          : "Concat effect of " + addons.map((addon) => addon.name).join(", "),
      };
      let params = {};
      addons.forEach((addon) => {
        params = {
          ...params,
          ...addon.lang.text.effects[addon.langId].parameters,
        };
      });
      resultData.text.effects[id.toLowerCase()].parameters = params;
      return resultData;
    },
  },
};
</script>

<style>
html,
body {
  height: 100%;
  width: 100%;
  margin: 0;
  padding: 0;
  overflow: auto !important;
}

html {
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  -khtml-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}

.list-group {
  min-height: 20px;
}
.list-group {
  display: -ms-flexbox;
  display: -webkit-box;
  display: flex;
  -ms-flex-direction: column;
  -webkit-box-orient: vertical;
  -webkit-box-direction: normal;
  flex-direction: column;
  padding-left: 0;
}
.list-group-item {
  cursor: move;
}
.list-group-item i {
  cursor: pointer;
}

.ghost {
  opacity: 0.5;
  background: rgba(150, 150, 150, 0.5);
}
.list-group-item {
  position: relative;
  padding: 0.75rem 1.25rem;
  margin-bottom: -1px;
  border: 1px solid rgba(150, 150, 150, 0.5);
}

.list-group-item:first-child {
  border-top-left-radius: 0.25rem;
  border-top-right-radius: 0.25rem;
}

.list-group-item:last-child {
  margin-bottom: 0;
  border-bottom-right-radius: 0.25rem;
  border-bottom-left-radius: 0.25rem;
}

.dragzone {
  padding: 10px;
  margin: 10px;
  margin-top: 58px;
  height: calc(100% - 68px);
  width: calc(100% - 20px);
  border: 2px dashed transparent !important;
  border-radius: 20px !important;
  background: transparent;
  transition: border 1s ease-in-out;
}
.dragover {
  padding: 10px;
  margin: 10px;
  margin-top: 58px;
  height: calc(100% - 68px);
  width: calc(100% - 20px);
  /* radius */
  border-radius: 20px !important;
  /* dashed border */
  border: 2px dashed #ccc !important;
}
.c3effectslist {
  /* radius */
  border-radius: 12px !important;
  /* dashed border */
  border: 1px solid rgba(150, 150, 150, 0.5) !important;
  border-top-right-radius: 0 !important;
  border-bottom-right-radius: 0 !important;
}
/* width */
::-webkit-scrollbar {
  width: 8px;
}

.concatenatedAddon {
  width: 80%;
  height: 30%;
  max-width: 500px;
  max-height: 500px;
  /* center on screen */
  position: absolute !important;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  border: 1px solid rgba(150, 150, 150, 0.5) !important;
  border-radius: 0.25rem !important;
  /* play animation */
  animation: scaleDownAndFadeIn 1.7s ease;
  /* align text vertically */
  text-align: center;
  vertical-align: middle;
  place-content: center;
  display: flex !important;
  flex-direction: column;
}

@keyframes scaleDownAndFadeIn {
  0% {
    transform: scale(1.3) translate(-40%, -40%);
    opacity: 0;
  }
  17% {
    transform: scale(0.9) translate(-55%, -55%);
    opacity: 1;
  }
  25% {
    transform: scale(1) translate(-50%, -50%);
    opacity: 1;
  }
  60% {
    transform: scale(1) translate(-50%, -50%);
    opacity: 1;
  }
  90% {
    opacity: 0;
  }
  100% {
    transform: translate(-50%, -50%) translateY(100px) scale(1);
    opacity: 0;
  }
}

.v-btn__content {
  flex: 1 0 !important;
}

details {
  /* border and a paddingLeft */
  margin-top: 20px;
  border: 1px solid rgba(150, 150, 150, 0.5) !important;
  border-radius: 0.25rem !important;
}
.content {
  padding-left: 0.5rem;
  padding-right: 0.5rem;
  padding-top: 0.5rem;
  padding-bottom: 0.5rem;
}
summary {
  padding: 1rem;
  padding-left: 2.2rem;
  display: block;
  cursor: pointer;
  position: relative;
  background-color: rgba(0, 0, 0, 0.1);
}
details summary::-webkit-details-marker {
  color: transparent;
}
details summary::marker {
  color: transparent;
}
details.detailsOpen > summary:before {
  transform: rotate(90deg);
}
details[open] > summary {
  border-bottom: 1px solid rgba(150, 150, 150, 0.5) !important;
}
summary:before {
  content: "";
  border-width: 0.4rem;
  border-style: solid;
  border-color: transparent transparent transparent #fff;
  position: absolute;
  top: 1.3rem;
  left: 1rem;
  transform: rotate(0);
  transform-origin: 0.2rem 50%;
  transition: 0.25s transform ease;
}

@media screen and (max-width: 600px) {
  .d-none-xs {
    display: none !important;
  }
}
</style>
<style lang="sass">
@import '~vuetify/src/styles/main.sass'
+theme(v-application) using ($material)
  summary:before
    border-color: transparent transparent transparent map-deep-get($material, 'text', 'primary')

  ::-webkit-scrollbar-thumb
    background-color: map-deep-get($material, 'text', 'primary')
  ::-webkit-scrollbar-thumb:hover
    background-color: map-deep-get($material, 'text', 'secondary')

  ::-webkit-scrollbar-track
    background-color: rgba(map-deep-get($material, 'text', 'disabled'), 0.12)
</style>
