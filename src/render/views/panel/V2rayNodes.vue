<template>
  <div class="panel-nodes flex flex-column h-100">
    <Tree
      class="node-tree flex-1 px-1 bg-white"
      :class="{ 'empty-tree': !appConfig.configs.length }"
      empty-text="暂无节点，点击添加添加新节点"
      :enable-cancel-select="false"
      :data="groupedNodes"
      @on-select-change="onSelect"
      @on-dbclick-node="onNodeDBClick"
      ref="tree"
    ></Tree>
    <div class="flex mt-1 flex-jc-center">
      <ButtonGroup class="w-6r mr-1 flex-inline">
        <Button class="flex-1" type="primary" @click="create">添加</Button>
        <Dropdown trigger="click" placement="top-end">
          <Button class="ivu-dropdown-btn-trigger" type="primary" icon="md-arrow-dropdown"></Button>
          <DropdownMenu slot="list">
            <DropdownItem @click.native="copyFromClipboard">从剪切板导入</DropdownItem>
            <DropdownItem @click.native="toSubscribe">添加订阅地址</DropdownItem>
          </DropdownMenu>
        </Dropdown>
      </ButtonGroup>
      <Poptip v-if="selectedGroupName" confirm title="确定删除该分组下所有节点？" @on-ok="removeGroup">
        <Button class="w-6r" :disabled="disabled.remove">删除</Button>
      </Poptip>
      <Button v-else class="w-6r" type="primary" @click="applyNode">应用</Button>
    </div>
    <div class="flex mt-1 flex-jc-center">
      <Button class="w-6r mr-1" :disabled="disabled.remove" @click="remove">删除</Button>
      <Button class="w-6r mr-1" :disabled="disabled.up" @click="updown(1)">上移</Button>
      <Button class="w-6r" :disabled="disabled.down" @click="updown(-1)">下移</Button>
    </div>
  </div>
</template>

<script>
import { ipcRenderer } from 'electron'
import { mapState, mapGetters, mapMutations, mapActions } from 'vuex'
import { hideWindow } from '../../ipc'
import Config from '../../../shared/v2ray'
import { groupConfigs, clone } from '../../../shared/utils'
import { EVENT_CONFIG_COPY_CLIPBOARD } from '../../../shared/events'

