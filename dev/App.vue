<template>
    <!--<div id="app">-->
    <!--<button @click="clearSelection">添加数据</button>-->
    <!--<button @click="console">还原数据</button>-->
    <!--<egrid ref="egrid"-->
    <!--v-bind="{rowKey:'id'}"-->
    <!--:column-type="['tree','selection','index']"-->
    <!--:column-type-props="columnTypeProps"-->
    <!--:column-key-map="{ label: 'name' }"-->
    <!--:data="data"-->
    <!--:columns="columns"-->
    <!--:columns-schema="columnsSchema"-->
    <!--@selection-change="selectionChange">-->
    <!--&lt;!&ndash;:columns-handler="columnsHandler"&ndash;&gt;-->
    <!--<template slot="expand" slot-scope="{ row }">-->
    <!--<section class="expand-detail">-->
    <!--<div v-for="col in columns" :key="col.label">-->
    <!--{{ col.name }}：{{ row[col.prop] }}-->
    <!--</div>-->
    <!--</section>-->
    <!--</template>-->
    <!--</egrid>-->
    <!--</div>-->
    <div id="app">
        <button @click="clearSelection">添加数据11</button>
        <button @click="console">还原数据</button>
        <egrid :data="data"
               :columns="columns"
               @row-click="selectionChange">
        </egrid>
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
                            // row.hasChild = !row.hasChild;
                            console.log('row-edit', row);
                        }
                    }
                },
                {
                    name: '操作2', prop: 'selected',
                    propsHandler({col, row}) {
                        return {selected: row[col.prop]}
                    },
                    component: Vue.extend({
                        props: ['selected'],
                        // render(h) {
                        //     return h('span', {
                        //         style: {color: '#20A0FF'}
                        //     }, this.selected)
                        // }
                        render() {
                            return <span>{this.selected}</span>
                        }
                    })
                }
            );
            return {
                radio2: 3,
                data: _data.filter(f => f['parent_id'] == null),
                columns: _columns,
                // columnsSchema: {
                //     '测试列': {
                //         // width: 'auto',
                //         // propsHandler 可用于转换传给自定义组件的 props
                //         propsHandler({col, row}) {
                //             return {address: row[col.prop]}
                //         },
                //         component: Vue.extend({
                //             props: ['address'],
                //             render(h) {
                //                 return h('span', {
                //                     style: {color: '#20A0FF'}
                //                 }, this.address)
                //             }
                //         })
                //     },
                //     '邮编': {
                //         formater(row, col) {
                //             return row[col.prop] + '2333'
                //         }
                //     }
                // },
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
                // this.selecetedRows = rows;
                // for(let item of this.data){
                //     item.selected = "false";
                // }
                // rows.selected = !rows.selected;
                console.log('selected = ', rows);
            },
            clearSelection() {
                // const {egrid} = this.$refs
                // if (egrid && egrid.clearSelection) {
                //     egrid.clearSelection()
                // }
                let _Data = Data.data.slice(0);
                for (let i = 0; i < 5; i++) {
                    _Data.push(..._Data)
                }
                this.data = _Data;
                console.log('Data.data = ', this.data);
            },
            console() {
                // const {egrid} = this.$refs
                // if (egrid && egrid.clearSelection) {
                //     egrid.clearSelection()
                // }
                this.data = Data.data.slice(0);
                console.log('Data.data = ', this.data);
            }
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
