<style scoped>
tr.ve-table-body-tr {
  cursor: pointer;
}

.tool-button {
  max-width: 10ch;
}

.progress-button {
  max-width: 25px;
}

.fl-toolbar {
  margin-bottom: 6px;
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
</style>

<template>
  <div>
    <HeaderNavbar @close-settings="settingsClosed" :extended="true">
      <template v-slot:extension>
        <search-navigator label="Library" :count="totalCount" :enableFirst="!!firstPageLink"
          :enablePrevious="!!previousPageLink" :enableNext="!!nextPageLink" :enableLast="!!lastPageLink"
          :first="firstPage" :previous="previousPage" :next="nextPage" :last="lastPage" :add="addNew"
          :showAdd="false" />
      </template>
    </HeaderNavbar>

    <div class="container-fluid bd-layout" style="padding-top: 114px">
      <v-form class="fl-toolbar">
        <v-row style="align-items: flex-end">
          <v-col>
            <v-text-field label="Name" v-model="searchFor" @input="searchFhirServer" hide-details="auto" />
          </v-col>
          <v-col class="status-col">
            <v-select label="Status" :items="searchPublishingStatuses" v-model="searchForStatus"
              @input="searchFhirServer" hide-details="auto" clearable />
          </v-col>
          <v-col>
            <v-combobox label="Use Context" :items="searchUseContexts" v-model="searchForUseContext"
              @input="searchFhirServer" item-text="display" item-value="code" multiple hide-details="auto" clearable />
          </v-col>
          <v-col>
            <v-text-field label="Publisher" v-model="searchForPublisher" @input="searchFhirServer"
              hide-details="auto" />
          </v-col>
          <v-col class="tool-button">
            <v-btn small @click="clearSearchFields">Clear</v-btn>
          </v-col>
        </v-row>
      </v-form>
      <OperationOutcomeOverlay v-if="outcome" :saveOutcome="outcome" :showOutcome="(outcome != undefined)"
        title="Search Errors/Warnings" :popupWhenErrors="false" @close="outcome = undefined" />
      <ve-table :columns="columns" :table-data="tableData" :event-custom-option="eventCustomOption"
        :expand-option="expandOption" row-key-field-name="id" />
      <div v-show="showEmpty && !loadingData" class="empty-data">
        (No results)
      </div>
      </div>
    <table-loading v-if="loadingData" />
  </div>
</template>

<script lang="ts">
import Vue, { VNode } from "vue";
import {
  LibraryTableDefinition,
  LibraryTableData,
} from "~/models/LibraryTableData";
import { formatDate } from "~/helpers/datetime";
import { getExtensionMarkdownValue } from "fhir-extension-helpers";
import { Bundle, CodeableConcept, UsageContext } from "fhir/r4b";
import { settings } from "~/helpers/user_settings";
import {
  searchPage,
  searchPublishingStatuses,
  toSearchDisplay_UseContext,
} from "~/helpers/searchFhir";
import {
  setFavourite,
  isFavourite,
  unsetFavourite,
} from "~/helpers/favourites";
import { ConformanceResourceTableData } from "~/models/ConformanceResourceTableData";
import { EasyTableDefinition_defaultValues } from "~/models/EasyTableDefinition";
import { ConformanceSearchData } from "models/ConformanceSearchData";
import { loadCustomUseContexts, mergeUseContexts, saveCustomUseContexts } from "~/helpers/useContext_helpers";

export default Vue.extend({
  head: {
    title: "Library",
  },
  mounted() {
    this.showAdvancedSettings = settings.showAdvancedSettings();
    const searchData = settings.getSearchData("Library");
    if (searchData) {
      this.searchForPublisher = searchData.publisher;
      this.searchForStatus = searchData.status;
      this.searchFor = searchData.name;
      this.searchForUseContext = searchData.useContext;
    }
    this.searchFhirServer();
    this.searchUseContexts = loadCustomUseContexts("library", this.defaultUseContexts);
  },
  methods: {
    settingsClosed() {
      this.showAdvancedSettings = settings.showAdvancedSettings();
    },
    clearSearchFields() {
      this.searchFor = undefined;
      this.searchForStatus = "active,draft";
      this.searchForUseContext = [];
      this.searchForPublisher = undefined;
      this.$forceUpdate();
      this.searchFhirServer();
    },

    async firstPage() {
      if (this.firstPageLink) {
        await this.searchPage(this.firstPageLink);
      }
    },
    async previousPage() {
      if (this.previousPageLink) {
        await this.searchPage(this.previousPageLink);
      }
    },
    async nextPage() {
      if (this.nextPageLink) {
        await this.searchPage(this.nextPageLink);
      }
    },
    async lastPage() {
      if (this.lastPageLink) {
        await this.searchPage(this.lastPageLink);
      }
    },
    async addNew() {
      this.$router.push("/Library/:new");
    },

    async searchPage(url: string) {
      await searchPage(this, url, (entries) => {
        var updateRequired = false;
        this.tableData = entries.map<LibraryTableData>((post) => {
          var vs = post.resource as fhir4b.Library;
          const mergeResult = mergeUseContexts(this.searchUseContexts, vs?.useContext);
          if (mergeResult.changed){
            this.searchUseContexts = mergeResult.contexts;
            updateRequired = true;
          }
          return {
            id: vs?.id ?? "",
            title: vs?.title ?? vs?.name ?? vs?.description ?? "(none)",
            url: vs?.url ?? "",
            version: vs?.version ?? "",
            date: formatDate(vs?.date, "", true),
            status: vs?.status ?? "(undefined)",
            useContext: toSearchDisplay_UseContext(vs?.useContext) ?? "",
            publisher: vs?.publisher ?? "",
            description: vs?.description,
            favourite: isFavourite(
              post.resource?.resourceType,
              post.resource?.id
            ),
          };
          if (updateRequired) saveCustomUseContexts("library", this.searchUseContexts, this.defaultUseContexts);
        });
      });
    },

    // https://www.sitepoint.com/fetching-data-third-party-api-vue-axios/
    async searchFhirServer() {
      let url = `${settings.getFhirServerUrl()}/Library?_count=${settings.getPageSize()}&_elements=id,name,title,description,url,version,date,status,publisher,useContext&content-type=text/fhirpath`;
      if (this.searchFor) {
        url += `&title=${encodeURIComponent(this.searchFor)}`;
      }
      if (this.searchForStatus) {
        url += `&status=${encodeURIComponent(this.searchForStatus)}`;
      }
      if (this.searchForPublisher) {
        url += `&publisher=${encodeURIComponent(this.searchForPublisher)}`;
      }
      if (this.searchForUseContext) {
        const contexts = this.searchForUseContext
          .map((coding) => {
            return coding.code;
          })
          .join();
        if (contexts) url += `&context=${contexts}`;
      }
      this.saveSearchData();
      await this.searchPage(url);
    },

    saveSearchData() {
      let searchData: ConformanceSearchData = {
        publisher: this.searchForPublisher,
        status: this.searchForStatus,
        name: this.searchFor,
        useContext: this.searchForUseContext,
      };
      settings.saveSearchData("Library", searchData);
    },

  },
  data(): LibraryTableDefinition {
    return {
      eventCustomOption: {
        bodyRowEvents: ({ row, rowIndex }: any) => {
          return {
            click: (event: PointerEvent) => {
              console.log("click::", row, rowIndex, event);
              var data: LibraryTableData = row;
              console.log("row data::", data);
              if (event.ctrlKey) {
                window.open("/Library/" + data.id, '_blank'); 
              }
              else {
                this.$router.push("/Library/" + data.id);
              }
            },
            contextmenu: (event: PointerEvent) => {
              console.log("contextmenu::", row, rowIndex, event);
              event.preventDefault();
            },
          };
        },
      },
      expandOption: {
        trigger: "icon",
        render: (
          {
            row,
            column,
            rowIndex,
          }: { row: ConformanceResourceTableData; column: any; rowIndex: number },
          h: any
        ): any => {
          return h("ConformanceResourcePreviewRow", { row: row }) as VNode;
        },
      },
      columns: [
        { field: "title", key: "a", title: "Name", align: "left", type: "expand" },
        { field: "version", key: "v", title: "Version", align: "left" },
        { field: "status", key: "c", title: "Status", align: "left" },
        { field: "useContext", key: "uc", title: "Use Context", align: "left" },
        { field: "date", key: "b", title: "Publish Date", align: "left" },
        { field: "publisher", key: "d", title: "Publisher", align: "left" },
        { field: "id", key: "id", title: "ID", align: "left" },
        {
          field: "favourite",
          key: "e",
          title: "",
          align: "center",
          renderBodyCell: (cellData: any, h: any) => {
            if ((cellData.row as ConformanceResourceTableData).favourite)
              return h("FavIcon") as VNode;
            return { text: "" } as VNode;
          },
        },
      ],
      tableData: [],
      outcome: undefined,
      searchFor: undefined,
      searchForStatus: "active,draft",
      searchForUseContext: [],
      searchForPublisher: undefined,
      searchPublishingStatuses: searchPublishingStatuses,
      defaultUseContexts: [
        { code: "demo", display: "Demonstration" },
        { system: "http://fhir.forms-lab.com/CodeSystem/contexts", code: "devdays", display: "DevDays Demo" },
        { code: "fhirpath-lab", display: "FHIRPath Lab Shared"},
        {
          system: "http://fhirpath-lab.com/CodeSystem/user-types",
          code: "unit-test",
          display: "HL7 Unit Test"
        },
      ],
      searchUseContexts: [],
      ...EasyTableDefinition_defaultValues
    };
  },
});
</script>
