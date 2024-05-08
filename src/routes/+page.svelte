<script>
import ConnectButton from './ConnectButton.svelte';
import { onMount } from 'svelte';

import {
	Contract,
	Networks,
	SorobanRpc,
	Transaction,
	TransactionBuilder,
	BASE_FEE,
	nativeToScVal,
	xdr,
} from "@stellar/stellar-sdk";
import pkg from '@stellar/freighter-api';
const {  signTransaction } = pkg;

import { contractAddress } from '$lib/contractAddress.js';

const server = new SorobanRpc.Server("https://soroban-testnet.stellar.org");
const contract = new Contract(contractAddress);

let sourceAccount;
let pk;

let t = 0
let count = GetData()

async function GetData() {
	try {
		let r = await server.getContractData(
				contractAddress,
				xdr.ScVal.scvLedgerKeyContractInstance()
			);
		t = r.val.contractData()
			.val()
			.instance()
			.storage()[0]
			.val()._value;
		return t;
	} catch {
		return 0;
	}
}

async function increment() {
	// Crear transaccion
	const transaction = new TransactionBuilder(sourceAccount, {
		fee: BASE_FEE,
		networkPassphrase: Networks.TESTNET,
	})
		.addOperation(contract.call('increment'))
		.setTimeout(30)
		.build();

	// Perparar transaccion
	let preparedTransaction = await server.prepareTransaction(transaction);
	// Firmar transaccion
	let signedTx = await signTransaction(preparedTransaction.toXDR(), { networkPassphrase: Networks.TESTNET})

	// Publicar transaccion firmada
	const txEnvelope = xdr.TransactionEnvelope.fromXDR(signedTx, 'base64');
	const tx = new Transaction(txEnvelope, Networks.TESTNET);
	const sendResponse = await server.sendTransaction(tx);
	if (sendResponse.errorResult) {
		throw new Error('Unable to submit tx');
	}

	// Esperara el resultado
	let txResponse = await server.getTransaction(sendResponse.hash);
	while (txResponse.status === SorobanRpc.Api.GetTransactionStatus.NOT_FOUND) {
		txResponse = await server.getTransaction(sendResponse.hash);
		await new Promise((resolve) => setTimeout(resolve, 1000));
	}
	t = txResponse.returnValue._value;
	return t
}

let sum = 1

async function incrementN() {
	const transaction = new TransactionBuilder(sourceAccount, {
		fee: BASE_FEE,
		networkPassphrase: Networks.TESTNET,
	})
		.addOperation(contract.call('increment_n', nativeToScVal(sum, { type: 'u32' })))
		.setTimeout(30)
		.build();

	let preparedTransaction = await server.prepareTransaction(transaction);
	let signedTx = await signTransaction(preparedTransaction.toXDR(), { networkPassphrase: Networks.TESTNET})
	const txEnvelope = xdr.TransactionEnvelope.fromXDR(signedTx, 'base64');
    const tx = new Transaction(txEnvelope, Networks.TESTNET);


	const sendResponse = await server.sendTransaction(tx);
	if (sendResponse.errorResult) {
		throw new Error('Unable to submit tx');
	}

	let txResponse = await server.getTransaction(sendResponse.hash);
	while (txResponse.status === SorobanRpc.Api.GetTransactionStatus.NOT_FOUND) {
		txResponse = await server.getTransaction(sendResponse.hash);
		await new Promise((resolve) => setTimeout(resolve, 1000));
	}
	t = txResponse.returnValue._value;
	return t
}

</script>

<div class="container">
<h1>Workshop Soroban</h1>

<ConnectButton server={server} bind:pk={pk} bind:sourceAccount={sourceAccount}/>

{#if sourceAccount }

{#await count}
	<p class="counter">{t}</p>
{:then n}
	<p class="counter">{n}</p>
	<button class="button" on:click={() => count = increment()}>Increment</button>
	<div class="increment_n">
		<input class="number-input" type="number" bind:value={sum} min="1" />
		<button class="button" on:click={() => count = incrementN()}>Increment_n</button>
	</div>
{/await}

{/if}

</div>

<style>

.container {
	width: 100vw;
	height: 100vh;
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
}

.counter {
	width: 10rem;
	display: flex;
	align-items: center;
	justify-content: center;
	height: 10rem;
	background: white;
	border-radius: 50%;
	border: purple 3px solid;
	color: black;
	font-size: 2rem;
	font-family: monospace;
}

.increment_n {
	color: inherit;
	background: #222;
	margin-top: 1rem;
	padding: 0.5rem 2rem;
	border-radius: 0.3rem;
	display: flex;
	gap: 1rem;
}

.number-input {
	width: 4rem;
	color: inherit;
	background: #424242;
	border: 0;
	font-size: 2rem;
	border-radius: 0.5rem;
	text-align: center;
}

.button {
	font-size: 2rem;
	color: inherit;
	background: #424242;
	padding: 0.5rem;
	border-radius: 0.5rem;
	border: 0;
	border-bottom: 2px grey solid;
	border-right: 2px grey solid;
}

</style>
