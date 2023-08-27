<template>
    <div>
        <b-alert :show="messageShow" :variant="messageVariant">
            {{ messageText }}
        </b-alert>
        <LoadingSpan v-if="showLoading" message="Loading Tool" />

        <b-modal v-model="showError" size="sm" :title="errorTitle | l" scrollable ok-only>
            <b-alert v-if="errorMessage" show variant="danger">
                {{ errorMessage }}
            </b-alert>
            <b-alert show variant="warning">
                The server could not complete this request. Please verify your parameter settings, retry submission and
                contact the Galaxy Team if this error persists. A transcript of the submitted data is shown below.
            </b-alert>
            <small class="text-muted">
                <pre>{{ errorContentPretty }}</pre>
            </small>
        </b-modal>
        <slot name="above-tool" />
        <ToolCard
            v-if="showForm"
            :id="formConfig.id"
            :version="formConfig.version"
            :title="formConfig.name"
            :description="formConfig.description"
            :options="formConfig"
            :disabled="disabled"
            :allow-object-store-selection="config.object_store_allows_id_selection"
            :preferred-object-store-id="preferredObjectStoreId"
            itemscope="itemscope"
            itemtype="https://schema.org/CreativeWork"
            @onChangeVersion="onChangeVersion"
            @onSetError="onSetError"
            @updatePreferredObjectStoreId="onUpdatePreferredObjectStoreId">
            <template v-slot:body>
                <FormMessage v-if="errorText" variant="danger" :message="errorText" :persistent="true" />
                <FormMessage :variant="messageVariant" :message="messageText" />
                <slot name="body" />
            </template>
            <template v-slot:header-buttons>
                <slot name="header-buttons" />
            </template>
            <template v-slot:buttons>
                <slot name="buttons" />
            </template>
        </ToolCard>
    </div>
</template>

<script>
import LoadingSpan from "components/LoadingSpan";

import { useConfig } from "@/composables/config";

import ToolCard from "./ToolCard";

import FormMessage from "@/components/Form/FormMessage.vue";

export default {
    components: {
        FormMessage,
        LoadingSpan,
        ToolCard,
    },
    props: {
        formConfig: {
            type: Object,
            required: true,
        },
        showLoading: {
            type: Boolean,
            required: true,
        },
        showForm: {
            type: Boolean,
            required: true,
        },
        messageShow: {
            type: Boolean,
            default: false,
        },
        messageVariant: {
            type: String,
            default: "info",
        },
        messageText: {
            type: String,
            default: "",
        },
        showErrorProp: {
            type: Boolean,
            default: false,
        },
        errorTitle: {
            type: String,
            default: null,
        },
        errorContent: {
            type: String,
            default: null,
        },
        errorMessage: {
            type: String,
            default: "",
        },
        disabled: {
            type: Boolean,
            default: false,
        },
        entryPoints: {
            type: Array,
            default: () => [],
        },
        preferredObjectStoreId: {
            type: String,
            default: "",
        },
    },

    setup() {
        const { config, isConfigLoaded } = useConfig(true);
        return { config, isConfigLoaded };
    },
    data() {
        return {
            showError: this.showErrorProp,
            errorText: null,
        };
    },
    computed: {
        errorContentPretty() {
            return JSON.stringify(this.errorContent, null, 4);
        },
    },
    watch: {
        showErrorProp: {
            immediate: true,
            handler(newShowErrorProp) {
                this.showError = newShowErrorProp;
            },
        },
        formConfig: {
            immediate: true,
            handler() {
                this.errorText = "";
            },
        },
    },
    methods: {
        onUpdatePreferredObjectStoreId(preferredObjectStoreId) {
            this.$emit("updatePreferredObjectStoreId", preferredObjectStoreId);
        },
        onChangeVersion(newVersion) {
            this.$emit("onChangeVersion", newVersion);
        },
        onSetError(errorText) {
            this.errorText = errorText;
        },
    },
};
</script>
