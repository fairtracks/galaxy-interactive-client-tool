<template>
    <div>
        <ToolCard
            v-if="showTool"
            :id="toolConfig.id"
            :version="toolConfig.version"
            :title="toolConfig.name"
            :description="toolConfig.description"
            :options="toolConfig"
            :disabled="disableTool"
            :allow-object-store-selection="config.object_store_allows_id_selection"
            :preferred-object-store-id="preferredObjectStoreId"
            @onChangeVersion="onChangeVersion"
            @onSetError="onSetError"
            @updatePreferredObjectStoreId="onUpdatePreferredObjectStoreId">
            <template v-slot:extra-tool-options-items>
                <b-dropdown-item :disabled="!serviceToolExists" @click="openServiceTool">
                    <FontAwesomeIcon icon="fa-wrench" /><span v-localize>Open Service tool</span>
                </b-dropdown-item>
            </template>

            <template v-slot:body>
                <slot name="tool-messages" />
                <CenterFrame
                    id="interactive_client_frame"
                    :height="iframeLoaded ? iframeHeight : '0px'"
                    :src="serviceStarting ? '' : entryPointTarget"
                    @load="onLoadIframe()" />
                <FontAwesomeIcon v-if="iframeLoading" icon="spinner" class="ml-3 mr-2" spin />
            </template>

            <template v-slot:header-buttons>
                <ButtonSpinner
                    v-if="!entryPointTarget || serviceStarting"
                    id="start-service-button"
                    :title="serviceStarting ? 'Starting Service...' : 'Start Service'"
                    class="btn-sm"
                    :disabled="!serviceToolExists"
                    :wait="serviceStarting"
                    tooltip="Start the related interactive tool service"
                    @onClick="startService()" />

                <ButtonSpinner
                    v-if="entryPointTarget && !serviceStarting"
                    id="stop-service-button"
                    :title="serviceStopping ? 'Stopping Service...' : 'Stop Service'"
                    class="btn-sm"
                    icon="stop"
                    :disabled="!serviceToolExists"
                    :wait="serviceStopping"
                    tooltip="Stop the related interactive tool service"
                    @onClick="stopService()" />
            </template>
            <template v-slot:buttons> </template>
        </ToolCard>
    </div>
</template>

<script>
import { FontAwesomeIcon } from "@fortawesome/vue-fontawesome";
import axios from "axios";
import { Services } from "components/InteractiveTools/services";
import { useConfig } from "composables/config";
import { getAppRoot } from "onload/loadConfig";
import { mapActions, mapState } from "pinia";
import { useEntryPointStore } from "stores/entryPointStore";
import { useJobStore } from "stores/jobStore";
import { refreshContentsWrapper } from "utils/data";
import { withPrefix } from "utils/redirect";

import { getToolFormData, submitJob } from "./services";

import ToolCard from "./ToolCard.vue";
import ButtonSpinner from "components/Common/ButtonSpinner.vue";
import CenterFrame from "entry/analysis/modules/CenterFrame.vue";

