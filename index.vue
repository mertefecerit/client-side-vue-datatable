<template>
    <div v-if="records.length > 0" class="overflow-auto bg-white p-2 rounded">
        <div class="flex justify-between mb-2">
            <div class="flex gap-2 items-center">
                <button @click.prevent="$emit('onCreate')" class="p-2 bg-gray-500 hover:bg-gray-600 text-white px-4 py-2 focus:outline-0 rounded"><i class="fas fa-plus mr-2"></i>Yeni Kayıt</button>
                <input v-model="filterInputValue" @keyup.prevent="filterRecords" type="text" placeholder="Kayıt Ara" class="p-2 bg-gray-100 border border-gray-200 focus:outline-0 rounded">
            </div>
            <div class="flex gap-2 items-center">
                <button @click="hiddenColumns.length = 0" class="py-2 px-4 bg-gray-200 hover:bg-gray-300 text-gray-500 rounded"><i class="fas fa-eye mr-2"></i>Tüm Kolonları Göster</button>
                <button @click.prevent="getExcel" class="py-2 px-4 bg-gray-200 hover:bg-gray-300 text-gray-500 focus:outline-0 rounded"><i class="fas fa-arrow-down-long"></i><i class="fas fa-file-excel mr-2"></i>Excel</button>
                <select @change.prevent="chunkRecords" v-model="perPageRecordNumber" class="p-2 border bg-gray-100 focus:outline-0 rounded">
                    <option v-for="val in perPageRecordNumbers" :value="val" :key="val">{{val}}</option>
                </select>
            </div>
        </div>
        <div class="rounded overflow-hidden border mb-2">
            <table class="table-auto w-full text-center">
                <thead class="bg-gray-300 text-gray-700">
                <tr>
                    <th v-if="processColumn"></th>
                    <th class="p-2" :class="{'hidden':hiddenColumns.includes(columnName.real)}" v-for="columnName in columnNames" :key="columnName">
                        <div class="flex gap-2 items-center justify-center">
                            <i @click.prevent="hiddenColumns.push(columnName.real)" class="fas fa-eye-slash hover:text-white cursor-pointer"></i>
                            <span @click="sortRecords(columnName.real)" class="hover:text-white cursor-pointer">
                                {{columnName.name}}
                                <i v-if="sortDirection === 'asc' && sortColumnName === columnName.real" class="fas fa-sort-down ml-1"></i>
                                <i v-else-if="sortDirection === 'desc' && sortColumnName === columnName.real" class="fas fa-sort-up ml-1"></i>
                            </span>
                        </div>
                    </th>
                </tr>
                </thead>
                <tbody class="divide-y divide-gray-100">
                <tr v-for="(record, index) in datatableRecords" :class="{'bg-gray-50':index % 2 !== 0}" :key="record">
                    <td v-if="processColumn" class="p-2 text-left">
                        <Dropdown classes="w-44 p-2 border">
                            <span class="inline-block p-2 rounded bg-gray-500 hover:bg-gray-600 text-white"><i class="fas fa-list"></i></span>
                            <template v-slot:items>
                                <ul class="text-left">
                                    <li v-for="item in menuItems" :key="item">
                                        <a @click.prevent="$emit(item.emitName, record)" href="#" :class="item.classes"><i :class="item.icon"></i>{{ item.name }}</a>
                                    </li>
                                </ul>
                            </template>
                        </Dropdown>
                    </td>
                    <td :class="{'hidden':hiddenColumns.includes(key)}" class="p-2" v-for="key in Object.keys(record)" v-show="!exceptColumns.includes(key)" :key="key">{{record[key]}}</td>
                </tr>
                </tbody>
            </table>
        </div>
        <div class="flex justify-between items-center">
            <span class="text-gray-500">Toplam Kayıt : {{records.length}}</span>
            <div class="rounded overflow-hidden">
                <button :disabled="currentPage === 0" @click.prevent="previousPage" class="p-2 bg-gray-500 text-white hover:bg-gray-600 transition-all disabled:cursor-not-allowed"><i class="fas fa-arrow-alt-circle-left mr-2"></i>Geri</button>
                <button class="p-2 bg-gray-500 text-white">{{(currentPage + 1)}} / {{pageCount}}</button>
                <button :disabled="(currentPage+1) === pageCount" @click.prevent="nextPage" class="p-2 bg-gray-500 text-white hover:bg-gray-600 transition-all disabled:cursor-not-allowed">İleri<i class="fas fa-arrow-alt-circle-right ml-2"></i></button>
            </div>
        </div>
    </div>
    <div v-else class="flex flex-col gap-4 items-center">
        <button @click.prevent="$emit('onCreate')" class="p-2 bg-gray-500 hover:bg-gray-600 text-white px-4 py-2 focus:outline-0 rounded"><i class="fas fa-plus mr-2"></i>Yeni Kayıt</button>
        <Alert>
            <span>No records here yet</span>
        </Alert>
    </div>
