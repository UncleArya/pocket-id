<script lang="ts">
	import CheckboxWithLabel from '$lib/components/checkbox-with-label.svelte';
	import FileInput from '$lib/components/file-input.svelte';
	import FormInput from '$lib/components/form-input.svelte';
	import { Button } from '$lib/components/ui/button';
	import Label from '$lib/components/ui/label/label.svelte';
	import type {
		OidcClient,
		OidcClientCreate,
		OidcClientCreateWithLogo
	} from '$lib/types/oidc.type';
	import { createForm } from '$lib/utils/form-util';
	import { set, z } from 'zod';
	import OidcCallbackUrlInput from './oidc-callback-url-input.svelte';

	let {
		callback,
		existingClient
	}: {
		existingClient?: OidcClient;
		callback: (user: OidcClientCreateWithLogo) => Promise<boolean>;
	} = $props();

	let isLoading = $state(false);
	let logo = $state<File | null | undefined>();
	let logoDataURL: string | null = $state(
		existingClient?.hasLogo ? `/api/oidc/clients/${existingClient!.id}/logo` : null
	);

	const client: OidcClientCreate = {
		name: existingClient?.name || '',
		callbackURLs: existingClient?.callbackURLs || [''],
		isPublic: existingClient?.isPublic || false,
		pkceEnabled: existingClient?.isPublic == true || existingClient?.pkceEnabled || false
	};

	const formSchema = z.object({
		name: z.string().min(2).max(50),
		callbackURLs: z.array(z.string()).nonempty(),
		isPublic: z.boolean(),
		pkceEnabled: z.boolean()
	});

	type FormSchema = typeof formSchema;
	const { inputs, ...form } = createForm<FormSchema>(formSchema, client);

	async function onSubmit() {
		const data = form.validate();
		if (!data) return;
		isLoading = true;
		const success = await callback({
			...data,
			logo
		});
		// Reset form if client was successfully created
		if (success && !existingClient) form.reset();
		isLoading = false;
	}

	function onLogoChange(e: Event) {
		const file = (e.target as HTMLInputElement).files?.[0] || null;
		if (file) {
			logo = file;
			const reader = new FileReader();
			reader.onload = (event) => {
				logoDataURL = event.target?.result as string;
			};
			reader.readAsDataURL(file);
		}
	}

	function resetLogo() {
		logo = null;
		logoDataURL = null;
	}
</script>

<form onsubmit={onSubmit}>
	<div class="grid grid-cols-2 gap-3 sm:flex-row">
		<FormInput label="Name" class="w-full" bind:input={$inputs.name} />
		<OidcCallbackUrlInput
			class="w-full"
			bind:callbackURLs={$inputs.callbackURLs.value}
			bind:error={$inputs.callbackURLs.error}
		/>
		<CheckboxWithLabel
			id="public-client"
			label="Public Client"
			description="Public clients do not have a client secret and use PKCE instead. Enable this if your client is a SPA or mobile app."
			onCheckedChange={(v) => {
				if (v == true) form.setValue('pkceEnabled', true);
			}}
			bind:checked={$inputs.isPublic.value}
		/>
		<CheckboxWithLabel
			id="pkce"
			label="PKCE"
			description="Public Key Code Exchange is a security feature to prevent CSRF and authorization code interception attacks."
			disabled={$inputs.isPublic.value}
			bind:checked={$inputs.pkceEnabled.value}
		/>
	</div>
	<div class="mt-8">
		<Label for="logo">Logo</Label>
		<div class="mt-2 flex items-end gap-3">
			{#if logoDataURL}
				<div class="h-32 w-32 rounded-2xl bg-muted p-3">
					<img
						class="m-auto max-h-full max-w-full object-contain"
						src={logoDataURL}
						alt={`${$inputs.name.value} logo`}
					/>
				</div>
			{/if}
			<div class="flex flex-col gap-2">
				<FileInput
					id="logo"
					variant="secondary"
					accept="image/png, image/jpeg, image/svg+xml"
					onchange={onLogoChange}
				>
					<Button variant="secondary">
						{logoDataURL ? 'Change Logo' : 'Upload Logo'}
					</Button>
				</FileInput>
				{#if logoDataURL}
					<Button variant="outline" on:click={resetLogo}>Remove Logo</Button>
				{/if}
			</div>
		</div>
	</div>
	<div class="w-full"></div>
	<div class="mt-5 flex justify-end">
		<Button {isLoading} type="submit">Save</Button>
	</div>
</form>
