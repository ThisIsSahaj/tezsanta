<script>
  import { BeaconWallet } from "@taquito/beacon-wallet";
  import { NetworkType } from "@airgap/beacon-types";
  import { TezosToolkit } from "@taquito/taquito";

  const rpcUrl = "https://ghostnet.ecadinfra.com";
  const Tezos = new TezosToolkit(rpcUrl);
  // const contractAddress = "KT1R4i4qEaxF7v3zg1M8nTeyrqk8JFmdGLuu";
  const contractAddress = "KT1W8FrDRM28BGy1VVKXfN9L61jW1dgAjHQi"


  let wallet;
  let address;
  let balance;
  let bankBalance;

  let depositAmount = 1;
  let depositButtonActive = false;
  let depositButtonLabel = "Deposit";

  let withdrawButtonActive = true;
  let withdrawButtonLabel = "Withdraw";

  const connectWallet = async () => {
    const newWallet = new BeaconWallet({
      name: "Tez Santa",
      network: {
        type: NetworkType.GHOSTNET,
      },
    });
    await newWallet.requestPermissions();
    address = await newWallet.getPKH();
    await getWalletBalance(address);
    await getBankBalance(address);
    wallet = newWallet;
    depositButtonActive = true;
  };

  const disconnectWallet = () => {
    wallet.client.clearActiveAccount();
    wallet = undefined;
  };

  const getWalletBalance = async (walletAddress) => {
    const balanceMutez = await Tezos.tz.getBalance(walletAddress);
    balance = balanceMutez.div(1000000).toFormat(2);
  };

  const getBankBalance = async (walletAddress) => {
    const contract = await Tezos.wallet.at(contractAddress);
    const storage = await contract.storage();
    const balanceMutez = await storage.get(walletAddress);
    bankBalance = isNaN(balanceMutez) ? 0 : balanceMutez / 1000000;
  };

  const deposit = async () => {
    depositButtonActive = false;
    depositButtonLabel = "Depositing...";

    Tezos.setWalletProvider(wallet);
    const contract = await Tezos.wallet.at(contractAddress);
    const transactionParams = await contract.methods
      .deposit()
      .toTransferParams({
        amount: depositAmount,
      });
    const estimate = await Tezos.estimate.transfer(transactionParams);

    const operation = await Tezos.wallet
      .transfer({
        ...transactionParams,
        ...estimate,
      })
      .send();

    console.log(`Waiting for ${operation.opHash} to be confirmed...`);

    await operation.confirmation(2);

    console.log(
      `Operation injected: https://ghost.tzstats.com/${operation.opHash}`
    );

    await getWalletBalance(address);
    await getBankBalance(address);
    depositButtonActive = true;
    depositButtonLabel = "Deposit";
  };

  const withdraw = async () => {
    withdrawButtonActive = false;
    withdrawButtonLabel = "Withdrawing...";

    Tezos.setWalletProvider(wallet);
    const contract = await Tezos.wallet.at(contractAddress);

    const transactionParams = await contract.methods
      .withdraw()
      .toTransferParams();
    const estimate = await Tezos.estimate.transfer(transactionParams);

    const operation = await Tezos.wallet
      .transfer({
        ...transactionParams,
        ...estimate,
      })
      .send();

    console.log(`Waiting for ${operation.opHash} to be confirmed...`);

    await operation.confirmation(2);

    console.log(
      `Operation injected: https://ghost.tzstats.com/${operation.opHash}`
    );

    await getWalletBalance(address);
    await getBankBalance(address);
    withdrawButtonActive = true;
    withdrawButtonLabel = "Withdraw";
  };
</script>

<main>
  
  <h1>Tez Santa dApp</h1>
  
  <div class="card">
    {#if wallet}
      <p>Your wallet address is {address}.</p>
      <p>You currently have {balance} tez in your wallet.</p>
      <p>Your wallet's balance in the bank is {bankBalance}.</p>
      <p>
        Deposit tez:
        <input type="number" bind:value={depositAmount} min="1" max="100" />
        <input type="range" bind:value={depositAmount} min="1" max="100" />
        <button on:click={deposit} disabled={!depositButtonActive}>
          {depositButtonLabel}
        </button>
      </p>
      <p>
        Withdraw tez:
        <button on:click={withdraw} disabled={!withdrawButtonActive}>
          {withdrawButtonLabel}
        </button>
      </p>
      <p>
        <button on:click={disconnectWallet}> Disconnect wallet </button>
      </p>
    {:else}
      <button on:click={connectWallet}> Connect wallet </button>
    {/if}
  </div>
</main>

<style>
</style>
