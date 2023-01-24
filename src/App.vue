<template>
  <div id="viewDiv" class="balt-theme">
    <!-- <div id="info" class="esri-widget">
    <h3>Select a feature to display its popup</h3>
    <h4>
      Edit the feature by clicking on the "Edit feature" action within the
      popup.
    </h4>
  </div> -->
  </div>
</template>
<script>
/* eslint-disable */
import { loadModules } from "esri-loader";
import accessPortal from "@/assets/js/accessPortal.json";
export default {
  mounted() {
    this.loadMap();
  },
  methods: {
    loadMap() {
      loadModules(
        [
          "esri/identity/IdentityManager",
          "esri/Map",
          "esri/views/MapView",
          "esri/layers/FeatureLayer",
          "esri/widgets/Search",
          "esri/widgets/Sketch",
          "esri/layers/GraphicsLayer",
          "esri/widgets/Editor",
          "esri/popup/content/AttachmentsContent",
          "esri/popup/content/TextContent",
          "esri/widgets/Home"
        ],
        {
          css: true,
        }
      ).then(([IdentityManager, ArcGISMap, MapView, FeatureLayer, Search, Sketch, GraphicsLayer, Editor, AttachmentsContent, TextContent, Home]) => {
        // create map with the given options
          const map = new ArcGISMap({
            basemap: "topo-vector",
          });
          // assign map to this view
          this.view = new MapView({
            container: this.$el,
            map: map,
            center: [-99.139, 19.376],
            zoom: 8,
            // zoom: 15,
            // center: [-106.48871516310629, 31.7980770813721]
          });
        //#region Recuperacion de Token
          console.log(accessPortal, "accessPortal");
          let self = this;
          var tokenEndpoint = "/rest/generateToken";
          
          const formData = new FormData();
          formData.append("username", accessPortal.portalUsr);
          formData.append("password", accessPortal.portalPwd);
          formData.append("referer", "null" !== window.origin ? window.origin : window.location.origin);
          formData.append("expiration", "60");
          formData.append("f", "json");
          // formData.append("urlAccessPortal", (accessPortal.portalUrl + accessPortal.tokenEndpoint));
          async function getToken(url = "", data = {}) {
            console.log(data);
            
            const response = await fetch(accessPortal.portalUrl + accessPortal.tokenEndpoint, {
              method: "POST",
              body: data,
            });

            return response.json();
          }
          
          getToken(accessPortal.portalUrl + accessPortal.tokenEndpoint, formData)
              .then(data => {
                if(data.error){
                  alert("Usuario y contrasenia incorrecto");
                  // console.warn("Usuario y contrasenia incorrecto");
                  return;
                }
                console.log(data);
                  IdentityManager.registerToken({
                      token: data.token,
                      server: accessPortal.portalUrl,
                      ssl: true,
                  });
                loadMaps(self);
              })
              .catch(error => {
                  console.log(error)
              });

        //#endregion
        function loadMaps(self){

          //#region Editor
          let editor, features;
          /*************************************************************
           * The PopupTemplate content is the text that appears inside the
           * popup. Bracketed {fieldName} can be used to reference the value
           * of an attribute of the selected feature. HTML elements can be
           * used to provide structure and styles within the content.
           **************************************************************/
          const editThisAction = {
            title: "Edit feature",
            id: "edit-this",
            className: "esri-icon-edit"
          };

          // Create a popupTemplate for the featurelayer and pass in a function to set its content and specify an action to handle editing the selected feature
          const template = {
            title: "Trail name: {trailName}",
            content: difficultyLevel,
            fieldInfos: [
              {
                fieldName: "trailName"
              },
              {
                fieldName: "difficulty"
              }
            ],
            actions: [editThisAction]
          };

          // Function that creates two popup elements for the template: attachments and text
          function difficultyLevel(feature) {
            // 1. Set how the attachments should display within the popup
            const attachmentsElement = new AttachmentsContent({
              displayType: "list"
            });

            const textElement = new TextContent();

            const level = feature.graphic.attributes.difficulty;
            const green =
              "<b><span style='color:green'>" + level + "</span></b>.";
            const purple =
              "<b><span style='color:purple'>" + level + "</span></b>.";
            const red = "<b><span style='color:red'>" + level + "</span></b>.";

            // If the feature's "difficulty" attribute is a specific value, set it a certain color
            // "easy" = green
            // "medium" = purple
            // "hard" = red
            switch (level) {
              case "easy":
                textElement.text =
                  "The {trailName} trail has a difficulty level of " + green;
                return [textElement, attachmentsElement];
                break;
              case "medium":
                textElement.text =
                  "The {trailName} trail has a difficulty level of " + purple;
                return [textElement, attachmentsElement];
                break;
              case "hard":
                textElement.text =
                  "The {trailName} trail has a difficulty level of " + red;
                return [textElement, attachmentsElement];
            }
          }
          
          const featureLayer = new FeatureLayer({
            url: "https://services.arcgis.com/V6ZHFr6zdgNZuVG0/arcgis/rest/services/El_Paso_Recreation_AttributeEditsOnly/FeatureServer/1",
            outFields: ["*"],
            popupTemplate: template
          });
          map.add(featureLayer);

          //#endregion

          

          

          const estadosLayer = new FeatureLayer({
            // url: "https://procesosags.sigsa.info/server/rest/services/Francisco_Lopez/MarginacionMexico/MapServer/0",
            url: "https://procesosags.sigsa.info/server/rest/services/Francisco_Lopez/MarginacionMexico/FeatureServer/0",
            // definitionExpression: "1=0"
          });
          map.add(estadosLayer, 0);

          const municipiosLayer = new FeatureLayer({
            url: "https://procesosags.sigsa.info/server/rest/services/Francisco_Lopez/MarginacionMexico/MapServer/1",
          });
          map.add(municipiosLayer, 0);

          //#region Widget HOME
          const homeBtn = new Home({
            view: self.view
          });

          // Add the home button to the top left corner of the view
          self.view.ui.add(homeBtn, "top-left");
          //#endregion

          //#region Continua Editor
          self.view.when(() => {
            // Create the Editor with the specified layer and a list of field configurations
            editor = new Editor({
              view: self.view,
              container: document.createElement("div"),
              layerInfos: [
                {
                  layer: featureLayer,
                  formTemplate: {
                    // autocasts to FormTemplate
                    elements: [
                      // autocasts to FieldElement
                      {
                        type: "field",
                        fieldName: "trailName",
                        label: "Trail name",
                        editable: false
                      },
                      {
                        type: "field",
                        fieldName: "condition",
                        label: "Condition",
                        hint: "The overall condition for running/biking?"
                      },
                      {
                        type: "field",
                        fieldName: "difficulty",
                        label: "Difficulty",
                        hint: "How difficult was this trail to run/bike?"
                      },
                      {
                        type: "field",
                        fieldName: "trckType",
                        label: "Track type",
                        hint: "Running or biking?"
                      },
                      {
                        type: "field",
                        fieldName: "notes",
                        label: "Additional comments"
                      }
                    ]
                  }
                }
              ]
            });

            // Execute each time the "Edit feature" action is clicked
            function editThis() {
              // If the EditorViewModel's activeWorkflow is null, make the popup not visible
              if (!editor.viewModel.activeWorkFlow) {
                self.view.popup.visible = false;
                // Call the Editor update feature edit workflow

                editor.startUpdateWorkflowAtFeatureEdit(
                  self.view.popup.selectedFeature
                );
                self.view.ui.add(editor, "top-right");
                self.view.popup.spinnerEnabled = false;
              }

              // We need to set a timeout to ensure the editor widget is fully rendered. We
              // then grab it from the DOM stack
              setTimeout(() => {
                // Use the editor's back button as a way to cancel out of editing
                let arrComp = editor.domNode.getElementsByClassName(
                  "esri-editor__back-button esri-interactive"
                );
                if (arrComp.length === 1) {
                  // Add a tooltip for the back button
                  arrComp[0].setAttribute(
                    "title",
                    "Cancel edits, return to popup"
                  );
                  // Add a listener to listen for when the editor's back button is clicked
                  arrComp[0].addEventListener("click", (evt) => {
                    // Prevent the default behavior for the back button and instead remove the editor and reopen the popup
                    evt.preventDefault();
                    self.view.ui.remove(editor);
                    self.view.popup.open({
                      features: features
                    });
                  });
                }
              }, 150);
            }

            // Event handler that fires each time an action is clicked
            self.view.popup.on("trigger-action", (event) => {
              if (event.action.id === "edit-this") {
                editThis();
              }
            });
          });

          // Watch when the popup is visible
          self.view.popup.watch("visible", (event) => {
            // Check the Editor's viewModel state, if it is currently open and editing existing features, disable popups
            if (editor.viewModel.state === "editing-existing-feature") {
              self.view.popup.close();
            } else {
              // Grab the features of the popup
              features = self.view.popup.features;
            }
          });

          featureLayer.on("apply-edits", () => {
            // Once edits are applied to the layer, remove the Editor from the UI
            self.view.ui.remove(editor);

            // Iterate through the features
            features.forEach((feature) => {
              // Reset the template for the feature if it was edited
              feature.popupTemplate = template;
            });

            // Open the popup again and reset its content after updates were made on the feature
            if (features) {
              self.view.popup.open({
                features: features
              });
            }

            // Cancel the workflow so that once edits are applied, a new popup can be displayed
            editor.viewModel.cancelWorkflow();
          });

          // self.view.ui.add("info", {
          //   position: "top-left",
          //   index: 1
          // });
          
          //#endregion

          //#region Query a Feature Layer (SQL)
          // SQL query array
          const parcelLayerSQL = [
            "Elija una cláusula where de SQL",
            "GRA_MARG = 'Bajo'",
            "GRA_MARG = 'Medio'",
            "GRA_MARG = 'Alto'",
          ];
          let whereClause = parcelLayerSQL[0];

          // Add SQL UI
          const select = document.createElement("select", "");
          select.setAttribute("class", "esri-widget esri-select");
          select.setAttribute(
            "style",
            "width: 240px; font-family: 'Avenir Next'; font-size: 1em"
          );
          parcelLayerSQL.forEach(function (query) {
            let option = document.createElement("option");
            option.innerHTML = query;
            option.value = query;
            select.appendChild(option);
          });

          self.view.ui.add(select, "top-right");

          // Listen for changes
          select.addEventListener("change", (event) => {
            console.log(event, ">>> event");
            whereClause = event.target.value;
            queryFeatureLayer(self.view);
          });

          function queryFeatureLayer(tempView) {
            const parcelQuery = {
              where: whereClause, // Set by select element
              spatialRelationship: "intersects", // Relationship operation to apply
              geometry: tempView.extent, // Restricted to visible extent of the map
              outFields: ["*"], // Attributes to return
              returnGeometry: true,
            };

            estadosLayer.queryFeatures(parcelQuery)
              .then((results) => {
                displayResults(results, tempView);
              })
              .catch((error) => {
                console.log(error);
              });
          }

          function displayResults(results, tempView) {
            // Create a blue polygon
            const symbol = {
              type: "simple-fill",
              color: [20, 130, 200, 0.5],
              outline: {
                color: "white",
                width: 0.5,
              },
            };

            const popupTemplate = {
              title: "Parcel ",
              content: "Type: x <br> Land value: xy <br> Tax Rate City: xyx",
            };
            // Assign styles and popup to features
            results.features.map((feature) => {
              feature.symbol = symbol;
              feature.popupTemplate = popupTemplate;
              return feature;
            });
            // Clear display
            tempView.popup.close();
            tempView.graphics.removeAll();
            // Add features to graphics layer
            tempView.graphics.addMany(results.features);
          }
          //#endregion

          //#region Query a Feature Lager (Spatial)
          // Add sketch widget
          const graphicsLayerSketch = new GraphicsLayer();
          map.add(graphicsLayerSketch);

          const sketch = new Sketch({
            layer: graphicsLayerSketch,
            view: self.view,
            creationMode: "update" // Auto-select
          });

          self.view.ui.add(sketch, "top-right");
          
          // Add sketch events to listen for and execute query
          sketch.on("update", (event) => {
            // Create
            if (event.state === "start") {
              queryFeaturelayer(event.graphics[0].geometry, self.view);
            }
            if (event.state === "complete"){
              graphicsLayerSketch.remove(event.graphics[0]); // Clear the graphic when a user clicks off of it or sketches new one
            }
            // Change
            if (event.toolEventInfo && (event.toolEventInfo.type === "scale-stop" || event.toolEventInfo.type === "reshape-stop" || event.toolEventInfo.type === "move-stop")) {
              queryFeaturelayer(event.graphics[0].geometry, self.view);
            }
          });

          function queryFeaturelayer(geometry, TempView) {
            const parcelQuery = {
              spatialRelationship: "intersects", // Relationship operation to apply
              geometry: geometry,  // The sketch feature geometry
              outFields: ["*"], // Attributes to return
              returnGeometry: true
            };

            estadosLayer.queryFeatures(parcelQuery)
              .then((results) => {

                displayResults(results, TempView);

              }).catch((error) => {
                console.log(error);
              });
          }
          //#endregion

          //#region Widget de Busqueda
          let searchWidget = new Search({
            view: self.view,
            sources: [
              {
                layer: estadosLayer,
                searchFields: ["FID"],
                name: "ID DEL OBJECT",
                zoomScale: 5000000,
              },
              {
                layer: estadosLayer,
                searchFields: ["ENTIDAD", "FID"],
                name: "NOMBRE DE ENTIDAD",
                zoomScale: 5000000,
              },
              {
                layer: municipiosLayer,
                searchFields: ["NOM_MUN"],
                name: "NOMBRE DE MUNICIPIO",
                zoomScale: 400000,
              },
            ],
          });
          self.view.ui.add(searchWidget, "top-right"); //Add to the map
          //#endregion
        
          //#region Filter a featureLayer with SQL
          // Create a UI with the filter expressions
            const sqlExpressions = [
              "Elija una cláusula where de SQL",
              "GRA_MARG = 'Bajo'",
              "GRA_MARG = 'Medio'",
              "GRA_MARG = 'Alto'",
            ];
            // UI
            const selectFilter = document.createElement("select", "");
            selectFilter.setAttribute("class", "esri-widget esri-select");
            selectFilter.setAttribute("style", "width: 275px; font-family: Avenir Next W00; font-size: 1em;");

            sqlExpressions.forEach(function(sql){
              let option = document.createElement("option");
              option.value = sql;
              option.innerHTML = sql;
              selectFilter.appendChild(option);
            });
            self.view.ui.add(selectFilter, "top-right");

            function setFeatureLayerFilter(expression) {
              estadosLayer.definitionExpression = expression;
            }
            // Event listener
            selectFilter.addEventListener('change', function (event) {
              console.log("Change");
              setFeatureLayerFilter(event.target.value);
            });
          //#endregion
          var divTip = document.createElement("div");
          divTip.setAttribute("id","info");
          divTip.innerHTML = `
          <div class="esri-widget" style="width:150px; padding:7px;"> 
            <h3>Selecciona un feature para mostrar sus detalles</h3> 
            <h4>Edita el feature dando click en "Edit feature" dentro del popup. 
            </h4> 
            <button 
              style="background-color: gray; 
                color: white; 
                padding: 5px; 
                border-radius: 5px; 
                border: 1px solid #000000;
                margin-left: 80px;"
              type="button" 
              onclick="
                let infoDiv = document.getElementById('info');
                infoDiv.innerHTML = '';
                "
            >Cerrar<button>
          </div>`
          self.view.ui.add(divTip,"top-left");
        }
        
      });
    },
  },
};
</script>
<style>
/* esri styles */
@import url("https://js.arcgis.com/4.4/esri/themes/light/main.css");
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
#nav {
  padding: 30px;
}
#nav a {
  font-weight: bold;
  color: #2c3e50;
}
#nav a.router-link-exact-active {
  color: #42b983;
}
#viewDiv {
  position: absolute;
  top: 0;
  bottom: 0;
  left: 0;
  right: 0;
}
</style>