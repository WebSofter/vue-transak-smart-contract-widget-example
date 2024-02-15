<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
  </div>
</template>

<script>
import { watchEffect, ref, defineComponent } from "vue"; 
import { TransakConfig, Transak } from "@transak/transak-sdk";
import Pusher from "pusher-js";
import { parseUnits } from "@ethersproject/units";
import { Interface, Logger } from "ethers/lib/utils";

// Public channel for all transak order events
let pusher = new Pusher("1d9ffac87de599c61283", { cluster: "ap2" });

/**
 * Smart contract address of the partner
 * Here we are using aave smart contract on polygon
 * https://mumbai.polygonscan.com/address/0xcC6114B983E4Ed2737E9BD3961c9924e6216c704
 * We will use this to deposit funds to protocol
 * ---------------------------------------------
 * Адрес смарт-контракта партнера
 * Здесь мы используем смарт-контракт aave для полигона.
 * https://mumbai.polygonscan.com/address/0xcC6114B983E4Ed2737E9BD3961c9924e6216c704
 * Мы будем использовать это для внесения средств в протокол.
 */
const SMART_CONTRACT_ADDRESS = "0xcC6114B983E4Ed2737E9BD3961c9924e6216c704";

/**
 * The address of the token which smart contract is going to receive
 * Here we are using WBTC on polygon
 * Transak will send this token address amount to the smart contract
 * ---------------------------------------------
 * Адрес токена, который получит смарт-контракт.
 * Здесь мы используем WBTC на полигоне.
 * Transak отправит сумму этого адреса токена в смарт-контракт.
 */
const SOURCE_TOKEN = "0x2Fa2e7a6dEB7bb51B625336DBe1dA23511914a8A";

/**
 * The address of the user
 * We will use this address to deposit funds to protocol
 * CallData needs to be encoded using the ABI for this wallet address
 * ---------------------------------------------
 * Адрес пользователя
 * Мы будем использовать этот адрес для внесения средств в протокол.
 * CallData необходимо закодировать с использованием ABI для этого адреса кошелька.
 */
const USER_WALLET_ADDRESS = "0xb21CB810816539d8371ef56A2E10731ADAed59DB";
/**
 * Supply function calldata
 * We will use this calldata to deposit funds to protocol
 * Data needs to be encoded using the ABI of the smart contract
 * ---------------------------------------------
 * Поставка данных вызова функции
 * Мы будем использовать эти данные вызова для внесения средств в протокол.
 * Данные необходимо закодировать с использованием ABI смарт-контракта.
 */
const getSupplyCalldata = (contributor, amount) => {
  if (!amount) return;

  /**
   * ABI for Supply function on aave SmartContract
   * You can find ABI of aave contract here
   * https://mumbai.polygonscan.com/address/0xbadd48c3eb42a10db791d7b02e3c07fbf95b3155#code
   * ---------------------------------------------
   * ABI для функции снабжения в aave SmartContract.
   * Вы можете найти ABI контракта aave здесь.
   * https://mumbai.polygonscan.com/address/0xbadd48c3eb42a10db791d7b02e3c07fbf95b3155#code
   */
  let ABI = [
    "function supply(address asset,uint256 amount,address onBehalfOf,uint16 referralCode)",
  ];
  let parsedAmmount = parseUnits(amount.toString(), 8);
  // Constructing the CallData / Создание данных вызова
  return new Interface(ABI).encodeFunctionData("supply", [
    SOURCE_TOKEN, // contract address for WBTC on mumbai testnet / Контрактный адрес WBTC в тестовой сети Мумбаи
    parsedAmmount, // amount user wants to buy / сумма, которую пользователь хочет купить
    contributor, // user who is depositing the funds / пользователь, который вносит средства
    0, // referal code. Will be 0 in case of aave / Промо-код. Будет 0 в случае aave
  ]);
};

