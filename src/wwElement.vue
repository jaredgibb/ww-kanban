<template>
    <div class="ww-kanban" :style="kanbanStyle">
        <template v-if="content.uncategorizedStack">
            <wwLayoutItemContext :index="0" :item="null" :data="uncategorizedStack" is-repeat>
                <wwElement
                    v-bind="content.stackElement"
                    :ww-props="{
                        ...stackConfig,
                        items: uncategorizedStack.items,
                        stack: null,
                    }"
                    class="ww-kanban-stack"
                    :states="isDragging ? ['dragging'] : []"
                ></wwElement>
            </wwLayoutItemContext>
        </template>

        <template v-for="(stack, index) in internalStacks" :key="'ww-stack-' + index">
            <wwLayoutItemContext :index="index" :item="null" is-repeat :data="stack" :repeated-items="internalStacks">
                <wwElement
                    v-bind="content.stackElement"
                    :ww-props="{ ...stackConfig, items: stack.items, stack: stack.value }"
                    class="ww-kanban-stack"
                    :states="isDragging ? ['dragging'] : []"
                ></wwElement>
            </wwLayoutItemContext>
        </template>
    </div>
</template>

<script>
import { provide, reactive, ref, watch, computed } from "vue";

export default {
    props: {
        content: { type: Object, required: true },
        uid: { type: String, required: true },
        /* wwEditor:start */
        wwElementState: { type: Object, required: true },
        wwEditorState: { type: Object, required: true },
        /* wwEditor:end */
    },
    emits: ["trigger-event", "update:content:effect"],
    setup(props, { emit }) {
        const internalStacks = ref([]);
        const uncategorizedStack = reactive({
            label: "Uncategorized",
            value: null,
            items: [],
        });
        const crossColumnStackCache = new Map();
        const crossColumnRemovedMetaCache = new Map();

        function cloneItem(item) {
            if (typeof structuredClone === "function") {
                try {
                    return structuredClone(item);
                } catch (error) {
                    // Fall back to recursive cloning for values structuredClone cannot handle.
                }
            }

            if (Array.isArray(item)) {
                return item.map((value) => cloneItem(value));
            }

            if (item && typeof item === "object") {
                return Object.keys(item).reduce((result, key) => {
                    result[key] = cloneItem(item[key]);
                    return result;
                }, {});
            }

            return item;
        }

        function setByPath(target, path, value) {
            if (!path || !target || typeof target !== "object") return;

            const segments = String(path)
                .match(/[^.[\]]+/g)
                ?.map((segment) => {
                    const trimmed = segment.trim();
                    const isQuoted =
                        (trimmed.startsWith("'") && trimmed.endsWith("'")) ||
                        (trimmed.startsWith('"') && trimmed.endsWith('"'));

                    return isQuoted ? trimmed.slice(1, -1) : trimmed;
                });
            if (!segments || !segments.length) return;

            let cursor = target;
            for (let i = 0; i < segments.length - 1; i++) {
                const segment = segments[i];
                const nextSegment = segments[i + 1];

                if (cursor[segment] === null || typeof cursor[segment] !== "object") {
                    cursor[segment] = /^\d+$/.test(nextSegment) ? [] : {};
                }

                cursor = cursor[segment];
            }

            cursor[segments[segments.length - 1]] = value;
        }

        function isSameItem(firstItem, secondItem) {
            if (firstItem === secondItem) return true;
            if (!firstItem || !secondItem) return false;

            if (props.content.itemKey) {
                const firstKey = wwLib.resolveObjectPropertyPath(firstItem, props.content.itemKey);
                const secondKey = wwLib.resolveObjectPropertyPath(secondItem, props.content.itemKey);

                if (firstKey !== undefined && secondKey !== undefined) {
                    return firstKey === secondKey;
                }
            }

            // Pragmatic fallback: many datasets have a stable `id` even if itemKey is not configured.
            if (firstItem.id !== undefined && secondItem.id !== undefined) {
                return firstItem.id === secondItem.id;
            }

            return false;
        }

        function removeMatchingItem(items, itemToRemove) {
            if (!Array.isArray(items)) return [];

            const index = items.findIndex((item) => isSameItem(item, itemToRemove));
            if (index === -1) return items.slice();

            return [...items.slice(0, index), ...items.slice(index + 1)];
        }

        function removeAllMatchingItems(items, itemToRemove) {
            if (!Array.isArray(items)) return [];
            return items.filter((item) => !isSameItem(item, itemToRemove));
        }

        function insertItemAtIndex(items, itemToInsert, index) {
            const list = Array.isArray(items) ? items.slice() : [];
            const normalizedIndex = Number.isInteger(index) ? Math.max(0, Math.min(index, list.length)) : list.length;

            list.splice(normalizedIndex, 0, itemToInsert);
            return list;
        }

        function getStackSnapshotMap() {
            const stackMap = new Map();

            stackMap.set(null, Array.isArray(uncategorizedStack.items) ? uncategorizedStack.items.slice() : []);

            for (const stack of internalStacks.value || []) {
                stackMap.set(stack.value, Array.isArray(stack.items) ? stack.items.slice() : []);
            }

            return stackMap;
        }

        function isDefinedStackValue(stackValue) {
            return (internalStacks.value || []).some((stack) => stack.value === stackValue);
        }

        function getBucketKeyFromStackedValue(stackValue) {
            return isDefinedStackValue(stackValue) ? stackValue : null;
        }

        function findRemovedMetaForItem(movedItem) {
            for (const [stackValue, meta] of crossColumnRemovedMetaCache.entries()) {
                if (meta && isSameItem(meta.item, movedItem)) {
                    return { stackValue, ...meta };
                }
            }

            return null;
        }

        function buildFinalBoardGroups({
            stackValue,
            updatedStackItems,
            movedItem,
            fromStackedValue,
            sourceBucketKey: explicitSourceBucketKey,
            isCrossColumnAdd,
            newIndex,
        }) {
            const stackMap = getStackSnapshotMap();

            if (isCrossColumnAdd) {
                const sourceBucketKey =
                    explicitSourceBucketKey !== undefined
                        ? explicitSourceBucketKey
                        : getBucketKeyFromStackedValue(fromStackedValue);
                const cachedSourceItems = crossColumnStackCache.get(sourceBucketKey);

                if (sourceBucketKey !== stackValue && Array.isArray(cachedSourceItems)) {
                    stackMap.set(sourceBucketKey, removeAllMatchingItems(cachedSourceItems, movedItem));
                } else if (sourceBucketKey !== stackValue) {
                    stackMap.set(sourceBucketKey, removeAllMatchingItems(stackMap.get(sourceBucketKey), movedItem));
                }

                const destinationItems = removeAllMatchingItems(stackMap.get(stackValue), movedItem);
                stackMap.set(stackValue, insertItemAtIndex(destinationItems, movedItem, newIndex));
            } else {
                const reorderedItems = removeAllMatchingItems(stackMap.get(stackValue), movedItem);
                stackMap.set(stackValue, insertItemAtIndex(reorderedItems, movedItem, newIndex));
            }

            return {
                uncategorizedItems: Array.isArray(stackMap.get(null)) ? stackMap.get(null).slice() : [],
                stacks: (internalStacks.value || []).map((stack) => ({
                    value: stack.value,
                    items: Array.isArray(stackMap.get(stack.value)) ? stackMap.get(stack.value).slice() : [],
                })),
            };
        }

        function buildUpdatedFullList({
            stacks,
            uncategorizedItems,
            movedItem,
            normalizeMovedIntoUncategorized = false,
        }) {
            const updatedFullList = [];
            const stackedByPath = props.content.stackedBy;
            const sortedByPath = props.content.sortedBy;

            const pushBucket = (items, { stackValue, isUncategorized }) => {
                if (!Array.isArray(items)) return;

                items.forEach((item, index) => {
                    const clonedItem = cloneItem(item);

                    if (stackedByPath) {
                        if (isUncategorized) {
                            if (normalizeMovedIntoUncategorized && movedItem && isSameItem(item, movedItem)) {
                                setByPath(clonedItem, stackedByPath, null);
                            }
                        } else {
                            setByPath(clonedItem, stackedByPath, stackValue);
                        }
                    }

                    if (sortedByPath) {
                        setByPath(clonedItem, sortedByPath, index);
                    }

                    updatedFullList.push(clonedItem);
                });
            };

            if (props.content.uncategorizedStack) {
                pushBucket(uncategorizedItems, { stackValue: null, isUncategorized: true });
            }

            for (const stack of stacks || []) {
                pushBucket(stack.items, { stackValue: stack.value, isUncategorized: false });
            }

            if (!props.content.uncategorizedStack) {
                pushBucket(uncategorizedItems, { stackValue: null, isUncategorized: true });
            }

            return updatedFullList;
        }

        function emitItemMovedEvent({
            item,
            from,
            to,
            oldIndex,
            newIndex,
            updatedList,
            fullListContext,
        }) {
            emit("trigger-event", {
                name: "item:moved",
                event: {
                    item,
                    from,
                    to,
                    oldIndex,
                    newIndex,
                    updatedList,
                    updatedFullList: buildUpdatedFullList(fullListContext),
                },
            });
        }

        function clearCrossColumnCaches() {
            crossColumnStackCache.clear();
            crossColumnRemovedMetaCache.clear();
        }

        provide("customHandler", (change, { stack: stackValue, updatedStackItems }) => {
            if (change.removed) {
                crossColumnStackCache.set(stackValue, Array.isArray(updatedStackItems) ? updatedStackItems.slice() : []);
                crossColumnRemovedMetaCache.set(stackValue, {
                    item: change.removed.element,
                    oldIndex: change.removed.oldIndex,
                });
            }

            if (change.moved) {
                const fullListContext = buildFinalBoardGroups({
                    stackValue,
                    updatedStackItems,
                    movedItem: change.moved.element,
                    fromStackedValue: stackValue,
                    isCrossColumnAdd: false,
                    newIndex: change.moved.newIndex,
                });

                emitItemMovedEvent({
                    item: change.moved.element,
                    from: stackValue,
                    to: stackValue,
                    oldIndex: change.moved.oldIndex,
                    newIndex: change.moved.newIndex,
                    updatedList: updatedStackItems,
                    fullListContext: {
                        ...fullListContext,
                        movedItem: change.moved.element,
                        normalizeMovedIntoUncategorized: false,
                    },
                });
                clearCrossColumnCaches();
                return;
            }

            if (change.added) {
                const fromFromItem = wwLib.resolveObjectPropertyPath(change.added.element, props.content.stackedBy);
                const removedMetaMatch = findRemovedMetaForItem(change.added.element);
                const sourceBucketKey =
                    removedMetaMatch?.stackValue !== undefined
                        ? removedMetaMatch.stackValue
                        : getBucketKeyFromStackedValue(fromFromItem);
                const from = removedMetaMatch ? removedMetaMatch.stackValue : fromFromItem;
                const removedMeta = removedMetaMatch || crossColumnRemovedMetaCache.get(sourceBucketKey);
                const crossColumnOldIndex =
                    removedMeta && isSameItem(removedMeta.item, change.added.element)
                        ? removedMeta.oldIndex
                        : change.added.oldIndex ?? null;
                const fullListContext = buildFinalBoardGroups({
                    stackValue,
                    updatedStackItems,
                    movedItem: change.added.element,
                    fromStackedValue: from,
                    sourceBucketKey,
                    isCrossColumnAdd: true,
                    newIndex: change.added.newIndex,
                });

                emitItemMovedEvent({
                    item: change.added.element,
                    from,
                    to: stackValue,
                    oldIndex: crossColumnOldIndex,
                    newIndex: change.added.newIndex,
                    updatedList: updatedStackItems,
                    fullListContext: {
                        ...fullListContext,
                        movedItem: change.added.element,
                        normalizeMovedIntoUncategorized: stackValue === null,
                    },
                });
                clearCrossColumnCaches();
            }
        });

        const isDraggingManager = reactive({});
        provide("customDragHandler", (isDragging, { stack }) => (isDraggingManager[stack] = isDragging));

        const { setValue: setDrag } = wwLib.wwVariable.useComponentVariable({
            uid: props.uid,
            name: "isDragging",
            type: "boolean",
            defaultValue: false,
            readonly: true,
        });
        watch(
            isDraggingManager,
            (value) => {
                setDrag(Object.values(value).some((isDragging) => isDragging));
            },
            { deep: true }
        );

        const isDragging = computed(() => Object.values(isDraggingManager).some((isDragging) => isDragging));

        const css = computed(() => `* { cursor: ${props.content.draggingCursor || "grabbing"} !important; }`);
        const styletag = wwLib.getFrontDocument().createElement("style");

        watch(
            isDragging,
            (value) => {
                if (value) {
                    styletag.appendChild(wwLib.getFrontDocument().createTextNode(css.value));
                    wwLib.getFrontDocument().body.appendChild(styletag);
                } else {
                    styletag.remove();
                }
            },
            { deep: true }
        );

        return { internalStacks, uncategorizedStack };
    },
    computed: {
        stacks() {
            const stacks = wwLib.wwCollection.getCollectionData(this.content.stacks);
            if (!Array.isArray(stacks)) return [];
            return stacks;
        },
        items() {
            const items = wwLib.wwCollection.getCollectionData(this.content.items);
            if (!Array.isArray(items)) return [];
            return items;
        },
        stackConfig() {
            return {
                sortable: this.content.sortable,
                group: "kanban-" + this.uid,
                itemKey: this.content.itemKey,
                handle: this.content.customDragHandle ? this.content.handleClass || "draggable" : null,
                readonly: this.content.readonly,
            };
        },
        kanbanStyle() {
            return {
                "--wrap-stacks": this.content.wrapStacks ? "wrap" : "nowrap",
            };
        },
        isReadonly() {
            /* wwEditor:start */
            if (this.wwEditorState.isSelected) {
                return this.wwElementState.states.includes("readonly");
            }
            /* wwEditor:end */
            return this.content.readonly;
        },
    },
    watch: {
        "content.stackValue"() {
            this.refreshStacks();
        },
        "content.stackedBy"() {
            this.refreshStacks();
        },
        "content.sortedBy"() {
            this.refreshStacks();
        },
        "content.sortOrder"() {
            this.refreshStacks();
        },
        "content.stacks": {
            handler() {
                this.refreshStacks();
            },
            deep: true,
        },
        stacks() {
            this.refreshStacks();
        },
        items: {
            handler() {
                this.refreshStacks();
            },
            deep: true,
        },
        isReadonly: {
            immediate: true,
            handler(value) {
                if (value) {
                    this.$emit("add-state", "readonly");
                } else {
                    this.$emit("remove-state", "readonly");
                }
            },
        },
    },
    methods: {
        refreshStacks() {
            this.internalStacks = this.stacks
                .map((stack) => ({
                    label: wwLib.resolveObjectPropertyPath(stack, this.content.stackLabel || "label") ?? "",
                    value: wwLib.resolveObjectPropertyPath(stack, this.content.stackValue || "value") ?? "",
                }))
                .map((stack) => ({
                    ...stack,
                    items: this.items
                        .filter((item) => wwLib.resolveObjectPropertyPath(item, this.content.stackedBy) === stack.value)
                        .sort((a, b) => {
                            if (!this.content.sortedBy) return 0;
                            const valueA = wwLib.resolveObjectPropertyPath(a, this.content.sortedBy);
                            const valueB = wwLib.resolveObjectPropertyPath(b, this.content.sortedBy);
                            if (this.content.sortOrder === "asc") {
                                return valueA > valueB ? 1 : -1;
                            } else {
                                return valueA > valueB ? -1 : 1;
                            }
                        }),
                }));
            const stacksList = this.stacks.map((stack) =>
                wwLib.resolveObjectPropertyPath(stack, this.content.stackValue || "value")
            );
            this.uncategorizedStack.items = this.items.filter(
                (item) => !stacksList.includes(wwLib.resolveObjectPropertyPath(item, this.content.stackedBy))
            );
        },
        /* wwEditor:start */
        getTestEvent() {
            if (!this.internalStacks.length) throw new Error("No stack found");
            if (!this.items.length) throw new Error("No item found");
            return {
                item: this.items[0],
                from: this.internalStacks[0].value,
                to: this.internalStacks[0].value,
                oldIndex: 0,
                newIndex: 1,
                updatedList: this.items,
                updatedFullList: this.items,
            };
        },
        /* wwEditor:end */
    },
    created() {
        this.refreshStacks();
    },
};
</script>

<style lang="scss" scoped>
.ww-kanban {
    flex-direction: row;
    flex-wrap: var(--wrap-stacks);
}
</style>
