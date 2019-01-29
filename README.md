# vue-group-table

> group-table

## Usage

``` bash
npm install vue-group-table
yarn add vue-group-table
```

# How to use
```html
<group-table :items="items">
    <!-- 확장하고 싶은 컬럼 slot -->
    <template slot="age" slot-scope="row">
        <span>{{ '나이는 : ' + row.value }}</span>
    </template>
</group-table>
```
