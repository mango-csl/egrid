<!--<template>-->
<!--<span>{{row.selected + '&#45;&#45; '+row._edit}}</span>-->
<!--</template>-->
<!--&lt;!&ndash;<button @click="todo">{{ row._edit ? 'Done' : 'Edit' }}</button>&ndash;&gt;-->
<!--&lt;!&ndash;<el-radio v-model="row.selected" :label="true" @change="todo">{{row.selected + '&#45;&#45; '+row._edit}}</el-radio>&ndash;&gt;-->
<!--<script>-->
<!--export default {-->
<!--props: ['row', 'column'],-->

<!--methods: {-->
<!--todo () {-->
<!--console.log('row-edit', this.row);-->
<!--this.$emit('row-edit', this.row);-->
<!--}-->
<!--}-->
<!--}-->
<!--</script>-->

<template>
    <button @click="todo">{{ row.selected ? 'Done' : 'Edit' }}</button>
</template>

<script>
    export default {
        props: ['row', 'column'],

        methods: {
            todo() {
                this.$emit('row-edit', this.row)
                this.row.selected = !this.row.selected;
                // this.$set(this.row, '_edit', !this.row._edit)
            }
        }
    }
</script>

