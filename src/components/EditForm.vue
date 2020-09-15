<template>
    <div style="padding: 0 15px 0 10px; font-size: 14px; width: 200px; margin-top: 2em">
        <div>
            <strong style="color: white">Edit Form:</strong>
        </div>
        <hr/>
        <div v-if="cellData && cellData.value">
            <div style="display: block;padding-top: 5px; align-items: baseline">
                <div style="width: 150px; color: white">
                    Name:
                </div>
                <div style="flex: auto">
                    <input @focus="$event.target.select()" type="text" @input="debounce_change"
                           v-model="cellData.value.name"/>
                </div>
            </div>
            <div style="display: block;padding-top: 5px; align-items: baseline">
                <div style="width: 150px; color: white">
                    Type:
                </div>
                <div style="flex: auto">
                    <input @focus="$event.target.select()" type="text" @input="debounce_change"
                           v-model="cellData.value.type"/>
                </div>
            </div>
        </div>
        <div style="text-align: center; color: white; padding-top: 5px" v-else>
            No object selected
        </div>
        <hr/>
    </div>
</template>

<script>
    import debounce from '../helpers/debounce'

    export default {
        name: 'EditForm',
        props: {
            cellData: {
                type: Object
            }
        },
        methods: {
            // устраняем частые запросы
            debounce_change: debounce(function () {
                // сообщаем об изменениях
                this.$emit('change', this.cellData.value);
            }, 200)
        }
    }
</script>
