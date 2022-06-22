<template>
  <v-app>
    <v-app-bar absolute dense flat>
      <v-toolbar-title>C3 Effects Concatenator</v-toolbar-title>
      <v-spacer></v-spacer>
      <v-btn color="primary" @click="onIssue" plain>Report an issue</v-btn>
    </v-app-bar>
    <v-main>
      <!-- file upload button -->
      <v-layout column wrap style="margin: 10em">
        <v-file-input
          multiple
          chips
          label="Add effect addons"
          @change="onFileChange"
          ref="fileInput"
        ></v-file-input>
        <draggable v-model="addons" class="list-group" ghost-class="ghost">
          <transition-group>
            <v-layout
              row
              v-for="element in addons"
              :key="element.id"
              class="list-group-item"
            >
              <v-icon style="margin-right: 10px"
                >mdi-drag-horizontal-variant</v-icon
              >
              {{ element.name }}
              <v-spacer></v-spacer>
              <v-icon @click="remove(element.id)" big>mdi-close</v-icon>
            </v-layout>
          </transition-group>
        </draggable>
        <!-- generate button -->
        <v-btn color="primary" @click="onGenerate" :disabled="!addons.length">
          Concatenate
        </v-btn>
      </v-layout>
    </v-main>
  </v-app>
</template>

<script>
import JSZip from "jszip";
import draggable from "vuedraggable";
import { saveAs } from "file-saver";

