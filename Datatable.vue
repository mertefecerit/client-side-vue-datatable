<template>
    <div class="overflow-auto">
        <div class="flex justify-between mb-2">
            <div class="flex gap-2 items-center">
                <button @click.prevent="$emit('datatableNewRecord')" class="p-2 bg-gray-500 hover:bg-gray-600 text-white px-4 py-2 focus:outline-0 rounded"><i class="fas fa-plus mr-2"></i>Yeni Kayıt</button>
                <input v-model="filterInputValue" @keyup.prevent="filterRecords" type="text" placeholder="Kayıt Ara" class="p-2 bg-gray-100 border border-gray-200 focus:outline-0 rounded">
            </div>
            <div class="flex gap-2 items-center">
                <button @click="hiddenColumns.length = 0" class="p-2 bg-gray-500 text-white hover:bg-gray-600 rounded"><i class="fas fa-eye mr-2"></i>Tüm Kolonları Göster</button>
                <button @click.prevent="getExcel('current')" class="py-2 px-4 bg-gray-200 hover:bg-gray-300 text-gray-500 focus:outline-0 rounded"><i class="fas fa-arrow-down-long"></i><i class="fas fa-file-excel mr-2"></i>Mevcut</button>
                <button @click.prevent="getExcel('total')" class="py-2 px-4 bg-gray-200 hover:bg-gray-300 text-gray-500 focus:outline-0 rounded"><i class="fas fa-arrow-down-long"></i><i class="fas fa-file-excel mr-2"></i>Tamamı</button>
                <select @change.prevent="chunkRecords" v-model="perPageRecordNumber" class="p-2 border bg-gray-100 focus:outline-0 rounded">
                    <option v-for="val in perPageRecordNumbers" :value="val">{{val}}</option>
                </select>
            </div>
        </div>
        <div class="rounded overflow-hidden border mb-2">
            <table class="table-auto w-full text-center">
                <thead class="bg-gray-300 text-gray-700">
                <tr>
                    <th v-if="processColumn"></th>
                    <th :class="{'hidden':hiddenColumns.includes(columnName.real)}" class="p-2" v-for="columnName in columnNames">
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
                <tr v-for="(record, index) in datatableRecords" :class="{'bg-gray-50':index % 2 !== 0}">
                    <td v-if="processColumn" class="p-2 text-left">
                        <Dropdown classes="w-44 p-2 border">
                            <span class="inline-block p-2 rounded bg-gray-500 hover:bg-gray-600 text-white"><i class="fas fa-list"></i></span>
                            <template v-slot:items>
                                <ul class="text-left">
                                    <li><a @click.prevent="$emit('onEdit', record)" href="#" class="text-sky-500 hover:text-sky-600 inline-block p-2 w-full hover:bg-gray-100 rounded"><i class="fas fa-edit mr-2"></i>Düzenle</a></li>
                                    <li><a @click.prevent="$emit('onDelete', record)" href="#" class="text-red-500 hover:text-red-600 inline-block p-2 w-full hover:bg-gray-100 rounded"><i class="fas fa-trash mr-2"></i>Sil</a></li>
                                </ul>
                            </template>
                        </Dropdown>
                    </td>
                    <td :class="{'hidden':hiddenColumns.includes(key)}" class="p-2" v-for="key in Object.keys(record)" v-show="!exceptColumns.includes(key)">{{record[key]}}</td>
                </tr>
                </tbody>
            </table>
        </div>
        <div class="flex justify-between items-center">
            <span class="text-gray-500">Toplam Kayıt : {{records.length}}</span>
            <div class="rounded overflow-hidden">
                <button @click.prevent="previousPage" class="p-2 bg-gray-500 text-white hover:bg-gray-600 transition-all"><i class="fas fa-arrow-alt-circle-left mr-2"></i>Geri</button>
                <button class="p-2 bg-gray-500 text-white">{{(currentPage + 1)}} / {{pageCount}}</button>
                <button @click.prevent="nextPage" class="p-2 bg-gray-500 text-white hover:bg-gray-600 transition-all">İleri<i class="fas fa-arrow-alt-circle-right ml-2"></i></button>
            </div>
        </div>
    </div>
</template>

<script>
import {computed, ref, watch, inject} from "vue";
import Dropdown from "./Dropdown";

