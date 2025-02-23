<template>

    <div class="tree-view">

        <context-menu v-if="contextMenu" :contextMenuItems="contextMenuItems"></context-menu>

        <drop-between-zone
                @nodeDrop="dropNodeAtPosition(0)"
                v-if="draggedNode !== null && draggedNode.data !== data[0]">
        </drop-between-zone>
        <template v-for="(nodeData, index) in data">
            <tree-node
                    :key="nodeData[nodeKeyProp]"
                    :keyProp="nodeKeyProp"
                    :renameOnDblClick="renameNodeOnDblClick"
                    :childrenProp="nodeChildrenProp"
                    :labelProp="nodeLabelProp"
                    :data="nodeData"
                    :draggable="nodesDraggable"
                    :defaultIconClass="defaultIconClass"
                    :iconClassProp="iconClassProp"
                    :showIcon="showIcons"
                    :prependIconClass="prependIconClass"
                    :contextMenu="contextMenu"
                    ref="rootNodes"
                    @node-select="nodeSelect"
                    @node-drag-start="nodeDragStart"
                    @delete-node="deleteNode">
            </tree-node>
            <drop-between-zone
                    @nodeDrop="dropNodeAtPosition(index + 1)"
                    v-if="draggedNode !== null && draggedNode.data !== nodeData && (index + 1 >= data.length || draggedNode.data !== data[index + 1])">
            </drop-between-zone>
        </template>

    </div>

</template>

<script>

    import TreeNode from './TreeNode.vue';
    import EventBus from '../EventBus.js';
    import DropBetweenZone from './DropBetweenZone.vue'
    import ContextMenu from './ContextMenu.vue'

    export default {
        props: {
            data: {
                type: Array,
                required: true
            },
            allowMultiple: {
                type: Boolean,
                default: false
            },
            nodeKeyProp: {
                type: String,
                default: 'id'
            },
            nodeChildrenProp: {
                type: String,
                default: 'children'
            },
            nodeLabelProp: {
                type: String,
                default: 'name'
            },
            nodesDraggable: {
                type: Boolean,
                default: false
            },
            contextMenu: {
                type: Boolean,
                default: true
            },
            contextMenuItems: {
                type: Array,
                default: [{code: 'DELETE_NODE', label: 'Delete node'}, {code: 'RENAME_NODE', label: 'Rename node'}]
            },
            renameNodeOnDblClick: {
                type: Boolean,
                default: true
            },
            // class added to every icon no matter what
            prependIconClass: {
                type: String,
                default: null
            },
            // default icon if node icon is not specified
            defaultIconClass: {
                type: String,
                default: null
            },
            // where to search for node icon
            iconClassProp: {
                type: String,
                default: "icon"
            },
            // show icons
            showIcons: {
                type: Boolean,
                default: false
            }
        },
        data() {
            return {
                draggedNode: null
            }
        },
        components: {
            TreeNode,
            DropBetweenZone,
            ContextMenu
        },
        methods: {
            createNodeMap() {
                if (this.nodeMap === undefined) {
                    let nodeMap = this.nodeMap = new Map()

                    let nodes = this.$refs.rootNodes.slice()

                    for (let i = 0; i < nodes.length; i++) {
                        let tmpNode = nodes[i]
                        nodes.push(...tmpNode.getChildNodes())
                    }

                    for (let tmpNode of nodes) {
                        nodeMap.set(tmpNode.data[this.nodeKeyProp], tmpNode) // TODO: change to getter
                    }
                }
            },
            getNodeByKey(key) {
                return this.nodeMap.get(key)
            },
            // event bubbles up to the roots
            nodeSelect(node, isSelected) {
                this.$emit('node-select', node, isSelected)
                if (isSelected) {
                    if (this.selectedNode !== null) {
                        this.selectedNode.deselect()
                    }
                    this.selectedNode = node
                } else if (node === this.selectedNode) {
                    this.selectedNode = null
                }
            },
            nodeDragStart() {
                EventBus.$on('drop-ok', this.cutNode)
            },
            cutNode() {
                EventBus.$off('drop-ok')
                let idx = this.data.indexOf(window._bTreeView.draggedNodeData)
                this.data.splice(idx, 1)
                // let's notify that node data was successfully cut (removed from array)
                EventBus.$emit('cut-ok')
            },
            draggingStarted(draggedNode) {
                this.draggedNode = draggedNode
                // let's listen for the drag end event
                EventBus.$on('node-drag-end', this.draggingEnded)
            },
            draggingEnded() {
                // stop listening for the dragging end event
                EventBus.$off('node-drag-end', this.draggingEnded)
                this.draggedNode = null
            },
            dropNodeAtPosition(pos) {
                // position can change if we move node within the same parent node (same level)
                // so it's better to remember node at previous position
                let insertAfter = pos - 1 < 0 ? null : this.data[pos - 1]
                EventBus.$on('cut-ok', () => {
                    let pos = this.data.indexOf(insertAfter) + 1
                    this.data.splice(pos, 0, window._bTreeView.draggedNodeData)
                    delete window._bTreeView.draggedNodeKey
                    delete window._bTreeView.draggedNodeData
                    EventBus.$off('cut-ok');
                })
                EventBus.$emit('drop-ok')
            },
            deleteNode(nodeData) {
                let nodes = this.data
                let idx = nodes.indexOf(nodeData)
                nodes.splice(idx, 1)
            },
            menuItemSelected(item, node) {
                switch (item.code) {
                    case 'DELETE_NODE':
                        node.delete()
                    case 'RENAME_NODE':
                        node.startRenaming()
                    default:
                        this.$emit('context-menu-item-select', item, node)
                }
            }
        },
        created() {
            this.selectedNode = null
            EventBus.$on('node-drag-start', this.draggingStarted)
            EventBus.$on('context-menu-item-select', this.menuItemSelected)
            this.$nextTick(() => {
                this.createNodeMap()
            })
        }
    }

</script>

<style>

    .tree-view {
        text-align: left;
    }

</style>
