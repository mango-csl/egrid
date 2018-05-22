<template>
  <div class="table-container" ref="tableContainer">
    <!--header-row-class-name="test"-->
    <!--:rowKey="rowKey"-->
    <el-table class="egrid"
              :data="data" ref="grid"
              header-row-class-name="wms-table-header"
              v-loading="loading"
              v-bind="tableBind"
              v-on="$listeners">
      <template v-for="tp in typesColumns">
        <el-table-tree-column v-if="tp.type === 'tree'"
                              :fixed='tp.fixed'
                              :remote="remote"
                              v-bind="tp.props"
        >
        </el-table-tree-column>
        <el-table-column
          v-else-if="tp.type === 'expand'"
          v-bind="tp.props"
          type="expand"
          :fixed='tp.fixed'
        >
          <template slot-scope="props">
            <slot name="expand" v-bind="props"></slot>
          </template>
        </el-table-column>
        <el-table-column v-else-if="tp.type === 'index'" :render-header="renderHeader"
                         :fixed='tp.fixed'
                         v-bind="tp.props">
          <template slot-scope="scope">
            {{pageSize * ( pageNow - 1 ) + Number(scope.$index) + 1}}
          </template>
        </el-table-column>
        <el-table-column v-else-if="tp.type === 'selection'"
                         :type="tp.type"
                         :fixed='tp.fixed'
                         v-bind="tp.props">
        </el-table-column>
      </template>
      <el-table-column v-for="col in showColumns"
                       :key="col.label" v-bind="getColBind(col)">
        <template slot-scope="scope">
          <component :is="col.component"
                     v-bind="getCptBind(scope, col)"
                     v-on="col.listeners">
          </component>
        </template>
      </el-table-column>
      <template v-if="slotAppend" slot="append">
        <slot name="append"></slot>
      </template>
    </el-table>
    <div class="scroll" v-bind:class="{'scroll-hasY': isTableHasY ,'scroll-fixed':isScrollFixed}"
         v-bind:style="{ width: scrollContainerWidth }"
         ref="scrollElement"
         v-if="showScroll"
         @scroll="scrollBarMove($event)">
      <div v-bind:style="{ height: '1px', width: scrollbodyWidth }" ref="scrollbars"></div>
    </div>
  </div>
</template>