export default {
  name: "App",

  components: { draggable },

  data: () => ({
    addons: [],
    processedEffects: {},
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
      "uniform mediump vec2 pixelSize;",
      "uniform mediump float pixelWidth;",
      "uniform mediump float pixelHeight;",
      "uniform lowp float pixelWidth;",
      "uniform lowp float pixelHeight;",
      "uniform mediump float layerScale;",
      "uniform mediump float layerAngle;",
      "uniform mediump float devicePixelRatio;",
    ],
  }),
  methods: {
    remove(id) {
      this.addons = this.addons.filter((addon) => addon.id !== id);
    },
    onIssue() {
      window.open(
        "https://github.com/ConstructFund/c3-effects-concat/issues/new"
      );
    },
    onFileChange(files) {
      Promise.all(
        files.map((file) => {
          let zip = new JSZip();
          let addon = {
            name: file.name,
            data: null,
            id: "",
            fx: null,
            lang: null,
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
                // if addons already contains addon, replace it
                let index = this.addons.findIndex(
                  (element) => element.id === addon.id
                );
                if (index !== -1) {
                  this.addons[index] = addon;
                } else {
                  this.addons.push(addon);
                }
              }
            })
            .catch((err) => {
              console.log(err);
            });
          //clear file input
          // console.log(this.$refs.fileInput);
          // this.$refs.fileInput.value();
        })
      ).then(() => {
        //clear file input
        // console.log(this.$refs.fileInput);
        // this.$refs.fileInput.value();
        this.addons.forEach((addon) => {
          this.processEffect(addon);
        });
      });
    },
    onGenerate() {
      let zip = new JSZip();
      // create effect.fx, addon.json and lang/en-US.json
      zip.file(
        "effect.fx",
        this.concatEffects(this.addons.map((addon) => addon.id))
      );
      let addonJson = this.concatData(this.addons.map((addon) => addon.id));
      zip.file("addon.json", JSON.stringify(addonJson, null, 2));
      zip.file(
        "lang/en-US.json",
        JSON.stringify(
          this.concatLang(this.addons.map((addon) => addon.id)),
          null,
          2
        )
      );
      // download zip
      zip.generateAsync({ type: "blob" }).then((blob) => {
        saveAs(blob, addonJson.id + ".c3Addon");
      });
    },
    processEffect(addon) {
      let id = addon.id;
      let dataContent = addon.data;
      let langContent = addon.lang;
      let fxContent = addon.fx;
      let processedEffects = this.processedEffects;
      let commonUniforms = this.commonUniformsToRemove;

      if (id in processedEffects) return;

      processedEffects[id] = {};
      let fxArr = fxContent.split("\n");
      // trim lines and remove comments
      fxArr = fxArr.map((line) => {
        return line
          .trim()
          .replace(/\/\/.*/, "")
          .trim();
      });
      // remove empty lines
      fxArr = fxArr.filter((line) => {
        return line.length > 0;
      });

      // get all lines with precision
      let precisionLines = fxArr.filter((line) => {
        return line.startsWith("precision");
      });
      // replace all precision for unspecified variables
      precisionLines.forEach((line) => {
        let lineA = line.split(" ");
        let precision = lineA[1];
        let type = lineA[2].replace(/;/, "");
        let regex = new RegExp(`(?<!highp |lowp |mediump )${type}`);
        fxArr = fxArr.map((line) => {
          return line.replace(regex, `${precision} ${type}`);
        });
      });
      // remove precision lines
      fxArr = fxArr.filter((line) => {
        return !line.startsWith("precision");
      });

      // remove common uniforms
      fxArr = fxArr.filter((line) => {
        return !commonUniforms.includes(line);
      });

      processedEffects[id].uniforms = fxArr.filter((line) => {
        return line.startsWith("uniform");
      });
      // remove uniforms
      fxArr = fxArr.filter((line) => {
        return !line.startsWith("uniform");
      });

      processedEffects[id].uniforms = processedEffects[id].uniforms.map(
        (line) => {
          // uniforms look like this "uniform lowp float width;"
          let lineA = line.split(" ");
          let name = lineA[lineA.length - 1].replace(";", "");
          let newName = `${id}_${name}`;
          line = line.replace(name, newName);
          // replace all uses of that uniform with the new name in the main code
          fxArr = fxArr.map((line) => {
            return line.replaceAll(
              new RegExp(`(?<!\\w)${name}(?!\\w)`, "g"),
              newName
            );
          });

          let uniformParam = dataContent.parameters.find(
            (x) => x.uniform === name
          );
          if (uniformParam) {
            let lang =
              langContent.text.effects[id.toLowerCase()].parameters[
                uniformParam.id
              ];
            // remove that lang from the lang file
            delete langContent.text.effects[id.toLowerCase()].parameters[
              uniformParam.id
            ];
            if (uniformParam) {
              uniformParam.id = `${id}_${uniformParam.id}`;
              uniformParam.uniform = newName;
            }
            // add it back to lang with the correct id
            langContent.text.effects[id.toLowerCase()].parameters[
              uniformParam.id
            ] = lang;
          } else {
            console.log(`${id}_${name} not found in data.json`);
          }
          return line;
        }
      );

      processedEffects[id].defines = fxArr.filter((line) => {
        return line.startsWith("#define");
      });

      processedEffects[id].defines = processedEffects[id].defines.map(
        (line) => {
          // defines look like this "#define width 1.0"
          let lineA = line.split(" ");
          let name = lineA[1];
          let newName = `${id.toUpperCase()}_${name}`;
          line = line.replace(name, newName);
          // replace all uses of that define with the new name in the main code
          fxArr = fxArr.map((line) => {
            return line.replaceAll(
              new RegExp(`(?<!\\w)${name}(?!\\w)`, "g"),
              newName
            );
          });
          return line;
        }
      );

      // remove defines
      fxArr = fxArr.filter((line) => {
        return !line.startsWith("#define");
      });
      processedEffects[id].rawCode = fxArr.join("\n");

      // extract all functions
      let functions = {};
      let currentFunction = "";
      let currentFunctionLines = [];
      let functionLineRegex =
        /^(\w+) +(\w+ +)?(\w+) *\(((\w+ +)?(\w+ +)?(\w+ *),? *)*\) *\{? *$/;
      let nbOpenBrackets = 0;
      let isInFunction = false;
      fxArr.forEach((line) => {
        if (isInFunction) {
          currentFunctionLines.push(line);
          if (line.includes("}")) {
            nbOpenBrackets--;
            if (nbOpenBrackets === 0) {
              isInFunction = false;
              if (currentFunction !== "main") {
                let newName = `${id}_${currentFunction}`;
                currentFunctionLines = currentFunctionLines.map((line) => {
                  return line.replaceAll(currentFunction, newName);
                });
                fxArr = fxArr.map((line) => {
                  return line.replaceAll(currentFunction, newName);
                });
                functions[newName] = currentFunctionLines;
              } else {
                functions["main"] = currentFunctionLines;
              }
              currentFunctionLines = [];
              currentFunction = "";
            }
          }
          if (line.includes("{")) {
            nbOpenBrackets++;
          }
        }
        if (!isInFunction) {
          if (line.match(functionLineRegex)) {
            isInFunction = true;
            if (line.match(/\{/g)) nbOpenBrackets++;
            currentFunction = line.match(functionLineRegex)[3];
            currentFunctionLines.push(line);
          }
        }
      });
      // console.log(functions);
      processedEffects[id].functions = functions;

      // remove all lines that are in functions
      fxArr = fxArr.filter((line) => {
        return !Object.keys(functions).some((functionName) => {
          return functions[functionName].some((functionLine) => {
            return line === functionLine;
          });
        });
      });

      // trim lines
      fxArr = fxArr.map((line) => {
        return line.trim();
      });
      // remove empty lines
      fxArr = fxArr.filter((line) => {
        return line.length > 0;
      });
      // console.log(fxArr);
      // if fxArr is not empty, it means that there is some code left, warn the user
      if (fxArr.length > 0) {
        console.log(`Warning: ${id} has some code left`);
      }
    },

    concatEffects(list) {
      let resultArr = [];
      let processedEffects = this.processedEffects;
      let commonUniforms = this.commonUniforms;
      // append common uniforms
      resultArr = resultArr.concat(commonUniforms);

      resultArr.push("");

      // append uniforms for all effects
      list.forEach((id) => {
        let effect = processedEffects[id];
        resultArr = resultArr.concat(effect.uniforms);
        resultArr.push("");
      });

      // append defines for all effects
      list.forEach((id) => {
        let effect = processedEffects[id];
        resultArr = resultArr.concat(effect.defines);
        resultArr.push("");
      });

      // append functions for all effects
      list.forEach((id) => {
        let effect = processedEffects[id];
        Object.keys(effect.functions).forEach((functionName) => {
          if (functionName === "main") {
            return;
          }
          resultArr = resultArr.concat(effect.functions[functionName]);
          resultArr.push("");
        });
      });

      // append main function
      let isLastEffect = false;
      let prevEffectFunction = "";
      list.forEach((id, index) => {
        let effect = processedEffects[id];
        if (index === list.length - 1) {
          isLastEffect = true;
        }
        let newMain = effect.functions.main;
        if (prevEffectFunction !== "") {
          // replace "texture2D(samplerFront" with a call to the previous function
          newMain = newMain.map((line) => {
            return line.replaceAll(
              "texture2D(samplerFront,",
              `${prevEffectFunction}(`
            );
          });
        }
        if (!isLastEffect) {
          // replace main function name with a new name
          let replaced = false;
          prevEffectFunction = `${id}_main`;
          newMain = newMain.map((line) => {
            if (line.includes("main") && !replaced) {
              return line.replaceAll(
                /void main\([^)]*\)/g,
                `mediump vec4 ${prevEffectFunction}(mediump vec2 vTex)`
              );
            }
            return line;
          });
          // replace gl_FragColor ?= ? with return
          newMain = newMain.map((line) => {
            return line.replaceAll(/gl_FragColor *= */g, `return `);
          });
        }
        resultArr = resultArr.concat(newMain);
        resultArr.push("");
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
      let addons = list.map((id) => this.addons.find((x) => x.id === id));
      resultData.name = addons.map((addon) => addon.name).join(" + ");
      resultData.id = addons.map((addon) => addon.id).join("_");
      resultData.author = addons.map((addon) => addon.data.author).join(", ");
      resultData.description =
        "Concat effect of " + addons.map((addon) => addon.name).join(", ");
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
      let addons = list.map((id) => this.addons.find((x) => x.id === id));
      let id = addons.map((addon) => addon.id).join("_");
      resultData.text.effects[id.toLowerCase()] = {
        name: addons
          .map((addon) => addon.lang.text.effects[addon.id.toLowerCase()].name)
          .join(" + "),
        description:
          "Concat effect of " + addons.map((addon) => addon.name).join(", "),
      };
      let params = {};
      addons.forEach((addon) => {
        params = {
          ...params,
          ...addon.lang.text.effects[addon.id.toLowerCase()].parameters,
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
  overflow: hidden;
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
  margin-bottom: 20px;
}
.list-group-item {
  cursor: move;
}
.list-group-item i {
  cursor: pointer;
}

.ghost {
  opacity: 0.5;
  background: #c8ebfb;
}
.list-group-item {
  position: relative;
  display: block;
  padding: 0.75rem 1.25rem;
  margin-bottom: -1px;
  background-color: #fff;
  border: 1px solid rgba(0, 0, 0, 0.125);
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
</style>
