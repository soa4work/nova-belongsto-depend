<template>
    <default-field :field="field" v-if="showSelect" :show-help-text="showHelpText">
        <template slot="field">
            <div class="flex items-center">
                <multiselect
                    v-model="value"
                    :disabled="creatingViaRelatedResource"
                    :options="options"
                    :placeholder="this.field.placeholder ? this.field.placeholder : this.field.indexName + ' ' + __('Select')"
                    :selectLabel="__('Press enter to select')"
                    :selectedLabel="__('Selected')"
                    :deselectLabel="__('Press enter to remove')"
                    :custom-label="customLabel"
                    :open-direction="this.field.openDirection ? this.field.openDirection : ''"
                    @input="onChange"
                >
                    <span slot="noResult">{{ __("Oops! No elements found. Consider changing the search query.") }}</span>

                    <span slot="noOptions">{{ __("List is empty") }}</span>
                </multiselect>

                <create-relation-button v-if="canShowNewRelationModal" @click="openRelationModal" class="ml-1" :dusk="`${this.field.attribute}-inline-create`" />
            </div>

            <portal to="modals" transition="fade-transition">
                <create-relation-modal
                    v-if="relationModalOpen && canShowNewRelationModal"
                    @set-resource="handleSetResource"
                    @cancelled-create="closeRelationModal"
                    :resource-name="field.resourceName"
                    :resource-id="resourceId"
                    :via-relationship="viaRelationship"
                    :via-resource="viaResource"
                    :via-resource-id="viaResourceId"
                    width="800"
                />
            </portal>

            <div v-if="hasError" class="help-text error-text mt-2 text-danger">
                {{ firstError }}
            </div>
        </template>
    </default-field>
</template>

<script>
import { FormField, HandlesValidationErrors } from "laravel-nova";
import Multiselect from "vue-multiselect";
import _ from "lodash";

