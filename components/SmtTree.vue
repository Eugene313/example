<template>
  <div class="smt-tree">
    <SmtTreeHeader
      v-if="search"
      v-model="filter.phrase"
      :placeholder="placeholder || defaultPlaceholder"
      @on-change-filter="changeFilter"
    />
    <SmtTreeActions
      :filter="filter"
      :tree-model-length="treeModel.length"
    />
    <SmtTreeCols
      :tree-expanded-state="treeExpandedState"
      :header="header || defaultHeader"
      :is-single-mode="isSingleMode"
      :is-all-items-checked="isAllItemsChecked"
      :is-some-items-checked="isSomeItemsChecked"
      @on-change-tree-expanded-state="changeTreeExpandedState"
      @on-change-all-item-checked="changeAllItemChecked"
    />
    <div
      v-if="treeExpandedState"
      ref="tree"
      class="smt-tree_body"
    >
      <div
        v-if="!loading"
        :style="lazyScrollStyle"
        class="smt-tree_lazy-scroll"
      >
        <template v-for="(treeItem, treeItemIndex) in visibleTreeModel">
          <SmtTreeItem
            v-if="isTreeItemVisible(treeItemIndex)"
            :key="treeItem.id"
            :tree-item="treeItem"
            :header="header || defaultHeader"
            :tree-item-index="treeItemIndex"
            @on-change-item-checked="changeItemChecked"
            @on-change-item-expanded="changeItemExpanded"
          />
        </template>
      </div>
      <div
        v-if="loading"
        class="smt-tree_loading"
      >
        <div class="smt-tree_loading-spinner" />
      </div>
      <div
        v-if="!loading && !visibleTreeModel.length"
        class="smt-tree_nothing-found"
      >
        {{ $t('nothingFound') }}
      </div>
    </div>
    <SmtTreeTags
      v-if="showTags"
      :header="header || defaultHeader"
      :checked-items="checkedItems"
      @on-change-item-checked="changeItemChecked"
    />
  </div>
</template>
<script>
import SmtTreeHeader from '~/components/Common/SmtTree/SmtTreeHeader';
import SmtTreeActions from '~/components/Common/SmtTree/SmtTreeActions';
import SmtTreeCols from '~/components/Common/SmtTree/SmtTreeCols';
import SmtTreeItem from '~/components/Common/SmtTree/SmtTreeItem';
import SmtTreeTags from '~/components/Common/SmtTree/SmtTreeTags';

