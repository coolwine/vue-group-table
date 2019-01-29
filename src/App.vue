<template>
    <div id="app">
        <group-table :items="items">
            <template slot="age" slot-scope="row">
                <span>{{ '나이는 : ' + row.value }}</span>
            </template>
        </group-table>
    </div>
</template>

<script>
import testData from '../static/test';
import GroupTable from './components/GroupTable';
console.log(testData);

export default {
    name: 'App',
    components: {
        GroupTable
    },
    mounted() {
        this.items = testData;
    },
    data(){
        return {
            items: [],
        }
    }
}
</script>

<style>
    #app {
        font-family: 'Avenir', Helvetica, Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        text-align: center;
        color: #2c3e50;
        margin-top: 60px;
    }
</style>
