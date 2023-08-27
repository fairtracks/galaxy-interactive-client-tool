<template>
    <div v-if="currentUser && currentHistoryId">
        <ToolForm
            :form-config-prop="formConfig"
            :current-version="currentVersion"
            :show-loading="showLoading"
            :show-form-prop="showForm"
            :disabled-prop="disabled"
            :message-show="messageShow"
            :message-variant="messageVariant"
            :message-text="messageText"
            @onChangeVersion="onChangeVersion" />
    </div>
</template>

<script>
import { mapState } from "pinia";

import { useHistoryStore } from "@/stores/historyStore";
import { useUserStore } from "@/stores/userStore";

import { getToolFormData } from "./services";

import ToolForm from "./ToolForm.vue";

export default {
    components: {
        ToolForm,
    },
    props: {
        id: {
            type: String,
            default: null,
        },
        version: {
            type: String,
            default: null,
        },
        job_id: {
            type: String,
            default: null,
        },
        history_id: {
            type: String,
            default: null,
        },
    },
    data() {
        return {
            disabled: false,
            showLoading: true,
            showForm: false,
            formConfig: {},
            messageShow: false,
            messageVariant: "",
            messageText: "",
            currentVersion: this.version,
        };
    },
    computed: {
        ...mapState(useUserStore, ["currentUser"]),
        ...mapState(useHistoryStore, ["currentHistoryId"]),
    },
    created() {
        this.requestTool();
    },
    methods: {
        onChangeVersion(newVersion) {
            this.requestTool(newVersion);
        },
        requestTool(newVersion) {
            this.currentVersion = newVersion || this.currentVersion;
            this.disabled = true;
            console.debug("ToolForm - Requesting tool.", this.id);
            return getToolFormData(this.id, this.currentVersion, this.job_id, this.history_id)
                .then((data) => {
                    if (data.model_class == "InteractiveClientTool") {
                        console.log(data);
                        //galaxy_main.location = Galaxy.root + 'tool_runner/rerun?job_id=' + this.job_id;
                    }
                    this.formConfig = data;
                    // this.remapAllowed = this.job_id && data.job_remap;
                    this.showForm = true;
                    this.messageShow = false;
                    if (newVersion) {
                        this.messageVariant = "success";
                        this.messageText = `Now you are using '${data.name}' version ${data.version}, id '${data.id}'.`;
                    }
                })
                .catch((error) => {
                    this.messageVariant = "danger";
                    this.messageText = `Loading tool ${this.id} failed: ${error}`;
                    this.messageShow = true;
                })
                .finally(() => {
                    this.disabled = false;
                    this.showLoading = false;
                });
        },
    },
};
</script>
