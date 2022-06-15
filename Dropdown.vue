<template>
    <div ref="dropdown">
        <a href="#" @click.prevent="onToggle">
            <slot></slot>
        </a>
        <transition
            enter-active-class="transition duration-100 ease-out"
            enter-from-class="transform scale-95 opacity-0"
            enter-to-class="transform scale-100 opacity-100"
            leave-active-class="transition duration-75 ease-in"
            leave-from-class="transform scale-100 opacity-100"
            leave-to-class="transform scale-95 opacity-0">
            <div class="overflow-hidden absolute mt-1 bg-white rounded shadow z-40" :class="classes" v-if="status">
                <slot name="items"></slot>
            </div>
        </transition>
    </div>
</template>

<script>
import {ref} from "vue";

export default {
    name: "Dropdown",
    props:['classes'],
    setup(){
        const status = ref(false);
        const dropdown = ref();
        const checkOutsideClick = (e) => {
            if (!dropdown.value.contains(e.target)){
                status.value = false;
                document.removeEventListener("click",checkOutsideClick)
            }
        }
        const onToggle = () => {
            status.value = !status.value
            if (status.value){
                document.addEventListener("click",checkOutsideClick)
            }else{
                document.removeEventListener("click",checkOutsideClick)
            }
        }
        return {
            status,
            dropdown,
            onToggle
        }
    }
}
</script>

<style scoped>

</style>
