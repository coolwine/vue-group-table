<template>
    <div class="my-table-fixed my-cursor-pointer table-responsive" style="min-height: 300px;">
        <!--<div class="mb-3">-->
            <!--<b-badge v-for="key in table.groups"-->
                     <!--variant="warning" class="mr-3">{{ key }}-->
            <!--</b-badge>-->
        <!--</div>-->

        <table class="table b-table table-hover table-sm table table-striped my-table-group">
            <thead>
            <tr>
                <th v-for="field in table.fields"
                    :data-field="field.name"
                    :draggable="draggable(field)"
                    @dragstart='dragstart' @dragover="dragover"
                    @drop="drop">
                    <span v-html="field.name" :style="tableHeadStyle(field)"></span>
                    <b-dropdown size="sm" class="my-group-button" variant="link" no-caret>

                        <template slot="button-content">
                            <!--&#x1f50d;<span class="sr-only">Search</span>-->
                            <i class="icon-menu icons"></i>▼
                        </template>

                        <!-- FILTER -->
                        <b-dropdown-item :exact="true" :append="true" >
                            <b-form-input v-model="field.filter" debounce="200"
                                          @change="makeGroup"
                                          @keydown.enter="makeGroup"
                                          @click.native.stop placeholder="FILTER">
                            </b-form-input>
                            <b-badge small style="position: absolute;top: 10px;right: 3px;"
                                     @click.native.stop
                                     @click="clearFilter(field)"
                                     variant="danger">X</b-badge>
                        </b-dropdown-item>

                        <!-- Order Alpha -->
                        <b-dropdown-item-button @click="groupOrderChange('alpha', field)">
                            <span v-if='field.orderAsc'>
                                <i class="fa fa-sort-amount-asc"></i>Asc (alpha)
                            </span>
                            <span v-else>
                                <i class="fa fa-sort-amount-desc"></i>Desc (alpha)
                            </span>
                        </b-dropdown-item-button>

                        <!-- GROUP -->
                        <b-dropdown-header>Group</b-dropdown-header>
                        <b-dropdown-item-button @click="makeGroup(field)">
                            <i class="fa fa-object-group"></i> {{ field.flagGroup ? 'UnGroup' : 'Group ' }}
                        </b-dropdown-item-button>
                        <!--<template v-if="field.flagGroup">-->
                            <!--&lt;!&ndash; Group Order Cnt &ndash;&gt;-->
                            <!--&lt;!&ndash;<b-dropdown-header>Group Order</b-dropdown-header>&ndash;&gt;-->
                            <!--<b-dropdown-item-button @click="groupOrderChange('cnt', field)">-->
                                <!--<span v-if='field.orderAsc'>-->
                                    <!--<i class="fa fa-sort-amount-asc"></i>Asc (cnt)-->
                                <!--</span>-->
                                <!--<span v-else>-->
                                    <!--<i class="fa fa-sort-amount-desc"></i>Desc (cnt)-->
                                <!--</span>-->
                            <!--</b-dropdown-item-button>-->
                        <!--</template>-->
                    </b-dropdown>
                </th>
            </tr>
            </thead>

            <!-- Group 적용상태 -->
            <tbody v-if="table.groups.length">
                <template v-for="rootItems in table.rows">
                    <template v-for="items in rootItems">
                        <tr>
                            <td v-for="n in emptyTrCnt(items)"></td>
                            <td @click="toggleDetails(items)"
                                class="header"
                                :colspan="table.fields.length - emptyTrCnt(items)">
                                <i v-if="items.rows"
                                   :class='getClass("detailToggle", items)'>
                                </i>
                                <span>{{ items.field }}</span>
                                <small class="text-muted pl-1" v-if="items.rows">({{ items.rows.length }})</small>
                            </td>
                        </tr>
                        <tr v-if="items.rows && items.show"
                            v-for="row in items.rows"
                            class="rows">
                            <td v-for="field in table.fields">
                                <slot :name="field.name" :item="row" :value="row[field.name]">{{ row[field.name] }}</slot>
                            </td>
                        </tr>
                    </template>
                </template>
            </tbody>
            <!-- Group 미적용상태 -->
            <tbody v-else>
                <tr v-for="item in getCalculateList">
                    <td v-for="field in table.fields">
                        <!--@click="trClick"-->
                        <slot :name="field.name" :item="item" :value="item[field.name]">{{ item[field.name] }}</slot>
                    </td>
                </tr>
            </tbody>
        </table>
    </div>
</template>