<script>
  import ElTable from 'element-ui/lib/table';
  import ElTableColumn from 'element-ui/lib/table-column';
  import ElTreeGrid from 'element-tree-grid';
  import throttle from 'throttle-debounce/throttle';
  import methods from './methods';
  import Text from './text';

  const BOOLEAN_KEYS = [
    'fit',
    'stripe',
    'border',
    'show-header',
    'highlight-current-row',
    'default-expand-all',
    'show-summary'
  ];
  // element table 表格默认配置项
  const DEFAULT_BOOLEAN_KEYS = {
    'highlight-current-row': ''
  };

  const COLUMN_PROPS = {
    align: 'left',
    component: Text
  };

  const TYPES = ['tree', 'selection', 'expand', 'index'];

  const TYPE_COLUMN_OPTION = {
    selection: {
      width: 48,
      reserveSelection: false
    },
    expand: {
      width: 48
    },
    index: {
      width: 80
    },
    tree: {
      expandAll: false,
      treeKey: 'id',
      childNumKey: 'child_num',
      parentKey: 'parent_id',
      width: 48
    }
  };

  const COLUMN_KEY_MAP = {
    label: 'label',
    prop: 'prop'
  };

  export default {
    name: 'Egrid',
    mixins: [methods],
    props: {
      data: {
        type: Array,
        default: []
      },
      loading: {
        type: Boolean,
        default: false
      },
      pageNow: { // 当前页码 控制序号
        type: Number,
        default: 0
      },
      pageSize: { // 当前页数据最多展示数量 控制序号
        type: Number,
        default: 20
      },
      columns: {
        type: Array,
        default: []
      },
      // columnType: [String, Array],
      columnType: Array,
      // 特殊列，自定义配置
      columnTypeProps: Object,
      columnKeyMap: Object,
      columnsProps: Object,
      columnsSchema: Object,
      columnsHandler: Function,
      slotAppend: Boolean
    },
    components: {
      [ElTable.name]: ElTable,
      [ElTreeGrid.name]: ElTreeGrid,
      [ElTableColumn.name]: ElTableColumn
    },
    data() {
      // 设置初始选择值
      let _filteredColumn = this.columns.map(item => item.prop);
      // key值转换
      const map = Object.assign({}, COLUMN_KEY_MAP, this.columnKeyMap);
      let _column = this.columns.map(item => {
        return {
          label: item[map.label],
          prop: item[map.prop]
        };
      });
      return {
        // rowKey: undefined, // 标识列
        showColumns: [], // 实际每列的配置
        typesColumns: [], // 特殊功能列配置
        exploreSize: {}, // 视窗大小参数
        scrollYWidth: 17, // 横向滚动条宽度
        column_key_Map: map,
        columnOptions: _column,
        filteredColumn_temp: _filteredColumn,
        _escapeListenerResize: undefined,
        _escapeListenerScroll: undefined,
        minColunmWidth: 80,
        isTableHasY: false, // table 是否存在纵向滚动条
        isScrollFixed: false, // 横向滚动条 是否需要更改position
        scrollbodyWidth: '0px',
        scrollContainerWidth: '0px',
        showScroll: true
      };
    },
    computed: {
      // 处理 $attrs 里面 Boolean 类型的 prop 和统一 prop 命名
      tableBind() {
        // let {$attrs} = this
        const $attrs = Object.assign({}, this.$attrs, DEFAULT_BOOLEAN_KEYS);
        const bind = {};
        Object.keys($attrs).forEach(key => {
          const v = $attrs[key];
          const uniformKey = key.replace(/([A-Z])/, '-$1').toLowerCase();
          bind[key] = ~BOOLEAN_KEYS.indexOf(uniformKey) && v === '' ? true : v;
        });
        return bind;
      },
      filteredColumn: {
        set(obj) {
          console.log('filteredColumn = ', obj);
          this.changeColumn(obj);
        },
        get() {
          return this.filteredColumn_temp;
        }
      }
    },
    methods: {
      remote(row, callback) {
        // let _that = this;
        if (this.columnTypeProps.tree && this.columnTypeProps.tree.remotePromise) {
          let _promise = this.columnTypeProps.tree.remotePromise;
          _promise(row).then((data) => {
            // let _options = _that.typesColumns.find(item => item.type === 'tree');
            // console.info('_options', _options.props);
            // callback(data.filter(f => f[_options.props.parentKey] === row[_options.props.treeKey]));
            let depth = parseInt(row.depth, 10);
            let depth_child = isNaN(depth) ? 1 : depth + 1;
            let _data = data;
            _data.map(item => Object.assign(item, {depth: depth_child}));
            callback(_data);
            // 重新判断是否需要'悬浮'横向滚动条
            this.$nextTick(() => {
              if (this._escapeListenerScroll) {
                this.judgeTableInView();
              }
            });
          });
        } else {
          callback();
        }
      },
      chooseColumn(value) {
        let _filteredColumn = this.filteredColumn.splice(0);
        let _index = _filteredColumn.findIndex(item => item === value);
        if (_index > -1) {
          _filteredColumn.splice(_index, 1);
        } else {
          _filteredColumn.push(value);
        }
        this.filteredColumn = _filteredColumn;
      },
      // 创建自定义头
      renderHeader(h, {row, column, store, $index}) {
        const items = this.columnOptions.map((item, index) => {
          return <el-option
            nativeOn-click={this.chooseColumn.bind(null, item.prop)}
            key={item.prop}
            label={item.label}
            value={item.prop}>
          </el-option>;
        });
        return <div class="suffix-span suffix-container">
          <div class="suffix-span">序号</div>
          <el-select
            value={this.filteredColumn}
            multiple
            collapse-tags
            placeholder="请选择">
            {items}
          </el-select>
        </div>;
      },
      // 列选择事件
      changeColumn(obj) {
        this.filteredColumn_temp = obj;
        this.renderNormalColumns();
      },
      renderNormalColumns() {
        const {
          columns,
          columnsHandler,
          filteredColumn_temp,
          columnsProps: props,
          columnsSchema: schema
        } = this;
        // 筛选要展示的列
        let _column = columns.filter(item => filteredColumn_temp.includes(item[this.column_key_Map.prop]));
        let renderColumns = _column.map(col => {
          const mix = schema && schema[col[this.column_key_Map.label]] || {};
          const it = Object.assign({}, COLUMN_PROPS, props, col, mix);
          it.label = it[this.column_key_Map.label];
          it.prop = it[this.column_key_Map.prop];
          it.showOverflowTooltip = true;
          return it;
        });
        let _tempColumns = columnsHandler && columnsHandler(renderColumns) || renderColumns;
        this.$nextTick(() => {
          this.showColumns = this.caculateColunmsWidth(_tempColumns);
        });
      },
      // 判断是否需要重新设置列宽
      checkColumn(data) {
        return data.width === 'auto' || !data.width;
      },
      // 用于渲染特殊列
      renderTypeColumns() {
        const {columnType: type, columnTypeProps} = this;
        let typeColums = [];
        // if (typeof type === 'string' && ~TYPES.indexOf(type)) {
        //     typeColums = [type]
        // }
        if (Array.isArray(type)) {
          typeColums = type.filter(it => ~TYPES.indexOf(it));
        }
        const map = columnTypeProps || {};

        this.typesColumns = typeColums.map(type => {
          const _options = Object.assign(TYPE_COLUMN_OPTION[type], map[type]);
          // if (type === 'tree' && map[type].hasOwnProperty('treeKey')) {
          //   this.rowKey = map[type].treeKey;
          // }
          return {
            type,
            props: _options,
            fixed: 'left'
          };
        });
      },
      getColBind(col) {
        const bind = Object.assign({}, col);
        delete bind.component;
        delete bind.listeners;
        delete bind.propsHandler;
        return bind;
      },
      getCptBind({row, column}, col) {
        const props = {row, col, column};
        const handler = col.propsHandler;
        return handler && handler(props) || props;
      },
      // 初始和数据源变化时，重新生成列
      createColunms() {
        // 注意先后执行顺序
        this.renderTypeColumns();
        this.renderNormalColumns();
      },
      // windowResize触发事件
      resize() {
        this.getViewport();
        this.renderNormalColumns();
      },
      resizeListener() {
        window.addEventListener('resize', throttle(100, this.resize));
        this._escapeListenerResize = () => {
          window.removeEventListener('resize', throttle(100, this.resize));
        };
      },
      // table是否需要横向滚动判断规则
      caculateColunmsWidth(_column) {
        let orderWidth = 0; // 已限定宽度
        let normalColumnsWidth = 0;// 预测table宽度
        for (let item of _column) {
          if (item.width) {
            orderWidth += Number(item.width);
            normalColumnsWidth += Number(item.width);
          } else {
            normalColumnsWidth += this.minColunmWidth;
          }
        }
        let containerWidth = this.$refs.tableContainer.clientWidth;
        let typesColumnsWidth = 0; // 特殊列总宽度
        if (this.$method.hasValue_arr(this.columnType)) {
          for (let key of this.columnType) {
            typesColumnsWidth += TYPE_COLUMN_OPTION[key].width;
          }
        }
        let tableWidth = normalColumnsWidth + typesColumnsWidth; // table总宽度
        if (containerWidth >= tableWidth) {
          this.destoryScroll();
          let count = Number((tableWidth - orderWidth - typesColumnsWidth) / 80);
          let _columnWidth = ~~((containerWidth - orderWidth - typesColumnsWidth) / count);
          for (let item of _column) {
            if (this.checkColumn(item)) {
              item.width = _columnWidth + '';
            }
          }
        } else {
          for (let item of _column) {
            if (this.checkColumn(item)) {
              item.width = 80 + '';
            }
          }
          this.creartScroll(tableWidth);
        }
        return _column;
      },
      // 创建横向滚动条
      creartScroll(width) {
        this.$nextTick(() => {
          if (!this.showScroll) {
            this.showScroll = true;
          }
          let _tableContainer = this.$refs.tableContainer;
          this.isTableHasY = (this.data.length + 1) * 44 > _tableContainer.clientHeight;
          this.scrollbodyWidth = width + 'px';
          this.scrollContainerWidth = _tableContainer.clientWidth + 'px';
          this.judgeTableInView();
          window.addEventListener('scroll', throttle(100, this.judgeTableInView));
          this._escapeListenerScroll = () => {
            window.removeEventListener('scroll', throttle(100, this.judgeTableInView));
          };
        });
      },
      destoryScroll() {
        if (this.showScroll) {
          this.showScroll = false;
        }
        if (this._escapeListenerScroll) {
          this._escapeListenerScroll();
          this._escapeListenerScroll = undefined;
        }
      },
      // 联动横向滚动
      scrollBarMove(event) {
        this.$refs.grid.$el.childNodes[2].scrollLeft = event.target.scrollLeft;
      },
      // 获取浏览器视窗大小
      getViewport() {
        // http://andylangton.co.uk/blog/development/get-viewport-size-width-and-height-javascript
        let e = window;
        let a = 'inner';
        if (!('innerWidth' in window)) {
          a = 'client';
          e = document.documentElement || document.body;
        }
        this.exploreSize = {
          width: e[a + 'Width'],
          height: e[a + 'Height']
        };
      },
      // 判断滚动条是否需要悬浮
      judgeTableInView() {
        let _tableContainer = this.$refs.tableContainer;
        if (_tableContainer) {
          let pos = _tableContainer.getBoundingClientRect();
          let nowTop = this.exploreSize.height - pos.top;
          let result = nowTop < _tableContainer.clientHeight && nowTop > 44;
          // + this.scrollYWidth
          if (result !== this.isScrollFixed) {
            this.isScrollFixed = result;
          }
          console.log('滚动条是否需要悬浮 = ', result);
        }
      }
    },
    mounted() {
      this.$nextTick(() => {
        this.getViewport();
        this.createColunms();
        this.resizeListener();
      });
    },
    beforeDestroy() {
      if (this._escapeListenerResize) {
        this._escapeListenerResize();
      }
      if (this._escapeListenerScroll) {
        this._escapeListenerScroll();
      }
    },
    watch: {
      data(val) {
        // console.log('data = ', val);
        this.createColunms();
      }
    }
  };
