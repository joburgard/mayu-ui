<!--
SPDX-FileCopyrightText: 2023 Jo Burgard <mail@joburgard.com>
SPDX-License-Identifier: Unlicense
-->

<script lang="ts" generics="T">
	import uFuzzy from '@leeoniya/ufuzzy';
	import { createCombobox, melt, type ComboboxOptionProps } from '@melt-ui/svelte';
	import { fly } from 'svelte/transition';
	import {
		inputPlaceholderVariants,
		inputVariants,
		type InputEvents,
		type InputProps,
	} from '../input';

	type $$Props = Omit<InputProps, 'value'> & {
		data: T[];
		value: T | undefined;
		createHaystack?: (item: T) => string;
		toOption?: (item: T) => ComboboxOptionProps<T>;
	};
	type $$Events = InputEvents;

	export let data: $$Props['data'];
	let className: $$Props['class'] = undefined;
	export { className as class };
	export let value: $$Props['value'] = undefined;
	export let placeholder: $$Props['placeholder'] = undefined;
	export let size: $$Props['size'] = undefined;
	export let createHaystack: $$Props['createHaystack'] = undefined;

	let haystack: T[] | string[] = data;

	if (createHaystack) {
		haystack = data.map(createHaystack);
	}

	export let toOption: Required<$$Props>['toOption'] = (item: T): ComboboxOptionProps<T> => {
		if (!(item && typeof item === 'object' && Object.keys(item).length)) {
			throw new Error(
				'If data is not of type {label;value;disabled?} then you have to provide your own toOption function.',
			);
		}
		return {
			// @ts-ignore
			label: item?.label,
			// @ts-ignore
			value: item?.value,
			// @ts-ignore
			disabled: item?.disabled,
		};
	};

	const {
		elements: { menu, input, option, label },
		states: { open, inputValue, touchedInput, selected },
		helpers: { isSelected },
	} = createCombobox<T>({
		forceVisible: true,
		positioning: {
			placement: 'bottom-start',
			sameWidth: false,
		},
	});

	selected.subscribe((newValue) => (value = newValue?.value));

	const fuzzySearch = new uFuzzy({ intraMode: 1 });
	const showAllResult = data.map((_, index) => index);

	function clearValueIfEmpty(event: InputEvents['input']) {
		if (event.currentTarget.value === '') {
			value = undefined;
			$selected = undefined;
		}
	}

	function clearValueAndInput() {
		value = undefined;
		$inputValue = '';
		$selected = undefined;
	}

	$: if (!$open) {
		$inputValue = $selected?.label ?? '';
	}

	$: filteredOptions =
		$touchedInput && $inputValue !== ''
			? fuzzySearch.filter(haystack as string[], $inputValue)
			: showAllResult;
</script>

<div class="isolate">
	<label class="relative block" use:melt={$label}>
		<input
			use:melt={$input}
			class={inputVariants({
				size,
				class: ['placeholder-transparent w-full pr-13', className],
			})}
			on:blur
			on:change
			on:click
			on:focus
			on:keydown
			on:keypress
			on:keyup
			on:mouseover
			on:mouseenter
			on:mouseleave
			on:paste
			on:input={clearValueIfEmpty}
			on:input
			{placeholder}
			{...$$restProps}
		/>
		{#if placeholder}<span class={inputPlaceholderVariants({ size })}>{placeholder}</span>{/if}
		{#if $inputValue !== ''}
			<button
				type="button"
				class="p-1 absolute right-6 top-1/2 z-10 -translate-y-1/2 hover:text-$color-primary"
				on:click={clearValueAndInput}><div class="i-lucide-x-circle" /></button
			>
		{/if}
		<div class="i-lucide-chevrons-up-down absolute right-2 top-1/2 z-10 -translate-y-1/2" />
	</label>
	{#if $open}
		<ul
			class="p-2 border min-h-0 max-h-[500px] rounded-[--roundedness-base] shadow-lg bg-white overflow-y-auto"
			use:melt={$menu}
			transition:fly={{ duration: 150, y: -5 }}
		>
			<!-- svelte-ignore a11y-no-noninteractive-tabindex -->
			<div class="flex min-h-0 flex-col gap-0 overflow-y-scroll" tabindex="0">
				{#each filteredOptions || [] as resultIndex, index (index)}
					{@const optionData = toOption(data[resultIndex])}
					<li
						class="px-3 py-1.5 scroll-my-2 cursor-pointer rounded-[--roundedness-sm] hover:(bg-gray-100) data-[highlighted]:bg-gray-200 select-none"
						use:melt={$option(optionData)}
					>
						<div class="break-words">{optionData.label}</div>
					</li>
				{:else}
					<li class="px-3 py-1.5 rounded-[--roundedness-sm] select-none">
						No matching entry found.
					</li>
				{/each}
			</div>
		</ul>
	{/if}
</div>
