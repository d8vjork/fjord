<template>
    <fj-base-field :field="field" :model="model">
        <template slot="title-right">
            <a href="#" @click="toggleExpand" v-if="!this.create">
                <fa-icon :icon="expandedAll ? 'angle-up' : 'angle-down'" />
                {{
                    __(
                        `crud.fields.blocks.${
                            expandedAll ? 'collapse_all' : 'expand_all'
                        }`
                    ).toLowerCase()
                }}
            </a>
        </template>
        <div class="form-control-expand" v-if="model.id">
            <div v-if="busy" class="d-flex justify-content-around">
                <fj-spinner />
            </div>
            <draggable
                v-model="sortableBlocks"
                @end="newOrder"
                handle=".fj-draggable__dragbar"
                tag="b-row"
                :class="{ 'mb-0': field.readonly }"
                v-else-if="sortableBlocks.length > 0"
            >
                <fj-field-repeatable
                    ref="block"
                    v-for="(block, index) in sortableBlocks"
                    :key="index"
                    :block="block"
                    :field="field"
                    :model="model"
                    :width="field.blockWidth"
                    :reload="reloadBlock"
                    :preview="field.repeatables[block.type].preview"
                    :fields="field.repeatables[block.type].form.fields"
                    :set-route-prefix="setFieldsRoutePrefixBlockId"
                    @deleteItem="deleteBlock(block)"
                />
            </draggable>

            <div v-else>
                <fj-field-alert-empty
                    :field="field"
                    :class="{ 'mb-0': field.readonly }"
                />
            </div>

            <fj-field-block-add-repeatable-buttons
                v-if="!field.readonly"
                :field="field"
                :model="model"
                :sortableBlocks="sortableBlocks"
                @newBlock="newBlock"
            />
        </div>
        <template v-else>
            <fj-field-alert-not-created :field="field" class="mb-0" />
        </template>
    </fj-base-field>
</template>

<script>
export default {
    name: 'FieldBlock',
    props: {
        field: {
            type: Object
        },
        model: {
            type: Object
        }
    },
    data() {
        return {
            sortableBlocks: [],
            busy: true,
            expandedAll: false
        };
    },
    beforeMount() {
        this.loadBlocks();

        Fjord.bus.$on('saved', () => {
            this.reloadBlocks();
        });
    },
    computed: {
        create() {
            return this.model.id === undefined;
        }
    },
    methods: {
        async reloadBlock(block) {
            let response = await axios.get(
                `${this.field.route_prefix}/block/${this.field.id}/${block.id}`
            );
            let newBlock = this.crud(response.data);
            for (let i in this.sortableBlocks) {
                let block = this.sortableBlocks[i];
                if (block.id == newBlock.id) {
                    this.sortableBlocks[i] = newBlock;
                }
            }
        },
        toggleExpand() {
            for (let i in this.$refs.block) {
                let block = this.$refs.block[i];
                block.$emit('expand', !this.expandedAll);
            }

            this.expandedAll = !this.expandedAll;
        },
        setFieldsRoutePrefixBlockId(fields, block) {
            for (let i in fields) {
                let field = fields[i];
                fields[i].route_prefix = field.route_prefix
                    .replace('{block_id}', block.id)
                    .replace('{id}', this.model.id);
                if (this.field.readonly) {
                    fields[i].readonly = true;
                }
            }
            return fields;
        },
        reloadBlocks() {
            this._loadBlocks();
        },
        async loadBlocks() {
            this.busy = true;
            await this._loadBlocks();
            this.busy = false;
        },
        async _loadBlocks() {
            if (this.create) {
                return;
            }
            let response = await axios.get(
                `${this.field.route_prefix}/block/${this.field.id}`
            );
            this.sortableBlocks = [];
            for (let i in response.data) {
                let block = response.data[i];
                this.newBlock(block, false);
            }
        },
        newBlock(block, open = true) {
            this.sortableBlocks.push(this.crud(block));
            if (open) {
                this.$nextTick(() => {
                    this.$refs.block[this.$refs.block.length - 1].$emit(
                        'expand',
                        true
                    );
                });
            }
        },
        async deleteBlock(block) {
            try {
                let response = await axios.delete(
                    `${this.field.route_prefix}/block/${block.field_id}/${block.id}`
                );
            } catch (e) {
                console.log(e);
                return;
            }
            this.sortableBlocks.splice(this.sortableBlocks.indexOf(block), 1);

            Fjord.bus.$emit('field:updated', 'block:deleted');

            this.$bvToast.toast(this.$t('fj.deleted_block'), {
                variant: 'success'
            });
        },
        async newOrder() {
            let payload = {
                ids: this.sortableBlocks.map(item => item.id)
            };
            try {
                let response = await axios.put(
                    `${this.field.route_prefix}/${this.field.id}/order`,
                    payload
                );
            } catch (e) {
                console.log(e);
                return;
            }

            Fjord.bus.$emit('field:updated', 'block:ordered');

            this.$bvToast.toast(this.$t('fj.order_changed'), {
                variant: 'success'
            });
        }
    }
};
</script>