</script>

<style>
  .el-table th, .el-table tr.wms-table-header {
    background-color: #38ADFF;
    font-size: 14px;
    color: white;
  }

  .scroll {
    overflow-x: auto;
    overflow-y: hidden;
    position: absolute;
    z-index: 222;
    bottom: 0;
    width: 100%;
  }

  .scroll-hasY {
    width: calc(100% - 16px);
  }

  .scroll-fixed {
    position: fixed;
  }

  .table-container {
    display: -webkit-box;
    display: -ms-flexbox;
    display: flex;
    position: relative;
    /* max-height: 300px; */
    height: 100%;
    width: 100%;
    -webkit-box-orient: vertical;
    -webkit-box-direction: normal;
    -ms-flex-direction: column;
    flex-direction: column;
  }

  .el-table th .suffix-span {
    line-height: 16px;
    box-sizing: border-box;
    white-space: nowrap;
    display: inline-block;
    padding: 0;
  }

  .el-table th .suffix-container {
    width: 100%;
    text-align: center;
    position: relative;
    top: 4px;
  }

  .el-table th .el-select {
    padding: 0;
    line-height: 16px;
    width: 25px;
  }

  .el-table th .el-input--suffix {
    padding: 0;
    line-height: 16px;
    height: 16px;
  }

  .el-table th .el-input--suffix .el-input__inner {
    padding: 0;
    border: 0;
    height: 0;
  }

  .el-table th .el-input--suffix .el-input__suffix {
    right: 0;
    /*line-height: 16px;*/
  }

  .el-table th .el-select__tags {
    display: none;
  }
</style>
<!--展开子节点数据-->
<!--<el-table-column v-if="tp.type === 'expand'"-->
<!--:fixed='tp.fixed'-->
<!--:width="tp.width">-->
<!--<template slot-scope="props">-->
<!--<div class='el-table__expand-icon el-table__expand-icon&#45;&#45;expanded' v-if="props.row.hasChild"-->
<!--@click=props.store.table.toggleRowExpansion(props.row)>-->
<!--<i class='el-icon el-icon-arrow-right'></i>-->
<!--</div>-->
<!--&lt;!&ndash;<slot name="expand" v-bind="props"></slot>&ndash;&gt;-->
<!--</template>-->
<!--</el-table-column>-->
<!--align="center"-->
