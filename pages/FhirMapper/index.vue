<template>
  <div>
    <HeaderNavbar @close-settings="settingsClosed" :extended="false">
    </HeaderNavbar>
    <table-loading v-if="loadingData" />

    <div class="container-fluid bd-layout" style="padding-top: 80px">
      <v-card>
        <v-toolbar flat color="primary">
          <v-toolbar-title>{{ tabTitle() }}</v-toolbar-title>
          <v-spacer />
          <v-select dark class="engineselector" :items="executionEngines" v-model="selectedEngine" hide-details="auto"
            @change="evaluateFhirPathExpression" />
          <v-btn icon dark accesskey="g" title="press alt+g to go" @focus="checkFocus"
            @click="evaluateFhirPathExpression">
            <v-icon>
              mdi-play
            </v-icon>
          </v-btn>
        </v-toolbar>
        <v-row dense>
          <v-col>
            <v-tabs vertical v-model="tab">
              <v-tab>
                <v-icon left> mdi-function-variant </v-icon>
                Map
              </v-tab>
              <v-tab class="left-resource">
                <v-icon left> mdi-clipboard-text-outline </v-icon>
                Resource
              </v-tab>
              <v-tab :disabled="!hasTraceData()">
                <v-icon left> mdi-format-list-bulleted </v-icon>
                Trace
              </v-tab>
              <v-tab>
                <v-icon left> mdi-bug-outline </v-icon>
                Debug
              </v-tab>

              <v-tabs-items touchless v-model="tab">
                <v-tab-item :eager="true">
                  <!-- Expression -->
                  <v-card flat>
                    <v-card-text>
                      <p class="fl-tab-header">Map</p>
                      <label class="v-label theme--light bare-label">Fhir Map </label>
                      <div height="85px" width="100%" ref="aceEditorExpression"></div>
                      <div class="ace_editor_footer"></div>

                      <div class="results">RESULTS <span class="processedBy">{{ processedByEngine }}</span></div>
                      <template v-for="(r2, i1) in results">
                        <v-simple-table :key="i1">
                          <tr v-if="r2.context">
                            <td class="context" colspan="2">
                              <div>Context: <b>{{ r2.context }}</b></div>
                            </td>
                          </tr>
                          <template v-for="(v1, index) in r2.result">
                            <tr :key="index">
                              <td class="result-value">
                                <div class="code-json">{{ v1.value }}</div>
                              </td>
                              <td class="result-type"><i v-if="v1.type">({{ v1.type }})</i></td>
                            </tr>
                          </template>
                        </v-simple-table>
                      </template>
                    </v-card-text>
                  </v-card>
                </v-tab-item>

                <v-tab-item :eager="true">
                  <!-- Resource -->
                  <v-card flat>
                    <v-card-text>
                      <p class="fl-tab-header">Resource</p>
                      <v-text-field label="Test Resource Id" v-model="resourceId" hide-details="auto" autocomplete="off"
                        autocorrect="off" autocapitalize="off" spellcheck="false">
                        <template v-slot:append>
                          <v-btn icon small tile @click="resourceId = undefined">
                            <v-icon> mdi-close </v-icon>
                          </v-btn>
                          <v-btn icon small tile @click="downloadTestResource">
                            <v-icon> mdi-download </v-icon>
                          </v-btn>
                        </template>
                      </v-text-field>
                      <label class="v-label theme--light bare-label">Test Resource JSON <i>{{
                      resourceJsonChangedMessage() }}</i></label>
                      <div height="85px" width="100%" ref="aceEditorResourceJsonTab"></div>
                      <!-- <div class="ace_editor_footer"></div> -->
                    </v-card-text>
                  </v-card>
                </v-tab-item>

                <v-tab-item>
                  <!-- Trace -->
                  <v-card flat>
                    <v-card-text>
                      <p class="fl-tab-header">Trace</p>
                      <template v-for="(r2, i1) in results">
                        <v-simple-table :key="i1">
                          <tr v-if="r2.context">
                            <td class="context" colspan="3">
                              <div>Context: <b>{{ r2.context }}</b></div>
                            </td>
                          </tr>
                          <template v-for="(v1, index) in r2.trace">
                            <tr :key="index">
                              <td class="result-value">
                                <div :class="traceTypeClass(v1.type)">{{ v1.value }}</div>
                              </td>
                            </tr>
                          </template>
                        </v-simple-table>
                      </template>
                    </v-card-text>
                  </v-card>
                </v-tab-item>

                <v-tab-item :eager="true">
                  <!-- Debug -->
                  <v-card flat>
                    <v-card-text>
                      <p class="fl-tab-header">Debug</p>
                      <div ref="aceEditorDebug"></div>
                    </v-card-text>
                  </v-card>
                </v-tab-item>

              </v-tabs-items>
            </v-tabs>
          </v-col>
          <v-col class="right-resource">
            <v-card flat>
              <v-card-text>
                <p class="fl-tab-header">Resource</p>
                <v-text-field label="Test Resource Id" v-model="resourceId" hide-details="auto" autocorrect="off"
                  autocapitalize="off" spellcheck="false">
                  <template v-slot:append>
                    <v-btn icon small tile @click="resourceId = undefined">
                      <v-icon> mdi-close </v-icon>
                    </v-btn>
                    <v-btn icon small tile @click="downloadTestResource">
                      <v-icon> mdi-download </v-icon>
                    </v-btn>
                  </template>
                </v-text-field>
                <label class="v-label theme--light bare-label">Test Resource JSON <i>{{ resourceJsonChangedMessage()
                }}</i></label>
                <div height="85px" width="100%" ref="aceEditorResourceJsonRight"></div>
              </v-card-text>
            </v-card>
          </v-col>
        </v-row>
      </v-card>

      <br />
      <OperationOutcomeOverlay v-if="showOutcome" :saveOutcome="saveOutcome" :showOutcome="showOutcome" title="Error"
        @close="clearOutcome" />
    </div>
  </div>