</template>

<script setup>
    import {ref, watch, inject} from "vue";
    import Dropdown from "./Dropdown";
    import Alert from "./Alert";

    const props = defineProps({
        menuItems:{
            default:[],
            type:Array
        },
        processColumn:{
            default: false,
            type: Boolean
        },
        records:{
            required: true,
            type: Array
        },
        columnNames:{
            required: true,
            type: Array
        },
        exceptColumns:{
            default: ['id','deleted_at','created_at','updated_at'],
            type: Array
        },
        exportFileName:{
            default:"records",
            type: String
        }
    });
    const emit = defineEmits(['datatableNewRecord'])

    // Init comp. needed reactive variable section
    const perPageRecordNumbers = ref([10,25,50,100]);
    const perPageRecordNumber = ref(10);
    const currentPage = ref(0);
    const hiddenColumns = ref([]);
    const sortDirection = ref();
    const sortColumnName = ref();
    const filterInputValue = ref();
    const datatableRecords = ref([]);
    const pageCount = ref(0);
    let allRecords = [];
    // Init comp. needed reactive variable section

    // Watch props.records and calculate data
    watch(props, () => {
        // get all records to new array
        allRecords = [...props.records];
        //calculate totalPage count
        pageCount.value = Math.ceil(allRecords.length / perPageRecordNumber.value);
        // set first page of datatable
        datatableRecords.value = _.chunk(allRecords,perPageRecordNumber.value)[0];
    },{deep:true})
    // Watch props.records and calculate data

    // PreviousPage or next button function
    const previousPage = () => {
        if(currentPage.value >= 1){
            datatableRecords.value = _.chunk(allRecords,perPageRecordNumber.value)[currentPage.value - 1]
            currentPage.value--
        }

    }
    const nextPage = () => {
        if ((currentPage.value + 1) < pageCount.value){
            datatableRecords.value = _.chunk(allRecords,perPageRecordNumber.value)[currentPage.value + 1]
            currentPage.value++
        }
    }
    // PreviousPage or next button function

    // Show pieces of records method
    const chunkRecords = () => {
        pageCount.value = Math.ceil(allRecords.length / perPageRecordNumber.value);
        datatableRecords.value = _.chunk(allRecords,perPageRecordNumber.value)[0];
    }
    // Show pieces of records method

    // Filter records inputbox method
    const filterRecords = () => {
        if(!filterInputValue.value){
            allRecords = props.records;
            pageCount.value = Math.ceil(props.records.length / perPageRecordNumber.value);
            datatableRecords.value = _.chunk(props.records,perPageRecordNumber.value)[0];
        }else{
            allRecords = props.records.filter(item => {
                return JSON.stringify(item).toLocaleLowerCase().indexOf(filterInputValue.value.toLocaleLowerCase()) > -1;
            });
            pageCount.value = Math.ceil(allRecords.length / perPageRecordNumber.value);
            datatableRecords.value = _.chunk(allRecords,perPageRecordNumber.value)[0];
        }
    }
    // Filter records inputbox method

    // Export excel file section
    const xlsx = inject('xlsx');
    const getExcel = () => {
        const worksheet = xlsx.utils.json_to_sheet(allRecords);
        const workbook = xlsx.utils.book_new();
        xlsx.utils.book_append_sheet(workbook, worksheet, "Records");
        xlsx.writeFile(workbook, props.exportFileName+'.xlsx');
    }
    // Export excel file section

    // sortRecords function
    const sortRecords = (columnName) => {
        sortColumnName.value = columnName;
        if (sortDirection.value === 'asc'){
            sortDirection.value = 'desc';
        }else{
            sortDirection.value = 'asc'
        }
        allRecords = _.orderBy(allRecords,[columnName],[sortDirection.value]);
        datatableRecords.value = _.chunk(allRecords,perPageRecordNumber.value)[0];
    }
    // sortRecords function
</script>

<style scoped>

</style>
