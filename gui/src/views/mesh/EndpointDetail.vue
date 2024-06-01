<script setup>
import { ref,onActivated,watch, computed } from "vue";
import { useRouter } from 'vue-router'
import PipyProxyService from '@/service/PipyProxyService';
import MeshSelector from './common/MeshSelector.vue';
import Services from './Services.vue';
import Ports from './Ports.vue';
import { useStore } from 'vuex';
import { useConfirm } from "primevue/useconfirm";
import freeSvg from "@/assets/img/free.svg";
import dayjs from 'dayjs';
import relativeTime from 'dayjs/plugin/relativeTime';
dayjs.extend(relativeTime)

const props = defineProps(['ep']);
const emits = defineEmits(['back']);
const store = useStore();
const router = useRouter();
const pipyProxyService = new PipyProxyService();
const confirm = useConfirm();
const loading = ref(false);
const loader = ref(false);
const status = ref({});
const endpoints = ref([]);

const isChat = computed(() => store.getters['account/isChat']);
const meshes = computed(() => {
	return store.getters['account/meshes']
});
const selectedMesh = computed(() => {
	return store.getters["account/selectedMesh"]
});
const timeago = computed(() => (ts) => {
	let label = "Last heartbeat: ";
	if(ts>0){
		const date = new Date(ts);
		return label + dayjs(date).fromNow();
	} else {
		return label + "None";
	}
})
onActivated(()=>{
	getEndpoints();
})
const getEndpoints = () => {
	loading.value = true;
	loader.value = true;
	pipyProxyService.getEndpoints(selectedMesh.value?.name)
		.then(res => {
			console.log("Endpoints:")
			console.log(res)
			loading.value = false;
			setTimeout(() => {
				loader.value = false;
			},2000)
			endpoints.value = res || [];
			endpoints.value.forEach((ep,ei)=>{
				ep.key = `${ei}`;
				ep.index = ei;
				ep.type = "ep";
				ep.label = ep?.name;
				ep.leaf = false;
				ep.loading = false;
			});
		})
		.catch(err => console.log('Request Failed', err)); 
}

const typing = ref('');

watch(()=>selectedMesh,()=>{
	if(selectedMesh.value){
		getEndpoints();
	}
},{
	deep:true,
	immediate:true
})


const expandNode = ref();
const expand = (node) => {
	if (!!props.ep) {
		node.loading = true;
		node.children = [];
		expandNode.value = node;
		/*
		 * get services
		 */
		pipyProxyService.getServices({
			mesh:selectedMesh.value?.name,
			ep:props.ep?.id
		})
			.then(res => {
				const _children = res;
				_children.forEach((service,sid)=>{
					service.type = "service";
				});
				node.loading = false;
			})
			.catch(err => {
				node.loading = false;
				console.log('Request Failed', err)
			}); 

		/*
		 * get ports
		 */
		pipyProxyService.getPorts({
			mesh:selectedMesh.value?.name,
			ep:props.ep?.id
		})
			.then(res => {
				const _children = res;
				_children.forEach((port,pid)=>{
					port.type = "port";
				});
			})
			.catch(err => console.log('Request Failed', err)); 
	}
};

const back = () => {
	emits('back')
}
const go = (path) => {
	router.push(path);
}
</script>

<template>
	
	<div class="min-h-screen surface-ground">
	<AppHeader :back="back">
			<template #center>
				<Status v-if="!!props.ep" :run="props.ep.online" :tip="timeago(props.ep.heartbeat)"  style=""/>
				<b v-if="!!props.ep">{{ props.ep.label|| 'Unknow EP' }} <span v-if="!!props.ep.username">({{props.ep.username}})</span></b>
				<span v-else>Loading...</span>
				
			</template>
	
			<template #end> 
				<span v-if="!!props.ep" class="mr-2 relative" style="top: -1px;"><Tag severity="contrast" >{{props.ep.isLocal?'Local':'Remote'}}</Tag></span>
				<Button v-if="isChat" icon="pi pi-comment" label="Chat" @click="go('/message/list')"/>
			</template>
	</AppHeader>
	<div class="text-center">
		<TabView class="" v-model:activeIndex="active">
			<TabPanel>
				<template #header>
					<div>
						<i class="pi pi-server mr-2" />Services
					</div>
				</template>
				<Services :embed="true" :ep="props.ep.id"/>
			</TabPanel>
			<TabPanel>
				<template #header>
					<div>
						<i class="pi pi-bullseye mr-2" />Ports
					</div>
				</template>
				<Ports :embed="true" :ep="props.ep.id"/>
			</TabPanel>
		</TabView>
	</div>
	</div>
</template>

<style scoped lang="scss">
:deep(.p-dataview-content) {
  background-color: transparent !important;
}
:deep(.p-tabview-nav),
:deep(.p-tabview-panels),
:deep(.p-tabview-nav-link){
	background: transparent !important;
}
:deep(.p-tabview-panels){
	padding: 0;
}
</style>