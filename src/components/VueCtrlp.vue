<template>
	<div v-show="show" class="vue-ctrlp" :aria-hidden="!show" role="modal" :data-open="show">
		<div class="vue-ctrlp__background" @click="onClose" :aria-hidden="true"></div>

		<form class="vue-ctrlp__form">
			<div class="vue-ctrlp__field">
				<input
					ref="input"
					class="vue-ctrlp__search"
					type="search"
					:disabled="searching && disableOnSearch"
					:name="name"
					v-model="searchString"
					@input="doSearch"
					@keydown.esc="onClose"/>
			</div>

			<div class="vue-ctrlp__meta">
				<template v-if="searching">
					<slot name="searching"/>
				</template>

				<template v-if="shouldShowResults">
					<template v-if="hasResults">
						<slot name="results" v-bind="{ results }">
							<ol class="vue-ctrlp__results">
								<li v-for="result in results" :key="result" class="vue-ctrlp__result">
									{{ result }}
								</li>
							</ol>
						</slot>
					</template>

					<template v-else>
						<slot name="noResults">
							No results.
						</slot>
					</template>
				</template>

				<slot name="all" v-bind="{ searching, shouldShowResults, hasResults, results }"/>
			</div>
		</form>
	</div>
</template>

<script>
	import VueTypes from "vue-types"
	import debounce from "lodash.debounce"

	//TODO: Trap focus

	export default {
		name: "vue-ctrlp",
		props: {
			show: VueTypes.bool.def(false),
			search: VueTypes.func.def(str => []),
			preserveState: VueTypes.bool.def(false),
			clearOnSearch: VueTypes.bool.def(true),
			delay: VueTypes.integer.def(0),
			disableOnSearch: VueTypes.bool.def(false),
			name:  VueTypes.string.def("vue-ctrlp"),
		},
		data(){
			return {
				results: [],
				searchString: "",
				searching: false,
				didSearch: false,
			};
		},
		computed: {
			hasResults(){
				return this.results.length > 0;
			},
			shouldShowResults(){
				return this.didSearch || this.hasResults;
			},
			doSearch(){
				if(this.delay === 0)
					return this.onSearch;

				return debounce(this.onSearch, this.delay);
			},
		},
		methods: {
			async onSearch(){
				if(this.searching)
					return;

				if(this.clearOnSearch)
					this.clearState(false);

				this.searching = true;

				try{
					const results = await this.search(this.searchString.trim());
					this.results = results;
					this.onResults(results);
				}catch(e){
					this.onError(e);
				}finally{
					this.didSearch = true;
					this.searching = false;
				}
			},

			onResults(results){
				this.$emit("results", results);
			},
			onError(e){
				this.$emit("error", e);
			},
			onClose(){
				this.$emit("close");
			},
			grabFocus(){
				setTimeout(() => {
					const { input } = this.$refs;
					//TODO: Focus after last character
					input.focus();
				}, 50);
			},

			clearState(shouldResetString = true){
				this.results = [];
				this.searching = false;

				if(shouldResetString)
					this.searchString = "";
			},
			onOpen(){
				if(!this.preserveState && this.hasResults)
					this.clearState();

				this.grabFocus();
			},
			onLeave(){
				if(!this.preserveState)
					this.clearState();
			},
		},
		mounted(){
			if(this.show)
				this.onOpen();
		},
		watch: {
			show(newVal, oldVal){
				if(oldVal === newVal)
					return;

				if(newVal)
					this.onOpen();
				else
					this.onLeave();
			},
			searchString(newVal, oldVal){
				if(oldVal === newVal)
					return;

				if(newVal.length === 0)
					this.didSearch = false;
			},
		},
	}
</script>