export default {
    name: "Datatable",
    emits:['datatableNewRecord','onEdit','onDelete'],
    components:{
        Dropdown
    },
    props:{
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
    },
    setup(props, context){
        // Init comp. needed reactive variable section
        const perPageRecordNumbers = ref([10,25,50,100]);
        const perPageRecordNumber = ref(10);
        const currentPage = ref(0);
        const hiddenColumns = ref([]);
        const sortDirection = ref();
        const sortColumnName = ref();
        const filterInputValue = ref();
        const pageCount = computed(() => {
            return Math.ceil(props.records.length / perPageRecordNumber.value);
        });
        // Init comp. needed reactive variable section


        const reset = () => {
            perPageRecordNumber.value = 10;
            currentPage.value = 0;
            datatableRecords.value = _.chunk(props.records,perPageRecordNumber.value)[0];
        }


        // Watch props.records
        const datatableRecords = ref([]);
        watch(props, () => {
            perPageRecordNumber.value = 10;
            currentPage.value = 0;
            datatableRecords.value = _.chunk(props.records,perPageRecordNumber.value)[0];
        },{deep:true})
        // Watch props.records



        // PreviousPage or next button function
        const previousPage = () => {
            if(currentPage.value >= 1){
                if (sortColumnName.value){
                    datatableRecords.value = _.orderBy(props.records,[sortColumnName.value],[sortDirection.value]);
                    datatableRecords.value = _.chunk(datatableRecords.value,perPageRecordNumber.value)[currentPage.value - 1]
                }
                else{
                    datatableRecords.value = _.chunk(props.records,perPageRecordNumber.value)[currentPage.value - 1]
                }

                currentPage.value--
            }

        }
        const nextPage = () => {
            if ((currentPage.value + 1) < pageCount.value){
                if (sortColumnName.value){
                    datatableRecords.value = _.orderBy(props.records,[sortColumnName.value],[sortDirection.value]);
                    datatableRecords.value = _.chunk(datatableRecords.value,perPageRecordNumber.value)[currentPage.value + 1]
                }
                else{
                    datatableRecords.value = _.chunk(props.records,perPageRecordNumber.value)[currentPage.value + 1]
                }

                currentPage.value++
            }
        }
        // PreviousPage or next button function



        // Show pieces of records method
        const chunkRecords = () => {
            currentPage.value = 0;
            if (sortColumnName.value){
                datatableRecords.value = _.orderBy(props.records,[sortColumnName.value],[sortDirection.value]);
                datatableRecords.value = _.chunk(datatableRecords.value,perPageRecordNumber.value)[0];
            }else{
                datatableRecords.value = _.chunk(props.records,perPageRecordNumber.value)[0];
            }

        }
        // Show pieces of records method


        // Filter records inputbox method
        const filterRecords = () => {
            if(!filterInputValue.value){
                datatableRecords.value = _.chunk(props.records,perPageRecordNumber.value)[0];
            }else{
                datatableRecords.value = props.records.filter(item => {
                    return JSON.stringify(item).toLocaleLowerCase().indexOf(filterInputValue.value.toLocaleLowerCase()) > -1;
                });
            }
        }
        // Filter records inputbox method



        // Export excel file section
        const xlsx = inject('xlsx');
        const getExcel = (type) => {
            let data = props.records;
            if (type === "current"){
                data = datatableRecords.value;
            }
            const worksheet = xlsx.utils.json_to_sheet(data);
            const workbook = xlsx.utils.book_new();
            xlsx.utils.book_append_sheet(workbook, worksheet, "Records");
            xlsx.writeFile(workbook, props.exportFileName+'.xlsx');
        }
        // Export excel file section




        const sortRecords = (columnName) => {
            reset();
            sortColumnName.value = columnName;
            if (sortDirection.value === 'asc'){
                sortDirection.value = 'desc';
            }else{
                sortDirection.value = 'asc'
            }
            datatableRecords.value = _.orderBy(props.records,[columnName],[sortDirection.value]);
            datatableRecords.value = _.chunk(datatableRecords.value,perPageRecordNumber.value)[0];

        }


        return {
            perPageRecordNumber,
            pageCount,
            datatableRecords,
            perPageRecordNumbers,
            chunkRecords,
            filterRecords,
            previousPage,
            nextPage,
            currentPage,
            hiddenColumns,
            getExcel,
            sortDirection,
            sortRecords,
            sortColumnName,
            filterInputValue
        }
    }
}
</script>

<style scoped>

</style>