export default {
    components: {
        CenterFrame,
        ButtonSpinner,
        FontAwesomeIcon,
        ToolCard,
    },
    props: {
        toolConfig: {
            type: Object,
            required: true,
        },
        currentVersion: {
            type: String,
            required: true,
        },
        showTool: {
            type: Boolean,
            required: true,
        },
        disableTool: {
            type: Boolean,
            default: false,
        },
    },
    setup() {
        const { config, isConfigLoaded } = useConfig(true);
        return { config, isConfigLoaded };
    },
    data() {
        return {
            preferredObjectStoreId: null, // Placeholder - support added later, if needed
            clientToolConfigInspected: false,
            clientToolConfigValid: false,
            entryPoint: null,
            allServiceToolEntryPoints: [],
            triedToFetchServiceTool: false,
            serviceToolConfig: null,
            serviceStarting: false,
            serviceStopping: false,
            iframeLoaded: false,
            refreshIframeHeightFlip: false,
        };
    },
    computed: {
        ...mapState(useEntryPointStore, { activeEntryPoints: "entryPoints" }),

        serviceToolId() {
            console.log(this.toolConfig.ict_server_tool_id);
            return this.toolConfig.ict_server_tool_id;
        },
        serviceToolVersion() {
            console.log(this.toolConfig.ict_server_tool_version);
            return this.toolConfig.ict_server_tool_version;
        },
        serviceToolEntryPointLabel() {
            console.log(this.toolConfig.ict_server_entrypoint_label);
            return this.toolConfig.ict_server_entrypoint_label;
        },
        serviceToolExists() {
            return true;
            console.log(`serviceToolsExists: ${this.triedToFetchServiceTool && this.serviceToolConfig !== null}`);
            return this.triedToFetchServiceTool && this.serviceToolConfig !== null;
        },
        entryPointTarget() {
            if (this.entryPoint && this.entryPoint.target) {
                const params = new URLSearchParams(this.toolConfig.state_inputs);
                const url = `${this.entryPoint.target}?${params.toString()}`;
                console.log(url);
                return url;
            } else {
                console.log(null);
                return null;
            }
        },
        iframeLoading() {
            return this.entryPointTarget && !this.iframeLoaded;
        },
        iframeHeight() {
            this.refreshIframeHeightFlip;

            if (this.iframeLoaded) {
                const iframeObj = this.getIframeObj();
                if (iframeObj) {
                    const win = iframeObj.contentWindow;
                    const html = win.document.documentElement;

                    // Idea from https://stackoverflow.com/a/64110252
                    if (html) {
                        html.style.overflowX = "auto";
                        html.style.overflowY = "hidden";
                        const htmlStyle = win.getComputedStyle(html);
                        const htmlHeight = parseInt(htmlStyle.getPropertyValue("height"));

                        return `${htmlHeight}px`;
                    }
                }
            }
            return "0px";
        },
    },
    watch: {
        toolConfig() {
            console.log("load");
            this.load();
        },
        async clientToolConfigInspected() {
            console.log("clientToolConfigInspected");
            if (!this.clientToolConfigInspected) {
                this.inspectClientToolConfig();
            } else {
                await this.fetchEntryPoint();
            }
        },
        async activeEntryPoints() {
            console.log("activeEntryPoints");
            await this.fetchEntryPoint();
        },
        async entryPointTarget() {
            console.log("entryPointTarget");
            if (this.entryPointTarget && this.serviceStarting) {
                await this.waitForPageLoad();
            }
            if (!this.entryPointTarget) {
                this.iframeLoaded = false;
            }
        },
    },
    async created() {
        console.log("created");
        this.root = getAppRoot();
        this.interactiveToolsServices = new Services({ root: this.root });
        await this.ensurePollingEntryPoints();
        this.load();
    },
    methods: {
        ...mapActions(useEntryPointStore, ["ensurePollingEntryPoints", "fetchEntryPoints"]),

        load() {
            this.resetData();
            this.$nextTick(() => {
                this.inspectClientToolConfig();
            });
        },
        resetData() {
            this.clientToolConfigInspected = false;
            this.clientToolConfigValid = false;
            this.entryPoint = null;
            this.allServiceToolEntryPoints = [];
            this.triedToFetchServiceTool = false;
            this.serviceToolConfig = null;
            this.iframeLoaded = false;
            this.serviceStarting = false;
        },
        async onChangeVersion(newVersion) {
            this.$emit("onChangeVersion", newVersion);
        },
        onSetError(errorObj) {
            this.$emit("onSetError", errorObj);
        },
        onUpdatePreferredObjectStoreId(preferredObjectStoreId) {
            this.preferredObjectStoreId = preferredObjectStoreId;
        },
        inspectClientToolConfig() {
            console.log("inspecting client tool config...");
            const validate_properties = [
                { value: this.serviceToolId, attr: "ict_server_tool_id", desc: "Tool id" },
                { value: this.serviceToolVersion, attr: "ict_server_tool_version", desc: "Tool version" },
                {
                    value: this.serviceToolEntryPointLabel,
                    attr: "ict_server_entrypoint_label",
                    desc: "Entry point label",
                },
            ];

            for (const prop of validate_properties) {
                if (!prop.value) {
                    this.onSetError({
                        message: `Error: ${prop.desc} for required interactive server tool has not been set for interactive
                                  client tool "${this.toolConfig.id}". Please set the "${prop.desc}" attribute of the
                                  "inputs" tab accordingly.`,
                    });

                    this.clientToolConfigValid = false;
                    this.clientToolConfigInspected = true;
                    return;
                }
            }
            console.log("finished validation");
            this.clientToolConfigValid = true;
            this.clientToolConfigInspected = true;
        },
        async fetchEntryPoint() {
            if (this.clientToolConfigValid) {
                const jobStore = useJobStore();
                const serviceToolEntryPoints = [];

                for (const entryPoint of this.activeEntryPoints) {
                    await jobStore.fetchJob(entryPoint.job_id);
                    const creatingJob = jobStore.getJob(entryPoint.job_id);

                    if (
                        creatingJob.tool_id === this.serviceToolId &&
                        creatingJob.tool_version === this.serviceToolVersion
                    ) {
                        serviceToolEntryPoints.push(entryPoint);
                    }
                }

                for (const entryPoint of serviceToolEntryPoints) {
                    if (entryPoint.label === this.serviceToolEntryPointLabel) {
                        this.onSetError(null);
                        this.entryPoint = entryPoint;
                        this.allServiceToolEntryPoints = serviceToolEntryPoints;
                        return;
                    }
                }
                this.entryPoint = null;
                await this.setNotRunningError();
            }
        },
        async setNotRunningError() {
            await this.fetchServiceToolConfig();
            if (this.serviceToolExists) {
                this.onSetError({
                    message: `Required interactive server tool "${this.serviceToolId}" with version
                              "${this.serviceToolVersion}" not running. Click the "Start Service" button
                              at the right end of the tool header to start.`,
                });
            }
        },
        async fetchServiceToolConfig() {
            if (!this.triedToFetchServiceTool) {
                try {
                    this.serviceToolConfig = await getToolFormData(this.serviceToolId, this.serviceToolVersion);
                } catch (error) {
                    this.onSetError({
                        message: `Loading interactive client tool "${this.toolConfig.id}" failed due to an error loading
                              required interactive server tool "${this.serviceToolId}" with version
                              "${this.serviceToolVersion}": ${error}`,
                    });
                }
            }
            this.triedToFetchServiceTool = true;
        },
        async waitForPageLoad() {
            while (this.serviceStarting) {
                try {
                    console.log("trying to load page...");
                    await axios.get(withPrefix(this.entryPointTarget));
                    this.serviceStarting = false;
                } catch (error) {
                    console.log(error);
                    await new Promise((resolve) => setTimeout(resolve, 500));
                }
            }
        },
        openServiceTool() {
            this.$router.push(`/root?tool_id=${this.serviceToolId}&tool_version=${this.serviceToolVersion}`);
        },
        async startService() {
            this.serviceStarting = true;

            const jobDef = {
                tool_id: this.serviceToolId,
                tool_version: this.serviceToolVersion,
            };

            try {
                const jobResponse = await submitJob(jobDef);
                refreshContentsWrapper();
                const nJobs = jobResponse && jobResponse.jobs ? jobResponse.jobs.length : 0;
                if (nJobs === 0) {
                    this.onSetError({
                        dialog: true,
                        message: "",
                        title: "Job submission rejected.",
                        content: jobResponse,
                    });
                    this.serviceStarting = false;
                }
            } catch (error) {
                this.onSetError({
                    dialog: true,
                    message: error,
                    title: "Job submission failed.",
                    content: jobDef,
                });
                this.serviceStarting = false;
            }
        },
        async stopService() {
            this.serviceStopping = true;
            for (const entryPoint of this.allServiceToolEntryPoints) {
                await this.interactiveToolsServices.stopInteractiveTool(entryPoint.id);
            }
            this.entryPoint = null;
            this.allServiceToolEntryPoints = [];

            await this.fetchEntryPoints();
            await this.ensurePollingEntryPoints();

            this.serviceStopping = false;
        },
        getIframeObj() {
            return window.top.document.getElementById("interactive_client_frame");
        },
        // reloadIframe() {
        //     this.getIframeObj().src += "";
        // },
        onLoadIframe() {
            console.log("onLoadIframe");
            this.registerIframeEventHandlers();
            this.iframeLoaded = true;
            this.refreshIframeHeight();
            this.fetchServiceToolConfig().then(() => {});
        },
        registerIframeEventHandlers() {
            const iframeObj = this.getIframeObj();
            if (iframeObj) {
                const win = iframeObj.contentWindow;

                win.addEventListener("resize", this.refreshIframeHeight);
                win.addEventListener("mousedown", this.shortSleepAndRefreshIframeHeight);
                win.addEventListener("mouseup", this.shortSleepAndRefreshIframeHeight);
                win.addEventListener("keydown", this.shortSleepAndRefreshIframeHeight);
            }
        },
        async refreshIframeHeight(event) {
            this.refreshIframeHeightFlip = !this.refreshIframeHeightFlip;
        },
        async shortSleepAndRefreshIframeHeight(event) {
            await new Promise((resolve) => setTimeout(resolve, 1));
            this.refreshIframeHeight();
        },
    },
};
</script>