export default {
    components: { Multiselect },
    mixins: [FormField, HandlesValidationErrors],

    props: {
        resourceName: String,
        field: Object,
        resourceId: {},
        viaResource: {},
        viaResourceId: {},
        viaRelationship: {},
        relationshipType: {},
    },

    data() {
        return {
            options: [],
            relationModalOpen: false,
            selectedResourceId: null,
        };
    },

    computed: {
        /** Determine if we are editing an existing resource */
        editingExistingResource() {
            return Boolean(this.field.belongsToId);
        },

        /** Determine if we are creating a new resource via a parent relation */
        creatingViaRelatedResource() {
            return this.viaResource == this.field.resourceName && typeof this.viaResourceId !== "undefined" && this.field.reverse;
        },

        showSelect() {
            return !this.field.fallback || this.options.length !== 0;
        },

        showFallback() {
            return this.field.fallback && this.options.length === 0;
        },

        isLocked() {
            return this.viaResource == this.field.resourceName && this.field.reverse;
        },

        isReadonly() {
            return this.field.readonly || _.get(this.field, "extraAttributes.readonly");
        },

        canShowNewRelationModal() {
            return this.field.showCreateRelationButton && !this.shownViaNewRelationModal && !this.isLocked && !this.isReadonly && this.authorizedToCreate;
        },

        authorizedToCreate() {
            return _.find(Nova.config.resources, (resource) => {
                return resource.uriKey == this.field.resourceName;
            }).authorizedToCreate;
        },
    },

    beforeDestroy() {
        if (this.field.dependsOn) {

            if (!(this.field.dependsOn instanceof Array)) {
                throw new TypeError('dependsOn must be an array. Please use NovaBelongsToDepend::dependsOn Method')
            }

            let $busEvents = [];

            for (let i = 0; i < this.field.dependsOn.length; i++) {
                $busEvents.push("nova-belongsto-depend-" + this.field.dependsOn[i]);
            }

            Nova.$off($busEvents);
        }
    },

    created() {
        if (this.field.dependsOn) {

            if (!(this.field.dependsOn instanceof Array)) {
                throw new TypeError('dependsOn must be an array. Please use NovaBelongsToDepend::dependsOn Method')
            }

            let $busEvents = [];

            for (let i = 0; i < this.field.dependsOn.length; i++) {
                $busEvents.push("nova-belongsto-depend-" + this.field.dependsOn[i]);
            }

            Nova.$on($busEvents, async (dependsOnValue) => {
                this.value = "";

                Nova.$emit("nova-belongsto-depend-" + this.field.attribute.toLowerCase(), {
                    value: this.value,
                    field: this.field,
                });

                if (dependsOnValue && dependsOnValue.value) {
                    this.updateDependsMap(dependsOnValue);

                    this.options = (
                        await Nova.request().post("/nova-vendor/nova-belongsto-depend", {
                            resource: this.resourceName,
                            resourceId: this.resourceId,
                            viaResource: this.viaResource,
                            viaResourceId: this.viaResourceId,
                            viaRelationship: this.viaRelationship,
                            relationshipType: this.relationshipType,
                            relatedResource: this.viaResource,
                            modelClass: dependsOnValue.field.modelClass,
                            attribute: this.field.attribute,
                            dependsMap: this.field.dependsMap
                        })
                    ).data;

                    if (this.field.valueKey) {
                        this.value = this.options.find((item) => item[this.field.modelPrimaryKey] == this.field.valueKey);
                        Nova.$emit("nova-belongsto-depend-" + this.field.attribute.toLowerCase(), {
                            value: this.value,
                            field: this.field,
                        });
                    }
                } else {
                    this.cleanupDependsMap(dependsOnValue);
                }
            });
        }
    },

    methods: {
        updateDependsMap(dependsOnValue) {
            let exists = false;
            let index = -1;

            for (let i = 0; i < this.field.dependsMap.length; i++) {
                if (this.field.dependsMap[i].key === dependsOnValue.field.modelClass) {

                    exists = true;
                    index = i;
                    break;
                }
            }

            if (exists) {
                this.field.dependsMap[index].value = dependsOnValue.value[dependsOnValue.field.modelPrimaryKey];
            } else {
                this.field.dependsMap.push({
                    key: dependsOnValue.field.modelClass,
                    value: dependsOnValue.value[dependsOnValue.field.modelPrimaryKey]
                });
            }
        },

        cleanupDependsMap(dependsOnValue) {
            for (let i = 0; i < this.field.dependsMap.length; i++) {
                if (this.field.dependsMap[i].key === dependsOnValue.field.modelClass) {
                    this.field.dependsMap.splice(i, 1);
                }
            }
        },

        customLabel(item) {
            return item[this.field.titleKey];
        },

        /*
         * Set the initial, internal value for the field.
         */
        setInitialValue() {
            this.options = this.field.options;

            if (this.field.value) {
                this.value = this.options.find((item) => item[this.field.modelPrimaryKey] == this.field.valueKey);
            } else if (this.creatingViaRelatedResource) {
                this.value = this.options.find((item) => item[this.field.modelPrimaryKey] == this.viaResourceId);
            }

            if (this.value) {
                Nova.$emit("nova-belongsto-depend-" + this.field.attribute.toLowerCase(), {
                    value: this.value,
                    field: this.field,
                });
            }
        },

        /**
         * Fill the given FormData object with the field's internal value.
         */
        fill(formData) {
            if (this.value) {
                formData.append(this.field.attribute, this.value[this.field.modelPrimaryKey] || "");
            } else {
                if (this.field.fallback) {
                    formData.append(this.field.fallback.attribute, this.$refs["field-" + this.field.fallback.attribute].value);
                } else {
                    formData.append(this.field.attribute, "");
                }
            }
        },

        /**
         * Update the field's internal value.
         */
        handleChange(value) {
            this.value = value;
        },

        async onChange(value) {
            Nova.$emit("nova-belongsto-depend-" + this.field.attribute.toLowerCase(), {
                value,
                field: this.field,
            });
        },

        openRelationModal() {
            this.relationModalOpen = true;
        },

        closeRelationModal() {
            this.relationModalOpen = false;
        },

        handleSetResource({ id }) {
            this.closeRelationModal();
            this.selectedResourceId = id;
            this.getAvailableResources().then(() => this.selectInitialResource());
        },
    },
};
</script>

<style src="vue-multiselect/dist/vue-multiselect.min.css"></style>
<style>
.card.overflow-hidden {
    overflow: visible !important;
}

.multiselect {
    box-shadow: 0 2px 4px 0 rgba(0, 0, 0, 0.05);
    min-height: 36px !important;
    border-radius: 0.5rem;
}

.multiselect__tags {
    min-height: 36px !important;
    border: 1px solid var(--60) !important;
    color: var(--80);
    border-radius: 0.5rem !important;
}

.multiselect__select {
    background-repeat: no-repeat;
    background-size: 10px 6px;
    background-position: center right 0.75rem;
    background-image: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 10 6"><path fill="#35393C" fill-rule="nonzero" d="M8.293.293a1 1 0 0 1 1.414 1.414l-4 4a1 1 0 0 1-1.414 0l-4-4A1 1 0 0 1 1.707.293L5 3.586 8.293.293z"/></svg>');
}

.multiselect__select:before {
    content: none !important;
}
</style>
