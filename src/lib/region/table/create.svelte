<script lang="ts">
	import * as Dialog from '$lib/components/ui/dialog';

	import { Input } from '$lib/components/ui/input';
	import { Button } from '$lib/components/ui/button';
	import { PlusCircle } from 'lucide-svelte';

	// import { CollectionStore } from 'pocketbase-store';
	import { writable, derived, type Writable } from 'svelte/store';
	import { toastApi, shield, shrinkString } from '$lib/utils';
	import { icon } from '$lib/config';
	// import type { RegionsResponse, TeamsResponse } from '$lib/types';
	// import { isOwner } from '$lib/store/team_store';

	import { regionsStore, teamStore } from '$lib/stores';

	const regionDialogOpen = writable(false);
	const regionForm = writable({
		name: ''
	});

	async function createRegion() {
		const result = regionsStore.create({
			...$regionForm,
			team: $teamStore?.id as string
		});
		regionDialogOpen.set(false);
		await result;
	}
</script>

<!-- {#if $isOwner} -->
<Dialog.Root bind:open={$regionDialogOpen}>
	<Dialog.Trigger>
		{#snippet child({ props })}
			<Button
				{...props}
				class="relative h-[130px] w-full animate-fade-in-up text-center max-lg:h-[80px] sm:col-auto"
				variant="outline"
			>
				Create region
				<PlusCircle class={icon.left} />
			</Button>
		{/snippet}
	</Dialog.Trigger>
	<Dialog.Content class="sm:max-w-[425px]" trapFocus={false}>
		<Dialog.Header>
			<Dialog.Title>Create region</Dialog.Title>
			<Dialog.Description>Enter your new region name.</Dialog.Description>
		</Dialog.Header>
		<form
			class="flex gap-2"
			on:submit|preventDefault={toastApi.execAsync(
				createRegion,
				`Region ${$regionForm.name} created.`,
				`Failed to create region ${$regionForm.name}, my be this region already exists.`
			)}
		>
			<div class="grid w-full grid-cols-1 gap-2">
				<Input id="name" placeholder="Region name" bind:value={$regionForm.name} required />
			</div>
			<Button type="submit" class="h-full">Create <PlusCircle class={icon.left} /></Button>
		</form>
	</Dialog.Content>
</Dialog.Root>
