<script lang="ts">
	import { Button } from '$lib/components/ui/button';
	import CodeMirror from 'svelte-codemirror-editor';
	import { scriptStore } from '$lib/stores';
	import { githubLight } from '@fsegurai/codemirror-theme-github-light';
	import { githubDark } from '@fsegurai/codemirror-theme-github-dark';
	import { theme } from 'theme-selector';
	import { python, globalCompletion, localCompletionSource } from '@codemirror/lang-python';
	import { syntaxTree } from '@codemirror/language';
	import { linter, type Diagnostic } from '@codemirror/lint';
	import { autocompletion, CompletionContext } from '@codemirror/autocomplete';
	import type { EditorView } from '@codemirror/view';
	import { writable, derived } from 'svelte/store';
	import { onMount } from 'svelte';
	import { beforeNavigate } from '$app/navigation';
	import { pb } from '$lib/pocketbase/client';
	import { toastApi } from '$lib/utils';
	import { toast } from 'svelte-sonner';
	import { browser } from '$app/environment';
	import { Input } from '$lib/components/ui/input';
	import { Textarea } from '$lib/components/ui/textarea';

	let customCompletions: any[] = [];

	export const scriptName = writable($scriptStore?.name);
	export const scriptDescription = writable($scriptStore?.description);
	export const scriptExecutable = writable($scriptStore?.executable);

	export let hasCriticalErrors = writable(false);
	export let editable = true;

	let view: EditorView;

	const hasChanges = derived([scriptExecutable, scriptStore, scriptName, scriptDescription], ([$value, $scriptStore, $scriptName, $scriptDescription]) => {
		return (
			$scriptStore?.executable !== $value ||
			$scriptStore?.description !== $scriptDescription ||
			$scriptStore?.name !== $scriptName
		);
	});
	
	const handleBeforeUnload = (e: BeforeUnloadEvent) => {
		if ($hasChanges) {
			e.preventDefault();
			e.returnValue = '';
		}
	};

	const handleKeyDown = async (e: KeyboardEvent) => {
		if ((e.ctrlKey || e.metaKey) && e.key === 's') {
			e.preventDefault();
			if (!$hasCriticalErrors) {
				await toastApi.execAsync(
					updateScript,
					`Script ${$scriptStore?.name} updated.`,
					`Failed to update script ${$scriptStore?.name}, check syntax error.`
				);
			} else {
				toast.error('Cannot save due to syntax errors.');
			}
		}
	};

	// @ts-ignore
	onMount(async () => {
		try {
			const response = await fetch('/controlhub_completions.json');
			if (!response.ok) {
				throw new Error(`Failed to load completions: ${response.statusText}`);
			}
			const data = await response.json();
			// @ts-ignore
			customCompletions = data.map((item) => ({
				label: item.label.slice(0, item.label.length - 2),
				...item
			}));
			console.log('Loaded completions:', customCompletions); // Debugging
		} catch (error) {
			console.error('Error loading completions:', error);
		}

		window.addEventListener('beforeunload', handleBeforeUnload);
		window.addEventListener('keydown', handleKeyDown);

		return async () => {
			window.removeEventListener('beforeunload', handleBeforeUnload);
		window.removeEventListener('keydown', handleKeyDown);
		console.log('Editor component destroyed, event listeners removed');
		}
	});


	// SvelteKit navigation interception
	beforeNavigate(({ cancel }) => {
		if ($hasChanges) {
			const shouldSave = confirm(
				'You have unsaved changes. Do you want to save them before leaving?'
			);

			if (shouldSave) {
				if (!$hasCriticalErrors) {
					updateScript()
						.then(() => {
						})
						.catch((err) => {
							console.error('Error while saving:', err);
							if (!confirm('Error while saving. Do you want to continue without saving?')) {
								cancel();
							}
						});
				} else {
					alert('Cannot save script due to syntax errors.');
					if (!confirm('Do you want to continue without saving?')) {
						cancel();
					}
				}
			} else if (!confirm('Are you sure you want to leave the page without saving changes?')) {
				cancel();
			}
		}
	});

	const importRegex = /^import\s+(\w*)$/;
	const fromImportRegex = /^from\s+(\w+)\s+import\s+(\w*)$/;
	const fromRegex = /^from\s+(\w*)$/;

	const controlHubModuleCompletionSource = (context: CompletionContext) => {
		const line = context.state.doc.lineAt(context.pos);
		const textBefore = line.text.slice(0, context.pos - line.from);

		const importMatch = textBefore.match(importRegex);
		if (importMatch) {
			return {
				from: context.pos - (importMatch[1]?.length || 0),
				options: [{ label: 'controlhub', type: 'module', detail: 'ControlHub automation module' }],
				validFor: /^\w*$/
			};
		}

		const fromMatch = textBefore.match(fromRegex);
		if (fromMatch && !textBefore.includes('import')) {
			return {
				from: context.pos - (fromMatch[1]?.length || 0),
				options: [{ label: 'controlhub', type: 'module', detail: 'ControlHub automation module' }],
				validFor: /^\w*$/
			};
		}

		const fromImportMatch = textBefore.match(fromImportRegex);
		if (fromImportMatch && fromImportMatch[1] === 'controlhub') {
			return {
				from: context.pos - (fromImportMatch[2]?.length || 0),
				options: customCompletions,
				validFor: /^\w*$/
			};
		}

		const word = context.matchBefore(/[\w.]*/);
		if (!word || word.from == word.to) return null;

		const text = word.text;
		const isModuleAccess = text.startsWith('controlhub.');

		if (!isModuleAccess && !context.explicit) return null;

		if (isModuleAccess) {
			return {
				from: word.from + 'controlhub.'.length,
				options: customCompletions,
				validFor: /^\w*$/
			};
		}

		if (!context.explicit && !textBefore.trim().match(/^(import|from)/)) {
			return {
				from: word.from,
				options: [{ label: 'controlhub', type: 'module', detail: 'ControlHub automation module' }],
				validFor: /^controlhub$/
			};
		}

		return null;
	};

	const syntaxLinter = linter((view) => {
		const tree = syntaxTree(view.state);
		const diagnostics: Diagnostic[] = [];
		const doc = view.state.doc.toString();

		tree.iterate({
			enter: (node) => {
				if (node.type.isError) {
					const errorLine = view.state.doc.lineAt(node.from).number;
					const errorText = doc.slice(node.from, node.to);
					diagnostics.push({
						from: node.from,
						to: node.to,
						severity: 'error',
						message: `Syntax error at line ${errorLine}: "${errorText}"`
					});
				}
			}
		});

		hasCriticalErrors.set(diagnostics.length > 0);
		return diagnostics;
	});

	const checkSyntax = (view: EditorView) => {
		const tree = syntaxTree(view.state);
		const errors = [];

		tree.iterate({
			enter: (node) => {
				if (node.type.isError) {
					errors.push({
						from: node.from,
						to: node.to,
						message: 'Syntax error'
					});
				}
			}
		});

		hasCriticalErrors.set(errors.length > 0);
	};

	const updateScript = async () => {
		if ($hasCriticalErrors) {
			throw new Error('Syntax error in script!');
		}

		scriptStore.update((script) => {
			if (script) {
				script.name = $scriptName;
				script.description = $scriptDescription;
				script.executable = $scriptExecutable;
			}

			return script;
		});

		await pb.collection('scripts').update($scriptStore?.id as string, {
			name: $scriptName,
			description: $scriptDescription,
			executable: $scriptExecutable
		});
	};
