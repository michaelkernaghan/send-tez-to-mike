<script lang="ts">
  import { onMount, onDestroy } from "svelte";
  import { TezosToolkit } from "@taquito/taquito";
  import { BeaconWallet } from "@taquito/beacon-wallet";
  import { NetworkType } from "@airgap/beacon-sdk";
  import { HubConnectionBuilder, HubConnection } from "@microsoft/signalr";

  let Tezos: TezosToolkit;
  let wallet: BeaconWallet;
  let subscription: HubConnection;
  let blockHead: { protocol: string; level: number; lastUpdate: string };
  let confirmed: { confirmeation: string};

  const rpcUrl = "https://api.tez.ie/rpc/edonet";
  const packages: { name: string; display: string; version: number }[] = [
    { name: "svelte", display: "Svelte", version: 3 },
    { name: "webpack", display: "Webpack", version: 5 },
    { name: "typescript", display: "TypeScript", version: 3 },
    { name: "sass", display: "Sass", version: 5 },
    { name: "taquito", display: "Taquito", version: 8 },
    { name: "beacon", display: "Beacon SDK", version: 2 }
  ];

  const connect = async () => {
    try {
      wallet = new BeaconWallet({
        name: "Mike wants Tezos",
        preferredNetwork: NetworkType.EDONET
      });
      await wallet.requestPermissions({
        network: {
          type: NetworkType.EDONET,
          rpcUrl
        }
      });
      Tezos.setWalletProvider(wallet);
    } catch (err) {
      console.error(err);
    }
  };

  const disconnect = () => {
    wallet.client.destroy();
    wallet = undefined;
  };

  let success = false;
  const transfer = async () => {
    const amount = 1;
    const address = 'tz1cp2TFke4GdtSYVBtuTPfvawnemuaJwSno';
    const op = await Tezos.wallet.transfer({ to: address, amount: amount }).send();
    await op.confirmation();
    success = true;
  }   


  const subscribeToEvents = async () => {
    const connection = new HubConnectionBuilder()
      .withUrl("https://api.tzkt.io/v1/events")
      .build();

    // auto-reconnect
    connection.onclose(subscribeToEvents);

    connection.on("head", msg => {
      blockHead = {
        protocol: msg.data.protocol,
        level: msg.data.level,
        lastUpdate: msg.data.timestamp
      };
    });

    // open connection
    await connection.start();
    // subscribe to head
    await connection.invoke("SubscribeToHead");
    // return connection instance
    return connection;
  };

  onMount(async () => {
    Tezos = new TezosToolkit(rpcUrl);
    const headerInfo = await Tezos.rpc.getBlockHeader();
    blockHead = {
      protocol: headerInfo.protocol,
      level: headerInfo.level,
      lastUpdate: headerInfo.timestamp
    };
    subscription = await subscribeToEvents();
  });

  onDestroy(async () => {
    // closes subscription when component unmounts
    await subscription.stop();
  });
</script>

<style lang="scss">
  $tezos-blue: #f72e2e;

  .container {
    font-size: 20px;
    max-width: 80%;

    .title {
      color: $tezos-blue;
      font-size: 60px;
      margin: 20px;
    }

    .subtitle {
      font-size: 30px;
      color: #333;
      margin: 10px;
    }

    ul {
      width: 30%;
      margin: 0 auto;
      text-align: left;
      list-style: none;

      li {
        display: flex;
        justify-content: left;
        align-items: center;
        padding: 3px;

        img {
          width: 30px;
          height: 30px;
          margin-right: 20px;
        }
      }
    }

    .chain-info {
      font-size: 15px;

      p {
        margin: 5px;
        font-style: italic;
        color: $tezos-blue;
      }
    }

    button {
      appearance: none;
      border: solid 2px $tezos-blue;
      border-radius: 5px;
      background-color: white;
      padding: 20px;
      font-size: 20px;
      color: $tezos-blue;
      transition: 0.3s;
      cursor: pointer;
      outline: none;

      &:hover {
        color: white;
        background-color: $tezos-blue;
      }
    }
  }
</style>

<main>
  <div class="container">
    <div class="title">
      Mike wants Tez for his cat!
    </div>
    <div>
      <img src={'images/buddy.jpg'} alt="Buddy the Cat">
      <br />
    </div>
    <br />
    <div>
      {#if wallet}
        <button on:click={disconnect}>Disconnect your wallet!</button>
      {:else}
        <button on:click={connect}>Connect your wallet now!</button>
      {/if}
    </div>
    <br />
    <div>
      <button on:click={transfer}>Send 1 Tez to Mike for the Cat!</button>
      {#if success}         
      <p>Thank you for the Tez!</p>
      {/if}
    </div>
    <br />
    {#if blockHead}
      <div class="chain-info">
        <p>Protocol: {blockHead.protocol}</p>
        <p>Level: {blockHead.level}</p>
        <p>Block timestamp: {blockHead.lastUpdate}</p>
      </div>
    {/if}
    <br />
  </div>
</main>