// 避免因上/下移动分组/配置而导致index改变后选中项不是group的问题
let preventIndexAffect = false
export default {
  data () {
    return {
      buttonProps: {
        type: 'ghost',
        size: 'small',
      },
      // 记录ID
      selectedConfigId: this.$store.getters.selectedConfig ? this.$store.getters.selectedConfig.id : '',
      // 记录分组名
      selectedGroupName: '',
      selectedNode: {},
    }
  },
  mounted () {
    if (this.appConfig.configs.length && this.appConfig.index) {
      let tree = this.$el.getElementsByClassName('node-tree')[0]
      let top = tree.scrollHeight * (this.appConfig.index / this.appConfig.configs.length) - tree.clientHeight / 2
      tree.scrollTop = Math.round(top)
    }
  },
  computed: {
    ...mapState(['appConfig', 'editingConfig', 'editingGroup']),
    ...mapGetters(['selectedConfig', 'isEditingConfigUpdated']),
    // 配置节点
    configs () {
      if (this.appConfig && this.appConfig.configs && this.appConfig.configs.length) {
        return this.appConfig.configs.map(config => {
          return this.cloneConfig(config, this.selectedGroupName ? false : config.id === this.selectedConfigId)
        })
      }
      return []
    },
    // 分组后的配置节点
    groupedConfigs () {
      return groupConfigs(this.configs)
    },
    // 分组后的v2ray节点
    groupedNodes () {
      return Object.keys(this.groupedConfigs).map(groupName => {
        const node = {
          title: groupName,
          expand: true,
          selected: groupName === this.selectedGroupName,
          children: this.groupedConfigs[groupName],
        }
        return node
      })
    },
    // 选中的节点数据
    selectedConfigNode () {
      if (this.selectedConfigId) {
        return this.configs.find(config => config.id === this.selectedConfigId)
      }
      return null
    },
    // 按钮禁用状态
    disabled () {
      if (!this.selectedConfigId && !this.selectedGroupName) {
        return { remove: true, up: true, down: true }
      }
      if (this.selectedGroupName) {
        // 选中的是分组
        const index = this.groupedNodes.findIndex(node => node.title === this.selectedGroupName)
        const isUngrouped = this.selectedGroupName === '未分组'
        const isPrevUngrouped = index < 1 ? false : this.groupedNodes[index - 1].title === '未分组'
        const isNextUngrouped =
          index > this.groupedNodes.length - 2 ? false : this.groupedNodes[index + 1].title === '未分组'
        return {
          remove: isUngrouped,
          up: isUngrouped || isPrevUngrouped || index < 1,
          down: isUngrouped || isNextUngrouped || index > this.groupedNodes.length - 2,
        }
      }
      // 选中的是配置节点
      const currentGroup = this.selectedConfigNode.group || '未分组'
      const group = this.groupedConfigs[currentGroup]
      const inGroupIndex = group.indexOf(this.selectedConfigNode)
      return {
        remove: false,
        // 前一项不存在或前一项是分组节点
        up: inGroupIndex <= 0,
        // 后一项不存在或后一项是分组节点
        down: inGroupIndex >= group.length - 1,
      }
    },
  },
  watch: {
    'appConfig.index' (v) {
      if (preventIndexAffect) {
        preventIndexAffect = false
      } else {
        this.selectedGroupName = ''
        this.selectedConfigId = this.selectedConfig ? this.selectedConfig.id : ''
      }
    },
    'editingGroup.updated' (v) {
      if (v) {
        this.updateEditingGroup({ updated: false })
        this.selectedGroupName = this.editingGroup.title
      }
    },
    selectedConfigId () {
      this.setCurrentConfig(this.selectedConfigNode)
    },
  },
  methods: {
    ...mapMutations(['setCurrentConfig', 'updateEditingGroup', 'updateView', 'resetState']),
    ...mapActions(['updateConfigs', 'updateConfig', 'updateEditingBak']),
    // 复制节点并带上title和选中参数
    cloneConfig (config, selected) {
      return {
        title: `${config.ps || config.add} (${config.add}:${config.port})`,
        selected,
        ...config,
      }
    },
    // 设置节点选中状态
    setSelected (group, config) {
      this.selectedGroupName = group
      this.selectedConfigId = config
    },
    // 点击节点时
    onSelect (selection) {
      // 第二次点击是取消选择，这个bug曾经修复，又重新复现
      // https://github.com/iview/iview/issues/1533
      if (selection.length > 0) {
        this.selectedNode = selection[0]
      } else {
        this.selectedNode.selected = true
      }
      const node = this.selectedNode
      if (!node.children) {
        // 选中配置项
        this.setSelected('', node.id)
        this.updateEditingGroup({ show: false })
      } else {
        // 选中分组
        this.setSelected(node.title, '')
        this.updateEditingGroup({
          show: true,
          title: node.title === '未分组' ? '' : node.title,
        })
      }
    },
    // 双击选中该节点
    // TODO: 新版 iview tree 没有双击事件
    onNodeDBClick (selection) {
      const node = selection[0]
      this.updateConfig({
        index: this.appConfig.configs.findIndex(config => config.id === node.id),
      })
      this.resetState()
      hideWindow()
    },
    applyNode () {
      if (this.editingConfig.isValid()) {
        if (this.isEditingConfigUpdated) {
          const copy = this.appConfig.configs.slice()
          const index = copy.findIndex(config => config.id === this.editingConfig.id)
          copy.splice(index, 1)
          copy.splice(index, 0, clone(this.editingConfig))
          this.updateEditingBak()
          this.updateConfigs(copy)
        } else {
          const node = this.$refs.tree.getSelectedNodes()[0]
          if (!node.children) {
            this.updateConfig({
              index: this.appConfig.configs.findIndex(config => config.id === node.id),
            })
            this.resetState()
          }
          // hideWindow()
        }
      } else {
        window.alert('服务器配置信息不完整')
      }

    },
    // flat分组
    flatNodeGroups (groups) {
      groups = groups || this.groupedNodes
      const flatArr = []
      groups.forEach(group => {
        flatArr.push(...group.children)
      })
      return flatArr
    },
    // 从剪切板导入
    copyFromClipboard () {
      ipcRenderer.send(EVENT_CONFIG_COPY_CLIPBOARD)
    },
    // 跳转到订阅管理页面
    toSubscribe () {
      this.updateView({ page: 'Options', tab: 'subscribes', active: true })
    },
    // 新增
    create () {
      const newConfig = new Config(this.selectedConfigNode)
      const clone = this.appConfig.configs.slice()
      clone.push(newConfig)
      this.updateConfigs(clone)
      this.setSelected('', newConfig.id)
      this.updateEditingGroup({ show: false })
    },
    // 删除分组
    removeGroup () {
      const clone = this.appConfig.configs.slice()
      this.updateConfigs(clone.filter(config => config.group !== this.selectedGroupName))
      this.setSelected('', this.selectedConfig ? this.selectedConfig.id : '')
      this.updateEditingGroup({ show: false })
    },
    // 删除
    remove () {
      const clone = this.appConfig.configs.slice()
      const index = clone.findIndex(config => config.id === this.selectedConfigId)
      clone.splice(index, 1)
      this.updateConfigs(clone)
      const next = clone[index]
      const prev = clone[index - 1]
      this.setSelected('', next ? next.id : prev ? prev.id : '')
    },
    // 上/下移 direction = 1 上移 其它 下移
    updown (direction = 1) {
      const clone = this.groupedNodes.slice()
      if (this.selectedGroupName) {
        // 分组上/下移
        const index = clone.findIndex(node => node.title === this.selectedGroupName)
        const group = clone.splice(index, 1)
        clone.splice(direction === 1 ? index - 1 : index + 1, 0, group[0])
      } else {
        // 节点上/下移
        const currentGroup = this.selectedConfigNode.group || '未分组'
        const group = clone.find(node => node.title === currentGroup)
        const childrenClone = group.children.slice()
        const inGroupIndex = childrenClone.findIndex(node => node.id === this.selectedConfigId)
        childrenClone.splice(
          direction === 1 ? inGroupIndex - 1 : inGroupIndex + 1,
          0,
          childrenClone.splice(inGroupIndex, 1)[0]
        )
        group.children = childrenClone
      }
      preventIndexAffect = true
      this.updateConfigs(this.flatNodeGroups(clone))
    },
  },
}
</script>
<style lang="stylus">
@import '../../assets/styles/variable'

.panel-nodes
  width 12.5rem
  .ivu-dropdown-btn-trigger
    width 28px
    padding-left 4px
    padding-right @padding-left
    border-top-left-radius 0
    border-bottom-left-radius 0
  .empty-tree
    display flex
    justify-content center
    align-items center
  .node-tree
    border 1px solid $color-border
    border-radius 4px
    overflow-x hidden
    overflow-y auto
  .ivu-tree-children
    .ivu-tree-children
      .ivu-tree-arrow
        display none
</style>