export default {
  name: 'SmtTree',
  components: {
    SmtTreeHeader,
    SmtTreeActions,
    SmtTreeCols,
    SmtTreeItem,
    SmtTreeTags,
  },
  props: {
    value: {
      type: [Array, String],
      default: () => ([]),
    },
    mode: {
      type: Number,
      default: 4,
    },
    header: {
      type: Array,
      default: null,
    },
    url: {
      type: String,
      default: '',
    },
    params: {
      type: Object,
      default: () => ({}),
    },
    placeholder: {
      type: String,
      default: '',
    },
    search: {
      type: Boolean,
      default: true,
    },
    showTags: {
      type: Boolean,
      default: true,
    },
  },
  data() {
    return {
      treeScrollPosition: 0,
      treeExpandedState: true,
      loading: false,
      filter: {
        phrase: '',
        ...this.params,
      },
      treeModel: [],
      defaultPlaceholder: this.$t('defaultPlaceHolder'),
      defaultHeader: [
        {
          title: this.$t('code'),
          name: 'code',
          type: 'text',
          style: {
            minWidth: '150px',
          },
        },
        {
          title: this.$t('title'),
          name: 'title',
          type: 'text',
          style: {
            width: '100%',
          },
        },
      ],
    };
  },
  computed: {
    treeValue: {
      get() {
        return this.value;
      },
      set(value) {
        this.$emit('input', value);
      },
    },
    visibleTreeModel() {
      return this.treeModel.filter(treeItem => !treeItem.parent || treeItem.parent.state.expanded);
    },
    lazyScrollStyle() {
      return {
        height: `${this.visibleTreeModel.length * 32}px`,
      };
    },
    isSingleMode() {
      return this.mode === 1;
    },
    isMultipleMode() {
      return this.mode === 2;
    },
    isTreeMode() {
      return this.mode === 3;
    },
    isProzorroMode() {
      return this.mode === 4;
    },
    isAllItemsChecked() {
      return !this.isSingleMode ? this.treeModel.length === this.treeValue.length : false;
    },
    isSomeItemsChecked() {
      return !this.isSingleMode ? !this.isAllItemsChecked && !!this.treeValue.length : false;
    },
    checkedItems() {
      return this.treeModel.filter(item => item.state.checked);
    },
  },
  async created() {
    await this.getTreeModel();
    this.addScrollEventListener();
  },
  beforeDestroy() {
    this.removeScrollEventListener();
  },
  methods: {
    setTreeExpandedState(value) {
      this.treeExpandedState = value;
    },
    setLoading(value) {
      this.loading = value;
    },
    setTreeValue(value) {
      this.treeValue = value;
    },
    setTreeModel(tree) {
      this.treeModel = tree;
    },
    setItemChecked(item, value = false) {
      item.state.checked = value;
    },
    setItemExpanded(item, value = false) {
      item.state.expanded = value;
    },
    setParent(item, parentItem) {
      item.parent = parentItem;
    },
    setLevel(item, level) {
      item.level = level;
    },
    setValueToTreeModel(tree) {
      if (this.isSingleMode) {
        tree.forEach((itemTree) => {
          this.setItemChecked(itemTree, this.treeValue === itemTree.id);
        });
      } else {
        tree.forEach((itemTree) => {
          this.setItemChecked(itemTree, this.treeValue.includes(itemTree.id));
        });
      }
    },
    setTreeScrollPosition(value = 0) {
      this.treeScrollPosition = value;
    },
    addScrollEventListener() {
      this.$refs.tree?.addEventListener('scroll', (event) => {
        this.setTreeScrollPosition(event.target.scrollTop);
      });
    },
    removeScrollEventListener() {
      this.$refs.tree?.removeEventListener('scroll', this.setTreeScrollPosition);
    },
    isTreeItemVisible(index) {
      return (index + 1) * 32 > this.treeScrollPosition && (index * 32) < this.treeScrollPosition + 300;
    },
    traverseTreeUp(item, processItemFunction = () => {}) {
      let parent = item;
      while (parent) {
        processItemFunction(parent);
        parent = parent.parent;
      }
    },
    traverseTreeDown(tree = [], processItemFunction = () => {}, processChildFunction = () => {}) {
      let items = tree;
      while (items.length > 0) {
        items = items.reduce((total, item) => {
          processItemFunction(item);
          (item.children || []).forEach((child) => {
            processChildFunction(child, item);
            total.push(child);
          });
          return total;
        }, []);
      }
    },
    optimizedTree(tree) {
      tree.forEach((itemTree) => {
        Object.defineProperty(itemTree, 'id', { configurable: false });
        Object.defineProperty(itemTree, 'children', { configurable: false });
        Object.defineProperty(itemTree, 'parent', { configurable: false });
        Object.defineProperty(itemTree, 'iconType', { configurable: false });
        delete itemTree.parentId;
        Object.defineProperty(itemTree, 'columns', { configurable: false });
        Object.defineProperty(itemTree.state, 'disabled', { configurable: false });
        Object.defineProperty(itemTree.state, 'finded', { configurable: false });
        Object.defineProperty(itemTree, 'level', { configurable: false });
      });
    },
    checkedItemsHandler(items, value) {
      this.traverseTreeDown(items, (item) => {
        return this.setItemChecked(item, value);
      });
    },
    singleModCheckedHandler(item, value) {
      if (value) {
        this.traverseTreeDown(this.treeModel, (treeItem) => {
          return this.setItemChecked(treeItem, item.id === treeItem.id);
        });
        this.setItemExpanded(item, true);
      } else {
        this.setItemChecked(item, value);
      }
    },
    multipleModCheckedHandler(item, value) {
      this.setItemChecked(item, value);
      this.setItemExpanded(item, true);
    },
    treeModCheckedHandler(item, value) {
      if (value) {
        this.setItemChecked(item, value);
        this.checkedItemsHandler(item.children || [], value);
        this.setItemExpanded(item, true);
        this.traverseTreeUp(item.parent, (parent) => {
          this.setItemChecked(parent, parent.children.every(child => child.state.checked));
        });
      } else {
        this.setItemChecked(item, value);
        this.checkedItemsHandler(item.children || [], value);
        this.traverseTreeUp(item.parent, (parent) => {
          this.setItemChecked(parent, value);
        });
      }
    },
    prozorroModCheckedHandler(item, value) {
      if (value) {
        this.setItemChecked(item, value);
        this.checkedItemsHandler(item.children || [], value);
        this.setItemExpanded(item, value);
      } else {
        this.setItemChecked(item, value);
        this.checkedItemsHandler(item.children || [], value);
      }
    },
    refreshTreeValue() {
      const result = [];
      this.checkedItems.forEach((item) => {
        if (item.state.checked) {
          result.push(item.id);
        }
      });
      this.setTreeValue(this.isSingleMode ? result[0] : result);
    },
    getFlattenTree(items, parent, level = 0) {
      const result = [];
      items.forEach((item) => {
        this.setParent(item, parent);
        this.setLevel(item, level);
        result.push(item);
        if (item.children) {
          const children = this.getFlattenTree(item.children, item, level + 1);
          result.push(...children);
        }
      });
      return result;
    },
    async getTreeModel() {
      try {
        this.setLoading(true);
        const response = await this.$axios.get(this.url, { params: this.filter });
        const flattenTree = this.getFlattenTree(response.data || []);
        this.optimizedTree(flattenTree);
        this.setValueToTreeModel(flattenTree);
        this.setTreeModel(flattenTree);
      } catch (error) {
        this.errorHandler(error);
      } finally {
        this.setLoading(false);
      }
    },
    async changeFilter() {
      await this.getTreeModel();
    },
    changeTreeExpandedState(value) {
      this.setTreeExpandedState(value);
      if (value) {
        this.$nextTick(this.addScrollEventListener);
      } else {
        this.setTreeScrollPosition();
        this.$nextTick(this.removeScrollEventListener);
      }
    },
    changeAllItemChecked(value) {
      this.checkedItemsHandler(this.treeModel, value);
      this.refreshTreeValue();
    },
    changeItemChecked(item, value) {
      if (this.isSingleMode) {
        this.singleModCheckedHandler(item, value);
      } else if (this.isMultipleMode) {
        this.multipleModCheckedHandler(item, value);
      } else if (this.isTreeMode) {
        this.treeModCheckedHandler(item, value);
      } else if (this.isProzorroMode) {
        this.prozorroModCheckedHandler(item, value);
      }
      this.refreshTreeValue();
    },
    changeItemExpanded(item) {
      this.setItemExpanded(item, !item.state.expanded);
      if (!item.state.expanded) {
        this.traverseTreeDown(item.children, this.setItemExpanded);
      }
    },
    getCheckedItems() {
      return this.checkedItems;
    },
    errorHandler(error) {
      this.$Notice.error({
        title: this.$t('error'),
        error,
      });
    },
  },
  i18n: {
    messages: {
      uk: {
        code: 'Код',
        title: 'Назва',
        defaultPlaceHolder: 'Введіть назву категорії, товару чи послуги, або код ДК',
        nothingFound: 'На жаль, за вашим запитом нічого не знайдено.',
      },
      ru: {
        code: 'Код',
        title: 'Название',
        defaultPlaceHolder: 'Введите название категории, товара или услуги, или код ГК',
        nothingFound: 'К сожалению, по вашему запросу ничего не найдено.',
      },
      en: {
        code: 'Code',
        title: 'Name',
        defaultPlaceHolder: 'Enter a name for the category, product, or service, or SC code',
        nothingFound: 'Unfortunately, your search returned no results.',
      },
    },
  },
};
</script>
<style lang="sass">
.smt-tree
  .smt-tree_body
    height: 300px
    overflow-y: scroll
    box-shadow: 0 0 1px 0 #c2c2c2
  .smt-tree_lazy-scroll
    position: relative
  .smt-tree_loading
    width: 100%
    height: 100%
    display: flex
    justify-content: center
    align-items: center
  .smt-tree_loading-spinner
    width: 37px
    height: 37px
    margin-bottom: 15px
    border-radius: 50%
    border: 4px solid #e1e1e1
    border-top-color: #60a12b
    animation: circle
    animation-duration: 1s
    animation-timing-function: linear
    animation-iteration-count: infinite
    @-webkit-keyframes circle
      0%
        transform: rotateZ(0deg)
      100%
        transform: rotateZ(360deg)
  .smt-tree_nothing-found
    width: 100%
    height: 100%
    display: flex
    justify-content: center
    align-items: center
    font-weight: bold
    font-size: 18px
</style>
