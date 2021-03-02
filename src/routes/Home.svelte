<script>
	import RouteLayout from "../components/RouteLayout.svelte";
	import SecondaryButton from "./../components/SecondaryButton.svelte";
	import * as polyfill from "credential-handler-polyfill";
	import { v4 as uuid } from "uuid";
	import loadDIDKit from "../DIDKit.js";

	$: error = null;
	$: statusMessage = "";
	let checkVerifyPresentation = true;
	let checkVerifyCredential = true;
	let verification;
	let verifiableCredentials = [];
	let vcUrls = [];

	function toArray(value) {
		return !value ? [] : Array.isArray(value) ? value : [value];
	}

	function createJsonBlobUrl(value) {
		const blob = new Blob([JSON.stringify(value, null, 2)]);
		return URL.createObjectURL(blob, { type: "application/json" });
	}

	function appendVerification(verif) {
		if (!verification) {
			verification = verif;
		} else {
			verification.errors.push(verif.errors);
			verification.warnings.push(verif.warnings);
			verification.checks.push(verif.checks);
		}
	}

	async function requestCredentials1() {
		error = null;
		statusMessage = "Loading...";
		verification = null;
		await polyfill.loadOnce();
		const DIDKit = await loadDIDKit();
		statusMessage = "Requesting credentials...";
		const challenge = uuid();
		const credentialQuery = {
			web: {
				VerifiablePresentation: {
					challenge,
					query: {
						type: "QueryByExample",
						credentialQuery: [
							{
								reason: "Degen Verifier",
							}
						]
					}
				}
			}
		};
		const webCredential = await navigator.credentials.get(credentialQuery);
		if (webCredential.type !== "web") {
			console.log(webCredential);
			throw new Error("Unexpected credential type: " + webCredential.type);
		}

		if (webCredential.dataType !== "VerifiablePresentation") {
			console.log(webCredential);
			throw new Error("Unexpected credential dataType: " + webCredential.dataType);
		}

		const vp = webCredential.data;
		if (!vp) {
			console.log(webCredential);
			throw new Error("Missing credential data");
		}

		if (checkVerifyPresentation) {
			const verifyOptions = {
				proofPurpose: "authentication",
				challenge
			};
			verification = JSON.parse(await DIDKit.verifyPresentation(
				JSON.stringify(vp), JSON.stringify(verifyOptions)));
			console.log(verification);
		}

		if (checkVerifyCredential) {
			// Separately verify the credentials
			// https://github.com/spruceid/didkit/issues/78
			const vcs = toArray(vp.verifiableCredential);
			if (!vcs.length) {
				console.log(vp);
				throw new Error("Missing credentials in presentation");
			}
			for (let vc of vcs) {
				if (typeof vc === "string") {
					throw new Error("Unsupported JWT VC");
				}
				const verifyOptions = {
					proofPurpose: "assertionMethod"
				};
				const vcVerification = JSON.parse(await DIDKit.verifyCredential(
					JSON.stringify(vc), JSON.stringify(verifyOptions)));
				console.log(vcVerification);
				appendVerification(vcVerification);
			}
			verifiableCredentials = vcs;
			vcUrls = vcs.map(createJsonBlobUrl);
		}

		if (!verification) {
			// nothing verified
			return;
		}

		// TODO: check expirationDate

		if (verification.errors.length) {
			statusMessage = "Verification failed.";
		} else if (verification.warnings.length) {
			statusMessage = "Verified, but there are some warnings.";
		} else {
			statusMessage = "Verified!";
		}
	}

	async function requestCredentials() {
		try {
			await requestCredentials1();
		} catch(e) {
			statusMessage = "";
			error = e;
			console.error(e);
		}
	}
</script>

<RouteLayout>
	<!--
	<img src="/header_effect.svg" alt="header background effect" class="h-52" />
	-->
	<div class="flex flex-col justify-start flex-grow mx-4">
		<h1
			class="md:mt-8 md:mb-12 mb-4 font-semibold text-6xl text-white text-center"
		>
			Degen Verifier
		</h1>

		<ul>
			<li><label for="verify-presentation">
				<input name="verify-presentation" type="checkbox" bind:checked={checkVerifyPresentation}> Verify presentation
			</label></li>
			<li><label for="verify-credential">
				<input name="verify-credential" type="checkbox" bind:checked={checkVerifyCredential}> Verify credential
			</label></li>
		</ul>

		<SecondaryButton
			onClick={requestCredentials}
			label="Request Credentials"
		/>

		{#if error}
			<div class="error">
				<p><strong>{error.name}</strong>: {error.message}</p>
				<pre style="white-space: pre-wrap">{error.stack}</pre>
			</div>
		{/if}
		{#if statusMessage}
			<div>
				<p>{statusMessage}</p>
			</div>
		{/if}

		{#if verification}
			{#if verification.errors}
				<h4>Errors</h4>
				<ul>
				{#if verification.errors.length === 0}
					<li>· None</li>
				{/if}
				{#each verification.errors as error}
					<li class="error">✗ <code>{error}</code></li>
				{/each}
				</ul>
			{/if}

			{#if verification.warnings}
				<h4>Warnings</h4>
				<ul>
				{#if verification.warnings.length === 0}
					<li>· None</li>
				{/if}
				{#each verification.warnings as warnings}
					<li class="warning">! <code>{warning}</code></li>
				{/each}
				</ul>
			{/if}

			{#if verification.checks}
				<h4>Checks</h4>
				<ul>
				{#if verification.checks.length === 0}
					<li>· None</li>
				{/if}
				{#each verification.checks as check}
					<li class="check">✓ <code>{check}</code></li>
				{/each}
				</ul>
			{/if}
		{/if}

		{#if verifiableCredentials.length}
			<h3>Credentials</h3>
			{#each verifiableCredentials as vc, i}
				<strong>Credential #{i+1}</strong>
				<ul>
				{#if vc.id}
					<li>· <code>id</code>: <code>{vc.id}</code></li>
				{/if}
				{#if vc.type}
					<li>· Type: <code>{JSON.stringify(vc.type)}</code></li>
				{/if}
				{#if vc.issuer}
					<li>· Issuer: <code>{vc.issuer}</code></li>
				{/if}
				{#if vc.issuanceDate}
					<li>· Issuance date: <code>{vc.issuanceDate}</code></li>
				{/if}
				{#if vc.expirationDate}
					<li>· Expiration date: <code>{vc.expirationDate}</code></li>
				{/if}
				{#if vc.credentialSubject.id}
					<li>· Subject: <code>{vc.credentialSubject.id}</code></li>
				{/if}
				<div><a href={vcUrls[i]}>Download Credential</a></div>
				</ul>
			{/each}
		{/if}

	</div>
</RouteLayout>

<style>
.error {
	color: red;
}
.warning {
	color: yellow;
}
.check {
	color: silver;
}
p, h3, h4 {
	margin: 0.5em 0;
}
h3 {
	font-weight: bold;
}
h3, h4 {
	text-decoration: underline;
}
a:link {
	text-decoration: underline;
	color: #ff9;
}
</style>
