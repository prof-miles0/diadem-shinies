<script lang="ts">
	import Button from "@/components/ui/input/Button.svelte";
	import { fly } from "svelte/transition";
	import { CirclePlus, User, Users } from "@lucide/svelte";
	import {
		filtersetPageNew,
		filtersetPageSelect,
		getFiltersetPageTransition
	} from "@/lib/features/filters/filtersetPages.svelte.js";
	import { setCurrentSelectedFilterset } from "@/lib/features/filters/filtersetPageData.svelte.js";
	import { filterTitle } from "@/lib/features/filters/filtersetUtils.svelte";
	import FiltersetIcon from "@/lib/features/filters/FiltersetIcon.svelte";
	import Separator from "@/components/ui/Separator.svelte";
	import { getPremadeFiltersets } from "@/lib/features/filters/premadeFiltersets";
	import type { FilterCategory } from "@/lib/features/filters/filters";
	import type { AnyFilterset, FiltersetPokemon } from "@/lib/features/filters/filtersets";
	import { pokemonFiltersetRequiredFeature } from "@/lib/features/filters/filterUtilsPokemon";
	import { hasFeatureAnywhere } from "@/lib/services/user/checkPerm";
	import { getUserDetails } from "@/lib/services/user/userDetails.svelte";
	import { Features } from "@/lib/utils/features";
	import { getId } from "@/lib/utils/uuid";
	import { m } from "@/lib/paraglide/messages";
	import { isSupportedFeature } from "@/lib/services/supportedFeatures";
	import { getCompleteTrainers, refreshLinkedTrainers } from "@/lib/services/linkedTrainers.svelte";
	import { onMount } from "svelte";

	let {
		majorCategory,
		subCategory = undefined
	}: {
		majorCategory: FilterCategory;
		subCategory?: FilterCategory;
	} = $props();

	function isPremadePermitted(filterset: AnyFilterset): boolean {
		if (majorCategory !== "pokemon" && subCategory !== "pokemon") return true;

		const pokemon = filterset as FiltersetPokemon;
		if (
			(pokemon.personalShiny || pokemon.weight || pokemon.showcase) &&
			!(
				isSupportedFeature("personalShiny") &&
				hasFeatureAnywhere(getUserDetails().permissions, Features.POKEMON_PERSONAL_SHINY)
			)
		)
			return false;

		const required = pokemonFiltersetRequiredFeature(pokemon);
		if (required === Features.POKEMON) return true;
		return hasFeatureAnywhere(getUserDetails().permissions, required);
	}

	// premade personal/weight filtersets ask which linked trainer they're for before opening
	let trainerPickFor: FiltersetPokemon | undefined = $state();
	let trainers = $derived(getCompleteTrainers());

	/** trainerId: the chosen trainer, null = all linked trainers, undefined = nothing to choose */
	function selectPremade(filterset: AnyFilterset, trainerId?: string | null) {
		// copy, so editing the new filterset never changes the premade itself
		const data = $state.snapshot(filterset) as FiltersetPokemon;
		data.id = getId();
		if (trainerId !== undefined) {
			// personal shiny already ties the weight to the same trainer
			if (data.personalShiny) data.personalShinyTrainers = trainerId ? [trainerId] : [];
			else if (trainerId) data.weightTrainer = trainerId;
		}
		trainerPickFor = undefined;
		setCurrentSelectedFilterset(majorCategory, subCategory, data, false);
		filtersetPageSelect();
	}

	onMount(() => {
		if (isSupportedFeature("personalShiny")) refreshLinkedTrainers().then();
	});

	let allPremades = $derived(
		getPremadeFiltersets(majorCategory) ??
			(subCategory ? getPremadeFiltersets(subCategory) : []) ??
			[]
	);
	let premades = $derived(allPremades.filter(isPremadePermitted));
</script>

<div in:fly={getFiltersetPageTransition().in} out:fly={getFiltersetPageTransition().out}>
	<Button variant="secondary" size="lg" class="w-full" onclick={filtersetPageNew}>
		<CirclePlus size="18" />
		<span>
			{m.create_new()}
		</span>
	</Button>

	<Separator class="my-3" text={m.or_select_suggested_filter()} />

	<div class="-mx-4 px-4">
		{#if trainerPickFor}
			<div class="flex items-center gap-2 mb-2">
				<FiltersetIcon filterset={trainerPickFor} size={5} />
				<span class="font-semibold">{filterTitle(trainerPickFor)}</span>
			</div>
			<p class="text-sm text-muted-foreground mb-2">{m.premade_choose_trainer()}</p>
			<div class="flex flex-col gap-1">
				{#each trainers as trainer (trainer.id)}
					<Button
						class="w-full flex gap-2 items-center justify-start rounded-md py-2 h-12 m-0! pl-4 pr-2"
						size=""
						variant="outline"
						onclick={() => selectPremade(trainerPickFor!, trainer.id)}
					>
						<User size="18" />
						<span>{trainer.trainerName ?? m.linked_trainers_unnamed()}</span>
					</Button>
				{/each}
				<Button
					class="w-full flex gap-2 items-center justify-start rounded-md py-2 h-12 m-0! pl-4 pr-2"
					size=""
					variant="outline"
					onclick={() => selectPremade(trainerPickFor!, null)}
				>
					<Users size="18" />
					<span>{m.personal_shiny_all_trainers()}</span>
				</Button>
				<Button variant="ghost" class="mt-1" onclick={() => (trainerPickFor = undefined)}>
					{m.cancel()}
				</Button>
			</div>
		{:else}
			<div class="flex flex-col gap-1">
				{#each premades as filterset (filterset.id)}
					<Button
						class="w-full flex gap-2 items-center justify-start rounded-md py-2 h-12 m-0! pl-4 pr-2"
						size=""
						variant="outline"
						onclick={() => {
							const pokemon = filterset as FiltersetPokemon;
							const needsTrainer = pokemon.personalShiny
								? !pokemon.personalShinyTrainers
								: pokemon.weight && !pokemon.weightTrainer;
							if (!needsTrainer) selectPremade(filterset);
							else if (trainers.length > 1) trainerPickFor = pokemon;
							else selectPremade(filterset, trainers[0]?.id);
						}}
					>
						<FiltersetIcon {filterset} size={5} />
						<span>{filterTitle(filterset)}</span>
					</Button>
				{/each}
			</div>
		{/if}
	</div>
</div>