<script>
import 'bootstrap/dist/css/bootstrap.css'
import 'bootstrap-vue/dist/bootstrap-vue.css'
export default {
    name: "GroupTable",
    props: {
        items: {
            type: Array,
            default(){
                return []
            }
        },
        fields: Array,
    },
    mounted() {
    },
    watch: {
        items(items){
            this.table.fields = [];
            this.table.groups = [];
            if(items && items.length){
                Object.keys(items[0]).forEach(field=>{
                    this.table.fields.push({
                        name: field, flagGroup: false
                    });
                });
            }
        }
    },
    computed:{
        getCalculateList(){
            let items = [];
            /** Filter 적용 **/
            this.items.forEach(item=>{
                let exists = true;
                this.table.fields.forEach(field=>{
                    if(field.filter){
                        let _exists = this.groupFilterExists(item[field.name], field.filter);
                        if(_exists == false){
                            exists = _exists;
                        }
                    }
                });
                if(exists){
                    items.push(item);
                }
            });

            /** Order 적용 **/
            this.table.fields.forEach(field=>{
                this.listOrder(field, items);
            });
            return items;
        },
        getGroupField(){
            let fields = new Map();
            this.table.groups.map(field=>{
                fields.set(field.name, field);
            });
            this.table.fields.forEach(field=>{
                if(!fields.has(field.name)){
                    field.flagGroup = false;
                    fields.set(field.name, field);
                }
            });
            return [...fields.values()];
        },
    },
    data(){
        return {
            table: {
                groups: [], // {name: ''}
                groupList: [], // {key: '', list: [],}
                fields: [
                    // {name: '', flagGroup: false}
                ],
                rows: [],
            },
        }
    },
    methods: {
        /** 그룹 필드 구성 **/
        makeFieldGroup(field = {}){
            if(field.hasOwnProperty('flagGroup')){
                field.flagGroup = !field.flagGroup;
            }
            let groups = [],
                commonData = {
                    orderType: 'cnt',
                    orderAsc: false,
                    filter: '',
                };
            this.table.fields.forEach(field=>{
                if(field.flagGroup){
                    let _group = Object.assign({}, commonData, field);
                    groups.push(_group);
                }
            });
            this.table.groups = groups;
            this.table.fields = this.getGroupField;
            return JSON.parse(JSON.stringify(groups));
        },
        makeGroup(field){
            let groupInfo = this.makeFieldGroup(field);
            if(groupInfo.length){
                //this.$notify(Object.assign(constValueService.notify.warnEmpty, {text: '그룹 데이터 생성중..'}));
                /** 루트 그룹 생성 **/
                let firstGroup = groupInfo.shift();
                this.table.groupList = this._makeDepth(firstGroup, this.getCalculateList);
                //this.listOrder(firstGroup.name, this.table.groupList);
                /** 자식 그룹 생성 **/
                groupInfo.forEach(group=>{
                    this.table.groupList.forEach(items=>{
                        this._makeDepth(group, items);
                    })
                });
                this.recursiveListToTableList();
            }
        },
        /** 재귀형태 -> 테이블 구조로 변환 **/
        recursiveListToTableList(){
            this.table.rows = [];
            this.table.groupList.forEach((items, idx)=>{
                let result = [];
                this._makeTable(items, result);
                this.table.rows.push(result);
            });
        },
        /**
         * 테이블에 맞는 형태로 변환
         */
        _makeTable(items, result, depth = 1, headerInfo = {}){
            let header = {};
            if(items.hasOwnProperty('groupName')){
                header = {
                    groupName: items.groupName,
                    field: items.field,
                    show: false,
                    depth,
                };
                result.push(header);
            }else if(!items.hasOwnProperty('groupName')){
                if(headerInfo.rows){
                    headerInfo.rows.push(items);
                }else{
                    headerInfo.rows = [items];
                }
            }

            if(items.hasOwnProperty('list')){
                depth++;
                items.list.forEach(item=>{
                    this._makeTable(item, result, depth, header);
                })
            }
        },
        _makeDepth(groupInfo, items){
            if(items.list){
                let tmp = this._makeDepth(groupInfo, items.list);
                if(tmp.length) items.list = tmp;
            }else{
                let result = [];
                let _filterList = new Map();
                items.forEach(item=>{
                    if(item.list){
                        let tmp = this._makeDepth(groupInfo, item.list);
                        if(tmp && tmp.length) item.list = tmp;
                    }else{
                        if(_filterList.has(item[groupInfo.name])){
                            let _item = _filterList.get(item[groupInfo.name]);
                            if(!item){
                                _item = [item];
                            }else{
                                _item.push(item);
                            }
                            _filterList.set(item[groupInfo.name], _item);
                        }else{
                            _filterList.set(item[groupInfo.name], [item]);
                        }
                    }
                });

                for(let [key, info] of _filterList.entries()){
                    let tmp = {
                        groupName: groupInfo.name,
                        field: key,
                        list: info,
                        show: false,
                    };
                    result.push(tmp);
                }
                return result;
            }
        },
        tableHeadStyle(field){
            let style = {};
            if(field.flagGroup) style.color = 'red';
            return style;
        },
        getClass(mode, obj){
            let _class = [];
            switch(mode){
                case 'detailToggle':
                    _class = ['fa', 'pr-1'];
                    if(obj.show) _class.push('fa-arrow-circle-down');
                    else _class.push('fa-arrow-circle-up');
                    break;
            }
            return _class;
        },
        toggleDetails(items){
            items.show = !items.show;
        },
        emptyTrCnt(items){
            if(items.depth){
                return items.depth - 1;
            }else{
                return 0;
            }
        },
        groupOrderChange(type = 'cnt', field){
            field.orderType = type;
            field.orderAsc = !field.orderAsc;
            this.makeGroup();
        },
        /** 리스트 정렬 **/
        listOrder(field, items){
            switch(field.orderType) {
                case 'alpha':
                    items.sort((a, b)=>{
                        if(field.orderAsc){
                            if(a[field.name] < b[field.name]) { return -1; }
                            if(a[field.name] > b[field.name]) { return 1; }
                        }else{
                            if(a[field.name] < b[field.name]) { return 1; }
                            if(a[field.name] > b[field.name]) { return -1; }
                        }
                        return 0;
                    });
                    break;
                case 'cnt':
                default:
                    items.sort((a, b)=>{
                        let key = (a.hasOwnProperty('list'))?'list':'rows';
                        if(a[key] && b[key]){
                            if(field.orderAsc){
                                return a[key].length - b[key].length;
                            }else{
                                return b[key].length - a[key].length;
                            }
                        }
                    });
                    break;
            }
        },
        clearFilter(field){
            field.filter='';
            this.makeGroup();
        },
        groupFilterExists(text = '', filter = ''){
            if(!filter){
                return true;
            }

            if(typeof text == 'string'){
                return text.toLowerCase().includes(filter.toLowerCase());
            }else{
                return text == filter;
            }
        },
        getFiledInfo(fieldName){
            let field = this.table.fields.filter(field=>field.name == fieldName);
            return (field.length)?field.pop():false;
        },
        draggable(field){
            return !field.flagGroup;
        },
        dragstart(evt){
            evt.dataTransfer.effectAllowed = "move";
            evt.dataTransfer.setData('field', evt.target.dataset.field);
        },
        dragover(evt){
            evt.preventDefault();
        },
        drop(evt){
            let target = (evt.target.tagName !== 'TH')?evt.target.parentNode:evt.target;
            let fieldA = evt.dataTransfer.getData('field');
            let fieldB = target.dataset.field;
            let draggable = target.getAttribute('draggable');
            if(!draggable || draggable == 'false'){
                return;
            }

            let tmpA = {idx: 0, data: {}},
                tmpB = {idx: 0, data: {}};
            for(let idx in this.table.fields){
                if(this.table.fields[idx].name == fieldA){
                    tmpA.idx = idx;
                    tmpA.data = this.table.fields.splice(idx, 1);
                    break;
                }
            }
            for(let idx in this.table.fields){
                if(this.table.fields[idx].name == fieldB){
                    tmpB.idx = idx;
                    tmpB.data = this.table.fields.splice(idx, 1, tmpA.data.pop());
                    break;
                }
            }
            this.table.fields.splice(tmpA.idx, 0, tmpB.data.pop());
        }
    }
}
</script>

<style lang="scss" scoped>
    .my-cursor-pointer {
        cursor: pointer;
    }
    .my-table-fixed {
        display: inline-block;
        overflow-x: auto;
        white-space: nowrap;
    }
    .my-group-button {
        button.dropdown-toggle{
            width: 25px;
            height: 25px;
        }
    }
    .my-table-group {
        text-align:left;
        tbody {
            .header {
                color: #18d631;
            }
            tr.rows {
                border: 2px solid #2d2d2d;
            }
            td.link {
                color:#20a8d8;
                text-decoration: underline;
                background-color: transparent;
                cursor: pointer;
            }
        }
    }
</style>