</template>

<style lang="scss" scoped>
.ace_editor:focus-within+.ace_editor_footer {
  color: #1976d2;
  transition: 0.3s cubic-bezier(0.25, 0.8, 0.5, 1);
}

.trace_debug {
  color: grey;
  font-family: 'Courier New', Courier, monospace;
  white-space: pre-wrap;
}

.trace_info {
  color: #1976d2;
}
</style>

<style lang="scss" scoped >
tr.ve-table-body-tr {
  cursor: pointer;
}

.left-resource {
  display: flex;
}
.right-resource {
    display: none;
  }

@media (max-width: 1000px) {
  .left-resource {
    display: flex;
  }

  .right-resource {
    display: none;
  }
}

.engineselector {
  max-width: 14ch;
}

.tool-button {
  max-width: 10ch;
}

td {
  vertical-align: top;
  height: unset !important;
  padding: 8px;
}

.progress-button {
  max-width: 25px;
}

.fl-toolbar {
  margin-bottom: 6px;
}

.result-type {
//  border-bottom: silver 1px solid;
}

.result-value {
  width: 100%;
//  border-bottom: silver 1px solid;
  padding-top: 0px;
  padding-bottom: 0px;
}

.bare-label {
  transform-origin: top left;
  transform: scale(0.75);
  margin-top: 8px;
  height: 20px;
}

.results {
  padding: 4px 12px;
  color: white;
  font-style: bold;
  font-size: 1.25rem;
  line-height: 1.5;
  background-color: $card-header-color;
  margin-top: 10px;
}

.context {
  border-bottom: silver 1px solid;
  background-color: $tab-backcolor;
}

.empty-data {
  display: flex;
  align-items: center;
  justify-content: center;
  height: min(200px, 80vh);
  width: 100%;
  color: #666;
  font-size: 16px;
  border: 1px solid #eee;
  border-top: 0;
}

.code-json {
  white-space: pre-wrap;
}

.processedBy {
  float: right;
  font-size: small;
}
</style>

