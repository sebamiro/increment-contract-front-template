<script>
  import pg from '@stellar/freighter-api';
  const { isAllowed, setAllowed, getUserInfo } = pg;
  import { onMount } from 'svelte';

  export let pk;
  export let sourceAccount;
  export let server;

  async function getPk() {
    const { publicKey } = await getUserInfo();
    return publicKey;
  }

  async function setLoggedIn(publicKey) {
    pk = publicKey
  }

  let allowed = false;

onMount(async () => {
  if (await isAllowed()) {
    const publicKey = await getPk();
    if (publicKey) {
		setLoggedIn(publicKey);
		sourceAccount = await server.getAccount(publicKey);
		allowed = true;
	}
  }
})

let disable = false

async function connect() {
	disable = true;
	await setAllowed();
	const publicKey = await getPk();
	await setLoggedIn(publicKey);
	allowed = true;
	sourceAccount = await server.getAccount(publicKey);
}

</script>

<div id="freighter-wrap" class="wrap" aria-live="polite">
{#if allowed }
<p>Connected as {pk}</p>
{:else}
  <div class="ellipsis">
    <button disable={disable} on:click={() => connect()} data-connect aria-controls="freighter-wrap">Connect</button>
  </div>
{/if}
</div>

<style>
  .wrap {
    text-align: center;
  }

  .ellipsis {
    line-height: 2.7rem;
    margin: auto;
    max-width: 12rem;
    overflow: hidden;
    text-overflow: ellipsis;
    text-align: center;
    white-space: nowrap;
  }
</style>