</script>

<div class="grid grid-cols-1 gap-4">
	{#if editable}
		<Button
			onclick={toastApi.execAsync(
				updateScript,
				`Script ${$scriptStore?.name} updated.`,
				`Failed to update script ${$scriptStore?.name}, check syntax error.`
			)}
			class="w-full"
			variant={$hasCriticalErrors ? 'destructive' : !$hasChanges ? 'outline' : 'default'}
			disabled={$hasCriticalErrors || !$hasChanges}
		>
			Save

			{#if $hasChanges}
				<span class="text-sm font-medium text-amber-500">* Unsaved changes</span>
			{/if}

			{#if $hasCriticalErrors}
				<span class="text-sm font-medium text-red-500">⚠️ Cannot save due to syntax errors</span>
			{/if}
		</Button>
	{/if}

	<div class="grid grid-cols-1 gap-4">
		<Input
			bind:value={$scriptName}
			placeholder="Script name"
			class="w-full"
			disabled={!editable}
		/>
		<Textarea
			bind:value={$scriptDescription}
			placeholder="Script description"
			class="w-full"
			rows={7}
			disabled={!editable}
		/>

	<div class="overflow-hidden rounded-md">
		{#if browser}
			<CodeMirror
				bind:value={$scriptExecutable}
				lang={python()}
				theme={$theme == 'light' ? githubLight : githubDark}
				tabSize={4}
				{editable}
				extensions={[
					syntaxLinter,
					autocompletion({
						override: [controlHubModuleCompletionSource, globalCompletion, localCompletionSource]
					})
				]}
				styles={{
					'&': {
						width: '100%',
						height: '100%',
						'overflow-y': 'auto',
						'overflow-x': 'hidden'
					}
				}}
				on:ready={(e) => {
					view = e.detail;
					checkSyntax(view);
				}}
				on:change={() => {
					if (view) {
						checkSyntax(view);
					}
				}}
			/>
		{/if}
	</div>
</div>
</div>