<script lang="ts">
import Vue, { VNode } from "vue";
import { settings } from "~/helpers/user_settings";
import {
  requestFhirAcceptHeaders,
  requestFhirContentTypeHeaders,
  fhirResourceTypes,
} from "~/helpers/searchFhir";
import axios, { AxiosRequestHeaders, AxiosResponse } from "axios";
import { AxiosError } from "axios";
import { CancelTokenSource } from "axios";
import { getExtensionStringValue } from "fhir-extension-helpers";
// import { getPreferredTerminologyServerFromSDC } from "fhir-sdc-helpers";
import fhirpath from "fhirpath";
import fhirpath_r4_model from "fhirpath/fhir-context/r4";
import { Rules as FhirPathHightlighter_Rules, setCustomHighlightRules } from "~/helpers/fhirpath_highlighter"
import "~/assets/fhirpath_highlighter.scss"
import { IApplicationInsights } from '@microsoft/applicationinsights-web'

import "ace-builds";
import ace from "ace-builds";
import "ace-builds/src-noconflict/mode-text";
import "ace-builds/src-noconflict/mode-json";
import "ace-builds/src-noconflict/theme-chrome";

import { fhir } from '@fhir-typescript/r4b-core';
import { compressToEncodedURIComponent, decompressFromEncodedURIComponent } from "lz-string";
import { TestFhirMapData } from "~/models/testenginemodel";

const shareTooltipText = 'Copy a sharable link to this test expression';
const shareZulipTooltipText = 'Copy a sharable link for Zulip to this test expression';

interface FhirMapData {
  prevFocus?: any;
  raw?: fhir4.Parameters;
  resourceId?: string;
  resourceJsonChanged: boolean;
  loadingData: boolean;
  saveOutcome?: fhir4.OperationOutcome;
  showOutcome?: boolean;
  cancelSource?: CancelTokenSource;
  tab: any;
  results: ResultData[];
  selectedEngine: string;
  executionEngines: string[];
  expressionEditor?: ace.Ace.Editor;
  expressionContextEditor?: ace.Ace.Editor;
  debugEditor?: ace.Ace.Editor;
  resourceJsonEditor?: ace.Ace.Editor;
  processedByEngine?: string;
}

interface ResultItem {
  type: string;
  value: any;
}

interface ResultData {
  context?: string;
  result: ResultItem[];
  trace: TraceData[];
}

interface TraceData {
  name: string;
  type?: string;
  value: string;
}

function canonicalVariableName(name: string): string {
  if (name.startsWith("%")) name = name.substring(1);
  if (name.startsWith("`")) name = name.substring(1);
  if (name.endsWith("`")) name = name.substring(0, name.length - 1);
  if (name.indexOf('`') !== -1) name = name.replaceAll('\\`', '`');
  return name;
}

function isSystemVariableName(name: string): boolean {
  if (name === "ucum") return true;
  if (name === "resource") return true;
  if (name === "rootResource") return true;
  if (name === "context") return true;
  if (name === "terminologies") return true;
  return false;
}

function getValue(entry: fhir4.ParametersParameter): ResultItem[] {
  let result: ResultItem[] = [];
  var myMap = new Map(Object.entries(entry));
  for (let [k, v] of myMap.entries()) {
    if (k.startsWith('value'))
      result.push({ type: k.replace('value', ''), value: v });
    else if (k == 'resource')
      result.push({ type: (v as fhir4.Resource).resourceType, value: v });
  }
  const extVal = getExtensionStringValue(entry, "http://fhir.forms-lab.com/StructureDefinition/json-value");
  if (extVal)
    result.push({ type: entry.name, value: JSON.parse(extVal) });
  return result;
}
function getTraceValue(entry: fhir4.ParametersParameter): TraceData[] {
  let result: TraceData[] = [];
  if (entry.part) {
    for (var part of entry.part) {
      const val = getValue(part);
      if (part.name === "debug"){
        result.push({ name: entry.valueString ?? '', type: part.name, value: val[0].value });
      }
      else{
        result.push({ name: entry.valueString ?? '', type: part.name, value: JSON.stringify(val[0].value, null, 4) });
      }
    }
  }
  return result;
}

