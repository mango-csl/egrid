<template>
    <div class="table-container" ref="tableContainer">
        <el-select
                v-model="filteredColumn"
                multiple
                collapse-tags
                placeholder="请选择">
            <el-option
                    v-for="item in columnOptions"
                    :key="item.prop"
                    :label="item.label"
                    :value="item.prop">
            </el-option>
        </el-select>
        <el-table class="egrid"
                  :data="data" ref="grid"
                  v-loading="loading"
                  v-bind="tableBind"
                  v-on="$listeners">
            <template v-for="tp in typesColumns">
                <el-table-column
                        v-if="tp.type === 'expand'"
                        v-bind="tp.props"
                        type="expand"
                        :fixed='tp.fixed'
                        :width="tp.width"
                        :key="tp.type">
                    <template slot-scope="props">
                        <slot name="expand" v-bind="props"></slot>
                    </template>
                </el-table-column>
                <el-table-column v-else
                                 :key="tp.type"
                                 :type="tp.type"
                                 :fixed='tp.fixed'
                                 :width="tp.width"
                                 v-bind="tp.props">
                    <!--:width="tp.width"-->
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
        <div class="scroll" v-bind:class="{'scroll-hasY': istableHasY ,'scroll-fixed':isScrollFixed}" ref="scrollElement"
             v-if="showScroll"
             @scroll="scrollBarMove($event)">
            <div v-bind:style="{ height: '1px', width: scrollbodyWidth }" ref="scrollbars"></div>
        </div>
    </div>
</template>

