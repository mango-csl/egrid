<template>
    <div id="app">
        <egrid v-bind="{rowKey:'id'}"
               :column-key-map="{ label: 'name' }"
               :data="data"
               :columns="columns"
               :columns-schema="columnsSchema"
               @row-click="selectionChange">
            <!--:columns-handler="columnsHandler"-->
            <!--<template slot="expand" slot-scope="{ row }">-->
            <!--<section class="expand-detail">-->
            <!--<div v-for="col in columns" :key="col.label">-->
            <!--{{ col.name }}：{{ row[col.prop] }}-->
            <!--</div>-->
            <!--</section>-->
            <!--</template>-->
        </egrid>
        <el-button type="text" @click="dialogTableVisible = true">打开嵌套表格的 Dialog</el-button>
        <el-dialog ref="dialog" title="收货地址" :visible.sync="dialogTableVisible" @open>
            <!--:column-type="['tree','selection','index']"-->
            <!--:column-type-props="columnTypeProps"-->
            <egrid v-bind="{rowKey:'id',container:container}"
                   :column-key-map="{ label: 'name' }"
                   :data="data"
                   :columns="columns"
                   :columns-schema="columnsSchema"
                   @row-click="selectionChange">
                <!--:columns-handler="columnsHandler"-->
                <!--<template slot="expand" slot-scope="{ row }">-->
                    <!--<section class="expand-detail">-->
                        <!--<div v-for="col in columns" :key="col.label">-->
                            <!--{{ col.name }}：{{ row[col.prop] }}-->
                        <!--</div>-->
                    <!--</section>-->
                <!--</template>-->
            </egrid>
        </el-dialog>
    </div>
</template>

<script>
    import Vue from 'vue'
    import Egrid from '../src/index'
    import Data from './data'
    import Btn from './cell-btn.vue'

    export default {
        name: 'app',
        data() {
            let _data = Data.data;
            let _columns = Data.columns.splice(0);
            _columns.push({
                // fixed: 'right',
                name: '操作',
                align: 'left',
                component: Btn,
                // listeners 可用于监听自定义组件内部 $emit 出的事件
                listeners: {
                    'row-edit'(row) {
                        row.hasChild = !row.hasChild;
                        console.log('row-edit', row);
                    }
                }
            });
            return {
                dialogTableVisible: false,
                dialogTable: false,
                container:undefined,
                data: _data.filter(f => f['parent_id'] == null),
                columns: _columns,
                // 替换特殊列，配置
                columnTypeProps: {
                    selection: {
                        selectable(row, index) {
                            return index < 2
                        },
                        reserveSelection: true
                    },
                    tree: {
                        childNumKey: 'child_num',
                        remotePromise: (row) => {
                            return new Promise((resolve, reject) => {
                                let _data = Data.data;
                                setTimeout(() => {
                                    resolve(_data)
                                }, 500);
                            })
                        }
                    }
                },
                // columnsProps 用于定义所有 columns 公共的属性，
                // 这里属性可以参考这里： http://element.eleme.io/#/zh-CN/component/table#table-column-attributes
                // columnsProps: {
                //   width: 120,
                //   sortable: true,
                //   // 定义表格列如何渲染
                //   component: Editor
                // },
                // columnsSchema 可以用来单独定义 columns 的某一列，这里的设置会覆盖 columnsProps 的配置属性
                columnsSchema: {
                    '地址': {
                        // width: 'auto',
                        // propsHandler 可用于转换传给自定义组件的 props
                        propsHandler({col, row}) {
                            return {address: row[col.prop]}
                        },
                        component: Vue.extend({
                            props: ['address'],
                            render(h) {
                                return h('span', {
                                    style: {color: '#20A0FF'}
                                }, this.address)
                            }
                        })
                    },
                    '邮编': {
                        formater(row, col) {
                            return row[col.prop] + '2333'
                        }
                    }
                },
                selecetedRows: []
            }
        },
        methods: {
            // columnsHandler 可用于在现有的 columns 进行操作，对 columns 进行增删改，这里新增了操作列
            // columnsHandler(cols) {
            //     return cols.concat({
            //         // fixed: 'right',
            //         label: '操作',
            //         align: 'left',
            //         component: Btn,
            //         // listeners 可用于监听自定义组件内部 $emit 出的事件
            //         listeners: {
            //             'row-edit'(row) {
            //                 console.log('row-edit', row);
            //             }
            //         }
            //     })
            // },
            selectionChange(rows) {
                this.$refs.dialog;
                this.selecetedRows = rows;
                console.log('selected', rows);
            }
        },
        mounted(){
          this.container = this.$refs.dialog.$el;
        },
        components: {Egrid}
    }
</script>

<style>
    #app {
        font-family: 'Avenir', Helvetica, Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        text-align: center;
        color: #2c3e50;
        padding: 50px 60px;
    }

    .expand-detail div {
        text-align: left;
        padding: 4px 0;
    }
</style>
