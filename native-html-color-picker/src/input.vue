<template>
	<div class="wrap">
		<input type="color" class="picker" @input="emitValue" :value="value"/>
	</div>
</template>

<script>
	import mixin from "@directus/extension-toolkit/mixins/interface";

	export default {
		mixins: [mixin],
		methods: {
			emitValue(event) {
				const value = event.target.value;
				this.$emit("input", value);
			}
		}
	}
</script>

<style lang="scss" scoped>
	.wrap {
		padding: 3px;
		display: inline-block;
		border: var(--input-border-width) solid var(--lighter-gray);
		border-radius: var(--border-radius);
	}
	.picker {
		width: 80px;
		height: 30px;
		border: none;
		background: none;
	}
</style>