<script>
    import ElTable from 'element-ui/lib/table'
    import ElTableColumn from 'element-ui/lib/table-column'
    import methods from './methods'
    import Text from './text'

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

    const TYPES = ['selection', 'expand', 'index'];

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
                default() {
                    return []
                }
            },
            loading: {
                type: Boolean,
                default: false
            },
            columns: {
                type: Array,
                default() {
                    return []
                }
            },
            columnType: [String, Array],
            columnTypeProps: Object,
            columnKeyMap: Object,
            columnsProps: Object,
            columnsSchema: Object,
            columnsHandler: Function,
            slotAppend: Boolean
        },
        components: {
            [ElTable.name]: ElTable,
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
                }
            });
            return {
                showColumns: [], // 实际每列的配置
                typesColumns: [], // 特殊功能列配置
                exploreSize: {}, // 视窗大小参数
                column_key_Map: map,
                columnOptions: _column,
                filteredColumn_temp: _filteredColumn,
                _escapeListenerResize: undefined,
                _escapeListenerScroll: undefined,
                minColunmWidth: 80,
                istableHasY: false, // table 是否存在纵向滚动条
                isScrollFixed: false, // 横向滚动条 是否需要更改position
                scrollbodyWidth: '0px',
                showScroll: true,
            }
        },
        computed: {
            // 处理 $attrs 里面 Boolean 类型的 prop 和统一 prop 命名
            tableBind() {
                // let {$attrs} = this
                const $attrs = Object.assign({}, this.$attrs, this.DEFAULT_BOOLEAN_KEYS);
                const bind = {};
                Object.keys($attrs).forEach(key => {
                    const v = $attrs[key];
                    const uniformKey = key.replace(/([A-Z])/, '-$1').toLowerCase();
                    bind[key] = ~BOOLEAN_KEYS.indexOf(uniformKey) && v === '' ? true : v
                });
                return bind
            },
            filteredColumn: {
                set(obj) {
                    this.changeColumn(obj);
                },
                get() {
                    return this.filteredColumn_temp;
                }
            },
        },
        methods: {
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
                    return it
                });
                let _tempColumns = columnsHandler && columnsHandler(renderColumns) || renderColumns;
                this.showColumns = this.caculateColunmsWidth(_tempColumns);
            },
            // 用于渲染特殊列
            renderTypeColumns() {
                const {columnType: type, columnTypeProps} = this;
                let typeColums = [];
                if (typeof type === 'string' && ~TYPES.indexOf(type)) {
                    typeColums = [type]
                }
                if (Array.isArray(type)) {
                    typeColums = type.filter(it => ~TYPES.indexOf(it))
                }
                const map = columnTypeProps || {};
                this.typesColumns = typeColums.map(type => {
                    return {
                        type,
                        props: map[type],
                        width: 48,
                        fixed: 'left'
                    }
                });
            },
            getColBind(col) {
                const bind = Object.assign({}, col)
                delete bind.component
                delete bind.listeners
                delete bind.propsHandler
                return bind
            },
            getCptBind({row, column}, col) {
                const props = {row, col, column}
                const handler = col.propsHandler
                return handler && handler(props) || props
            },
            createColunms() {
                // 注意先后执行顺序
                this.renderTypeColumns();
                this.renderNormalColumns();
            },
            resize(){
                this.getViewport();
                this.renderNormalColumns();
            },
            resizeListener() {
                window.addEventListener('resize', this.resize);
                this._escapeListenerResize = () => {
                    window.removeEventListener('resize', this.resize);
                };
            },
            caculateColunmsWidth(_column) {
                let orderWidth = 0; // 已限定宽度
                let _tableWidth = 0;// 预测table宽度
                for (let item of _column) {
                    if (item.width) {
                        orderWidth += Number(item.width);
                        _tableWidth += Number(item.width);
                    } else {
                        _tableWidth += this.minColunmWidth;
                    }
                }
                let containerWidth = this.$refs.tableContainer.clientWidth - this.typesColumns.length * 48; // table容器宽度
                // this.istableHasY = (this.data.length + 1) * 44 > this.$refs.tableContainer.clientHeight;
                console.log('containerWidth----', containerWidth);
                if (containerWidth >= _tableWidth) {
                    this.destoryScroll();
                    let count = Number((_tableWidth - orderWidth) / 80);
                    let _columnWidth = (containerWidth - orderWidth) / count;
                    for (let item of _column) {
                        if (!item.width) {
                            item.width = _columnWidth + '';
                        }
                    }
                } else {
                    for (let item of _column) {
                        if (!item.width) {
                            item.width = 80 + '';
                        }
                    }
                    this.creartScroll(_tableWidth);
                }
                return _column;
            },
            // 创建横向滚动条
            creartScroll(width) {
                this.$nextTick(() => {
                    this.showScroll = true;
                    this.istableHasY = (this.data.length + 1) * 44 > this.$refs.tableContainer.clientHeight;
                    this.scrollbodyWidth = width + 'px';
                    this.judgeTableInView();
                    window.addEventListener('scroll', this.judgeTableInView);
                    this._escapeListenerScroll = () => {
                        window.removeEventListener('scroll', this.judgeTableInView);
                    };
                });
            },
            destoryScroll() {
                this.showScroll = false;
                if (this._escapeListenerScroll) {
                    this._escapeListenerScroll();
                    this._escapeListenerScroll = undefined;
                }
            },
            // 联动横向滚动
            scrollBarMove(event) {
                this.$refs.table.$el.childNodes[2].scrollLeft = event.target.scrollLeft;
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
            judgeTableInView() {
                let _tableContainer = this.$refs.tableContainer;
                let nowTop = this.exploreSize.height - _tableContainer.getBoundingClientRect().top;
                let result = nowTop < _tableContainer.clientHeight && nowTop > 44;
                if (result !== this.isScrollFixed) {
                    this.isScrollFixed = result;
                }
                this.$nextTick(()=>{
                    this.getCurPos();
                });
            },
            getCurPos() {
                let pos = this.$refs.scrollElement.getBoundingClientRect();
                let result;
                if (pos.top > this.exploreSize.height) {
                    result = '我在下面';
                } else if (pos.bottom < 0) {
                    result = '我在上面';
                } else if (pos.left > this.exploreSize.width) {
                    result = '我在右边';
                } else if (pos.right < 0) {
                    result = '我在左边';
                } else {
                    result = '我在当前屏';
                }
                console.log('判断滚动条是否在视窗内', result);
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
        }
    }
</script>

<style>
    .scroll {
        overflow-x: auto;
        overflow-y: hidden;
        position: absolute;
        z-index: 222;
        bottom: 0px;
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
</style>