export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
  setup() {
    watchEffect(() => {
      const depositAmount = 0.0008; // amount user want to deposit to protocol / сумма, которую пользователь хочет внести в протокол

      const calldata = getSupplyCalldata(
        USER_WALLET_ADDRESS, // user wallet address / адрес кошелька пользователя
        depositAmount, // amount user want to deposit. Transak will convert it to the fiat equivalent and display it to the user
        // сумма, которую пользователь хочет внести. Transak преобразует его в фиатный эквивалент и отобразит пользователю.
      );

      if (!calldata) return;

      /**
       * You can find list of all possible Transak config options here
       * https://docs.transak.com/docs/transak-one-query-parameters
       * ------------------------------------
       * Список всех возможных вариантов конфигурации Transak можно найти здесь.
       * https://docs.transak.com/docs/transak-one-query-parameters
       */
      const settings = {
        apiKey: "3758451a-36b9-4f92-a8a5-192bdf9d76de",
        environment: Transak.ENVIRONMENTS.STAGING,
        themeColor: "000000",
        defaultPaymentMethod: "credit_debit_card",
        /**
         * Wallet address of the user
         * The blockchain address of the user's wallet that the receipt token will be sent to.
         * ----------------------------------------------------
         * Адрес кошелька пользователя
         * Адрес блокчейна кошелька пользователя, на который будет отправлен токен квитанции.
         */
        walletAddress: USER_WALLET_ADDRESS,
        exchangeScreenTitle: "Deposit Funds",
        disableWalletAddressForm: true,
        smartContractAddress: SMART_CONTRACT_ADDRESS,
        estimatedGasLimit: 70_000,
        calldata,
        /**
         * Details of the token smart contract that is going to be used
         * in the transaction we are supplying WBTC to Aave protocol
         * So we are sending WBTC as sourceTokenData along with the amount
         * we want to deposit on users behalf
         * -----------------------------------------------------
         * Подробная информация о смарт-контракте токена, который будет использоваться.
         * в транзакции мы передаем WBTC по протоколу Aave
         * Итак, мы отправляем WBTC как sourceTokenData вместе с суммой.
         * мы хотим внести депозит от имени пользователей
         */
        sourceTokenData: [
          {
            sourceTokenCode: "WBTC",
            sourceTokenAmount: depositAmount,
          },
        ],
        /**
         * Details of the crypto user is going to receive after they do the transaction.
         * For example if you want to deposit WBTC, you will receive AWBTC in return.
         * So in this case we are sending AWBTC as cryptoCurrencyData
         * Along with name and image of the crypto currency.
         * This will be shown to the user.
         * ---------------------------------------------------------
         * Подробная информация о криптопользователе будет получена после совершения транзакции.
         * Например, если вы хотите внести WBTC, взамен вы получите AWBTC.
         * В данном случае мы отправляем AWBTC как cryptoCurrencyData.
         * Вместе с названием и изображением криптовалюты.
         * Это будет показано пользователю.
         */
        cryptoCurrencyData: [
          {
            cryptoCurrencyCode: "AWBTC",
            cryptoCurrencyName: "Aave Polygon WBTC",
            cryptoCurrencyImageURL:
              "https://assets.coingecko.com/coins/images/11734/standard/aWBTC.png",
          },
        ],
        network: "polygon",
        isTransakOne: true,
      };

      const transak = new Transak(settings);

      transak.init();
      const subscribeToWebsockets = (orderId) => {
        let channel = pusher.subscribe(orderId);

        // receive updates of all the events / получать обновления обо всех событиях
        pusher.bind_global((eventId, orderData) => {
          console.log(`websocket Event: ${eventId} with order data:`, orderData);
        });

        // receive updates of a specific event / получать обновления о конкретном событии
        channel.bind("ORDER_COMPLETED", (orderData) => {
          console.log("ORDER COMPLETED websocket event", orderData);
        });

        channel.bind("ORDER_FAILED", async (orderData) => {
          console.log("ORDER FAILED websocket event", orderData);
        });
      };

      Transak.on(Transak.EVENTS.TRANSAK_ORDER_CREATED, (orderData) => {
        console.log("callback transak order created", orderData);
        const eventData = orderData;

        const orderId = eventData.status?.id;

        if (!orderId) {
          return;
        }

        subscribeToWebsockets(orderId);
      });
    }, []);

  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  display: inline-block;
  margin: 0 10px;
}

a {
  color: #42b983;
}</style>