export default Vue.extend({
  components: {
  },
  head: {
    title: "FhirPathTester",
  },
  async mounted() {
    const CDN = 'https://cdn.jsdelivr.net/npm/ace-builds@1.6.0/src-min-noconflict';
    if (true) {
      ace.config.set('basePath', CDN);
      ace.config.set('modePath', CDN);
      ace.config.set('themePath', CDN);
      ace.config.set('workerPath', CDN);
    }

    // Update the editor's Mode
    var editorDiv: any = this.$refs.aceEditorExpression as Element;
    if (editorDiv) {
      this.expressionEditor = ace.edit(editorDiv, {
        wrap: "free",
        minLines: 3,
        maxLines: 37,
        highlightActiveLine: false,
        showGutter: true,
        fontSize: 14,
        cursorStyle: "slim",
        showPrintMargin: false,
        theme: "ace/theme/chrome",
        mode: "ace/mode/text",
        wrapBehavioursEnabled: true
      });

      setCustomHighlightRules(this.expressionEditor, FhirPathHightlighter_Rules);
      this.expressionEditor.setValue(`/// name = "SDOHCC-PRAPARE-Map"
/// status = draft
/// version = 0.1

map "http://fhirpath-lab.com/StructureMap/intro-patient-map" = "IntroPatientMap"

uses "http://hl7.org/fhir/StructureDefinition/Patient" as source
uses "http://hl7.org/fhir/StructureDefinition/Bundle" as target

group patientMap(source src : Patient, target bundle : Bundle) {
  src -> bundle.id = uuid() "bundle-id";
  src -> bundle.type = 'transaction' "bundle-type";

  // create a new entry and put the patient resource in it
  src -> bundle.entry as entry, entry.resource = src then
    SetEntryData(src, entry) "prep-entry";
}

group SetEntryData(source src: Patient, target entry)
{
  src.id as patientId log('patientId: ' & %src.id) -> entry.fullUrl = append('http://hl7.org/fhir/us/sdoh-clinicalcare/Patient/', patientId);
  
  src -> entry.request as request then {
    src -> request.method = 'POST' "obsn-request-method";
    src -> request.url = 'Observation' "obsn-request-url";
  } "obsn-entry-request";
}
      `);
      this.expressionEditor.clearSelection();
      this.expressionEditor.on("change", this.fhirpathExpressionChangedEvent)
    }

    var editorDebugDiv: any = this.$refs.aceEditorDebug as Element;
    if (editorDebugDiv) {
      this.debugEditor = ace.edit(editorDebugDiv, {
        wrap: "free",
        readOnly: true,
        minLines: 15,
        maxLines: 35,
        highlightActiveLine: true,
        showGutter: true,
        fontSize: 14,
        cursorStyle: "slim",
        showPrintMargin: false,
        theme: "ace/theme/chrome",
        mode: "ace/mode/json",
        wrapBehavioursEnabled: true
      });
    }

    const resourceEditorSettings: Partial<ace.Ace.EditorOptions> = {
      wrap: "free",
      minLines: 15,
      maxLines: 30,
      highlightActiveLine: true,
      showGutter: true,
      fontSize: 14,
      cursorStyle: "slim",
      showPrintMargin: false,
      theme: "ace/theme/chrome",
      mode: "ace/mode/json",
      wrapBehavioursEnabled: true
    };
    var editorResourceJsonRightDiv: any = this.$refs.aceEditorResourceJsonRight as Element;
    if (editorResourceJsonRightDiv) {
      this.resourceJsonEditor = ace.edit(editorResourceJsonRightDiv, resourceEditorSettings);
      this.resourceJsonEditor.session.on("change", this.resourceJsonChangedEvent);

      var editorResourceJsonLeftDiv: any = this.$refs.aceEditorResourceJsonTab as Element;
      if (editorResourceJsonLeftDiv) {
        let editor = ace.edit(editorResourceJsonLeftDiv, resourceEditorSettings);
        if (editor) {
          editor.setSession(this.resourceJsonEditor.session);
        }
      }
    }

    // Check for the encoded parameters first
    const parameters = this.$route.query.parameters as string;
    let data: TestFhirMapData;
    // Read in any parameters from the URL
    data = this.readParametersFromQuery();

    await this.applyParameters(data);
    await this.evaluateFhirPathExpression();
    this.loadingData = false;
  },
  methods: {
    readParametersFromQuery(): TestFhirMapData {
      let data: TestFhirMapData = {
        expression: this.$route.query.expression as string
      };
      {
        if (this.$route.query.context) {
          data.context = this.$route.query.context as string;
        }

        if (this.$route.query.resource) {
          data.resource = this.$route.query.resource as string;
        }

        const resourceJson = this.$route.query.resourceJson + '';
        if (resourceJson) {
          data.resourceJson = decompressFromEncodedURIComponent(resourceJson) ?? '';
        }

        if (this.$route.query.engine) {
          data.engine = this.$route.query.engine as string ?? '';
        }
      }
      return data;
    },
    async applyParameters(p: TestFhirMapData) {
      {
        if (p.expression) {
          if (p.resource) {
            this.resourceId = p.resource;
          }

          if (this.expressionContextEditor) {
            if (p.context) {
              this.expressionContextEditor.setValue(p.context ?? '');
              this.expressionContextEditor.clearSelection();
            }
            else {
              this.expressionContextEditor.setValue('');
            }
          }

          const resourceJson = p.resourceJson;
          if (resourceJson) {
            this.resourceJsonEditor?.setValue(JSON.stringify(JSON.parse(resourceJson), null, 2));
            this.resourceJsonChanged = true;
            this.resourceId = undefined;
            this.resourceJsonEditor?.clearSelection();
          }

          if (p.engine) {
            this.selectedEngine = p.engine ?? '';
          }

          if (this.expressionEditor) {
            this.expressionEditor.setValue(p.expression ?? '');
            this.expressionEditor.clearSelection();
          }
        }
      }
    },
    resourceJsonChangedEvent() {
      this.resourceJsonChanged = true;
    },
    fhirpathExpressionChangedEvent() {
    },
    resourceJsonChangedMessage(): string | undefined {
      if (this.resourceJsonChanged && this.resourceId) {
        return '(modified)';
      }
    },
    tabTitle() {
      if (this.getResourceJson() && this.resourceJsonChanged) return '(local resource JSON)';
      return this.resourceId;
    },
    settingsClosed() {
    },

    traceTypeClass(category: string|undefined): string {
      if (category === "debug") return "trace_debug";
      if (category === "info") return "trace_info";
      return "code-json";
    },

    getContextExpression(): string | undefined {
      const json = this.expressionContextEditor?.getValue();
      if (json && json.length > 0) {
        return json;
      }
      return undefined;
    },

    getFhirpathExpression(): string | undefined {
      const json = this.expressionEditor?.getValue();
      if (json && json.length > 0) {
        return json;
      }
      return undefined;
    },

    getResourceJson(): string | undefined {
      const json = this.resourceJsonEditor?.getValue();
      if (json && json.length > 0) {
        return json;
      }
      return undefined;
    },

    hasTraceData(): boolean {
      if (this.results.length === 0) return false;
      for (var rd of this.results) {
        if (rd.trace.length > 0) return true;
      }
      return false;
    },

    clearOutcome() {
      this.showOutcome = undefined;
    },

    setResultJson(result: string) {
      if (this.debugEditor) {
        this.debugEditor.setValue(result);
        this.debugEditor.clearSelection();
        this.debugEditor.renderer.updateFull(true);
      }
    },
    async executeRequest<T>(url: string, p: fhir4.Parameters) {
      try {
        if (this.cancelSource) this.cancelSource.cancel("new map test started");
        this.cancelSource = axios.CancelToken.source();
        this.loadingData = true;
        let token = this.cancelSource.token;
        let response: AxiosResponse<fhir4.Parameters | fhir4.OperationOutcome, any>;
        response = await axios.post<fhir4.Parameters>(url, p,
          {
            cancelToken: token,
            headers: {
              "Accept": requestFhirAcceptHeaders,
              "ContentType": requestFhirContentTypeHeaders
            }
          });
        if (token.reason) {
          console.log(token.reason);
          return;
        }
        this.cancelSource = undefined;
        this.loadingData = false;

        const results = response.data;
        if (results) {
          this.setResultJson(JSON.stringify(results, null, 4));
          if (results.resourceType === 'OperationOutcome') {
            this.saveOutcome = results;
            this.showOutcome = true;
            return;
          }
          this.raw = results;

          this.results = [];
          if (this.raw.parameter) {
            for (var entry of this.raw.parameter) {
              if (entry.name === 'parameters') {
                // read the processing engine version
                if (entry.part && entry.part.length > 0 && entry.part[0].name === 'evaluator') {
                  this.processedByEngine = entry.part[0].valueString;
                }

                continue; // skip over the configuration settings
              }

              // Anything else is a result
              // scan over the parts (values)
              let resultItem: ResultData = { context: entry.valueString, result: [], trace: [] };
              if (entry.part) {
                for (var part of entry.part) {
                  if (part.name === 'trace') {
                    resultItem.trace.push(...getTraceValue(part));
                  }
                  else {
                    resultItem.result.push(...getValue(part));
                  }
                }
              }
              this.results.push(resultItem);
            }
          }
        }
      } catch (err) {
        this.loadingData = false;
        if (axios.isAxiosError(err)) {
          const serverError = err as AxiosError<fhir4.OperationOutcome>;
          if (serverError && serverError.response) {
            this.setResultJson(JSON.stringify(serverError.response.data, null, 4));
            this.saveOutcome = serverError.response.data;
            this.showOutcome = true;
            return serverError.response.data;
          }
        } else {
          console.log("Client Error:", err);
        }
      }
    },

    async downloadTestResource() {
      try {
        if (!this.resourceId) return;
        let url = this.resourceId;
        if (this.resourceId && !this.resourceId.startsWith('http'))
          url = settings.getFhirServerUrl() + '/' + this.resourceId;

        if (this.cancelSource) this.cancelSource.cancel("new download started");
        this.cancelSource = axios.CancelToken.source();
        this.loadingData = true;
        let token = this.cancelSource.token;
        let headers: AxiosRequestHeaders = {
          "Cache-Control": "no-cache",
          "Accept": requestFhirAcceptHeaders
        }
        const response = await axios.get<fhir4.Resource>(url, {
          cancelToken: token,
          headers: headers
        });
        if (token.reason) {
          console.log(token.reason);
          return;
        }
        this.cancelSource = undefined;
        this.loadingData = false;

        const results = response.data;
        if (results) {
          if (this.resourceJsonEditor) {
            const resourceJson = JSON.stringify(results, null, 4);
            if (resourceJson) {
              this.resourceJsonEditor.setValue(resourceJson);
              this.resourceJsonChanged = false;
            }
            this.resourceJsonEditor.clearSelection();
          }
        }
      } catch (err) {
        this.loadingData = false;
        if (axios.isAxiosError(err)) {
          const serverError = err as AxiosError<fhir4.OperationOutcome>;
          if (serverError && serverError.response) {
            this.setResultJson(JSON.stringify(serverError.response, null, 4));
            if (serverError.response.data?.resourceType == 'OperationOutcome') {
              this.setResultJson(JSON.stringify(serverError.response, null, 4));
              this.saveOutcome = serverError.response.data;
            } else {
              if (serverError.response.status == 404)
                this.saveOutcome = { resourceType: 'OperationOutcome', issue: [] }
              this.saveOutcome?.issue.push({ code: 'not-found', severity: 'error', details: { text: 'Test resource not found' } });
            }
            this.showOutcome = true;
            return serverError.response.data;
          }
        } else {
          console.log("Client Error:", err);
        }
      }
    },

    checkFocus(event: any) {
      if (event.relatedTarget) {
        this.prevFocus = event.relatedTarget;
      }
    },

    // https://www.sitepoint.com/fetching-data-third-party-api-vue-axios/
    async evaluateFhirPathExpression() {

      // reset the processing engine
      this.processedByEngine = undefined;

      // Validate the test fhir resource object
      let resourceJson = this.getResourceJson();
      if (resourceJson) {
        let rawObj: object;
        try {
          rawObj = JSON.parse(resourceJson)
          let resource: fhir.FhirResource | null = fhir.resourceFactory(rawObj);
          if (resource) {
            const issues: fhir.FtsIssue[] = resource.doModelValidation();
            if (issues.length !== 0) {
              this.saveOutcome = { resourceType: 'OperationOutcome', issue: [] }
              this.saveOutcome?.issue.push(...issues as any);
              this.showOutcome = true;
            }
          }
        } catch (err) {
          console.log(err);
          this.saveOutcome = { resourceType: 'OperationOutcome', issue: [] }
          this.saveOutcome?.issue.push({ code: 'exception', severity: 'error', details: { text: `Failed to parse the resource: ${err}` } });
          this.showOutcome = true;
          this.loadingData = false;
        }
      }

      // brianpos hosted service
      // default the firely SDK/brianpos service
      // Source code for this is at https://github.com/brianpos/fhirpath-lab-dotnet
      let url = `https://fhir-mapping-lab.azurewebsites.net/StructureMap/$transform?debug=true`;
      // let url = `https://localhost:7089/StructureMap/$transform?debug=true`;
      // let url = `http://localhost:7071/api/$fhirpath`;

      let p: fhir4.Parameters = {
        resourceType: "Parameters",
        parameter: [{ name: "map", valueString: this.getFhirpathExpression() ?? 'today()' }]
      };

      const contextExpression = this.getContextExpression();
      if (contextExpression) {
        p.parameter?.push({ name: "context", valueString: contextExpression });
      }

      // for initial testing with .net
      if (!this.getResourceJson() && this.resourceId) {
        await this.downloadTestResource();
        resourceJson = this.getResourceJson();
      }

      if (this.selectedEngine == "java (matchbox)") {
        url = `https://test.ahdis.ch/fhir/StructureMap/$transform`;

        if (!this.getResourceJson() && this.resourceId) {
          await this.downloadTestResource();
          resourceJson = this.getResourceJson();
        }

        (this as any).$appInsights?.trackEvent({ name: 'evaluate Matchbox (map)' });
      }
      else {
        (this as any).$appInsights?.trackEvent({ name: 'evaluate .NET (map)' });
      }

      if (resourceJson) {
        p.parameter?.push({ name: "resource", resource: JSON.parse(resourceJson) });
      }
      else {
        if (!this.resourceId?.startsWith('http')) {
          p.parameter?.push({ name: "resource", valueString: settings.getFhirServerUrl() + '/' + this.resourceId });
        }
        else {
          p.parameter?.push({ name: "resource", valueString: this.resourceId });
        }
      }

      await this.executeRequest(url, p);

      // Set focus to the control that previously had focus (if was known)
      if (this.prevFocus) {
        this.prevFocus.focus();
      }
    },
  },
  data(): FhirMapData {
    return {
      prevFocus: null,
      tab: null,
      raw: undefined,
      resourceId: 'Patient/example',
      resourceJsonChanged: false,
      loadingData: true,
      saveOutcome: undefined,
      showOutcome: false,
      results: [],
      selectedEngine: ".NET (brianpos)",
      executionEngines: [
        ".NET (brianpos)",
        "java (matchbox)"
      ],
      expressionEditor: undefined,
      expressionContextEditor: undefined,
      debugEditor: undefined,
      resourceJsonEditor: undefined,
      processedByEngine: undefined,
    };
  },
});
</script>