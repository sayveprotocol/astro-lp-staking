# astro-lp-staking

lp staking contract that distributes lp rewards to users based on share of staked pool
Forked from anchor LP staking contract found here
https://docs.anchorprotocol.com/smart-contracts/anchor-token/staking
but uses blockheight while anchor's uses unix time
with initiation message
```
{
  "sayve_token": "terra16t7x97wuckxm5h927jygjfrt3tcwrzh3u2rlqm,
  "staking_token": "terra1dj3u83nfe9zpqd5r46plwmmmhe30kzr3axtx9x",
  "distribution_schedule": [
    [
      start block [block], 
      end block [block], 
      "distribution amount"
    ]
  ]
}
```
The Staking Contract contains the logic for LP Token staking and reward distribution. SAYVE tokens
allocated for as liquidity incentives are distributed pro-rata to stakers of the SAYVE-UST
Astroport pair LP token.
### Contract addresses
```
  const BOMBAY_CONTRACT_ADDRESS = {
  sayveUstPair: 'terra1x23y2hxpxph6wueyqj5m5grlr23z5dt4wvpn0r',
  sayveUstLPToken: 'terra1dj3u83nfe9zpqd5r46plwmmmhe30kzr3axtx9x',
  staking: 'terra12yfehzpc69jwxeleck83r6vl99np34frqsx0jv',
  SAYVE: 'terra16t7x97wuckxm5h927jygjfrt3tcwrzh3u2rlqm',
  };
```
## LP Staking Process
### Swap
1. Calculate price
```
// Calculate belief price using pool balances.
// Fetch the number of each asset in the pool.
const query_msg = {"pool":{}};
const { assets } = await terra.wasm.contractQuery(pair_address, query_msg);
const beliefbelief_price_sayveSell = (assets[1].amount / assets[0].amount).toFixed(18);
console.log(beliefbelief_price_sayveSell);
const AMOUNT_UST=(beliefbelief_price_sayveSell*AMOUNT_SAYVE).toFixed(6)
```

So price to buy Sayve
```
https://bombay-lcd.terra.dev/wasm/contracts/terra1x23y2hxpxph6wueyqj5m5grlr23z5dt4wvpn0r/store?query_msg=%7B%22pool%22:%7B%7D%7D
```
Response
```
{"height":"8097834","result":{"assets":[{"info":{"token":{"contract_addr":"terra16t7x97wuckxm5h927jygjfrt3tcwrzh3u2rlqm"}},"amount":"19777925683"},{"info":{"native_token":{"denom":"uusd"}},"amount":"2056309064"}],"total_share":"20056264"}}
```
belief_price_sayve = sayve_amount/ust_amount=19777925683/2056309064 = 9.61816782762

or you can do with simuliation, here we simulate how much sayve we get with 1 ust
```
{"simulation":{"offer_asset":{"amount":"1000000","info":{"native_token":{"denom":"uusd"}}}}}
```
which returns
```
{
  "return_amount": "9584652",
  "spread_amount": "4675",
  "commission_amount": "28840"
}
```
so we swapped 1000000 and got back 9584652, so belief_price = (return_amount/swap_amount) = 9584652/1000000=9.584652

If we want the actual token prie, we simulate an offer to sell   
Simulate sell of 1 sayve
```
{"simulation":{"offer_asset":{"amount":"1000000","info":{"token":{"contract_addr":"terra16t7x97wuckxm5h927jygjfrt3tcwrzh3u2rlqm"}}}}}
```
which response is
```
{
  "return_amount": "103653",
  "spread_amount": "5",
  "commission_amount": "311"
}
```
so we swapped 1000000 and got back 103653, so belief_price = (return_amount/swap_amount) = 103653/1000000=0.103653

2. Swap 1 UST for Sayve
```
const 1_UST=1
const MILLION=1000000
const buySwap = new MsgExecuteContract(
  wallet.key.accAddress,
  sayveUstPair, 
  {
    swap: {
      max_spread: "0.01",
      offer_asset: {
        info: {
          native_token: {
            denom: "uusd",
          },
        },
        amount: String(1_UST*MILLION), 
      },
      belief_price: belief_price_sayve,
    },
  },
  new Coins({ uusd: String(1_UST*MILLION) }),
);

const tx_buy = await wallet.createAndSignTx({ msgs: [buySwap] });
const result_buy = await terra.tx.broadcast(tx_buy);
```
### Provide Liquidity
3. Provide Liquidity
Increase Allowence on the token
```
const AMOUNT_SAYVE=10000
const increase_allowance = new MsgExecuteContract(
  wallet.key.accAddress,
  SAYVE, // Sayve token address.
{
  "increase_allowance": {
    "amount": String(AMOUNT_SAYVE*MILLION),
    "spender": sayveUstPair
  }
}
);
```
Now to provide liquidity we need to provide 50:50 so we need to calculate ust amount based on the price of sayve
```
const provide_liquidity = new MsgExecuteContract(
  wallet.key.accAddress,
  sayveUstPair, // pair address
{
  "provide_liquidity": {
    "assets": [
      {
        "info": {
          "token": {
            "contract_addr": SAYVE
          }
        },
        "amount": String(AMOUNT_SAYVE*MILLION)
      },
      {
        "info": {
          "native_token": {
            "denom": "uusd"
          }
        },
        "amount": String(AMOUNT_UST*MILLION)
      }
    ],
    "auto_stake": false,
    "slippage_tolerance": "0.02"
  }
},
new Coins({ uusd: String(AMOUNT_UST*MILLION) }),
);

const tx = await wallet.createAndSignTx({ msgs: [increase_allowance,provide_liquidity] });
const result = await terra.tx.broadcast(tx);
```
### Staking
4. Stake token
Now that we have an lp we can check the lp balance and amount
let us check for this user: terra1elk88ssjcdhkzx3sx8tnux6dk6nsdspmf3fuat
contract is the sayveUstLPToken: terra1dj3u83nfe9zpqd5r46plwmmmhe30kzr3axtx9x
Query Balance of lp
```
{
  "balance": {
    "address": 
  }
}
```
```
https://bombay-lcd.terra.dev/wasm/contracts/terra1dj3u83nfe9zpqd5r46plwmmmhe30kzr3axtx9x/store?query_msg={%22balance%22:%20{%22address%22:%20%22terra1elk88ssjcdhkzx3sx8tnux6dk6nsdspmf3fuat%22}}
```
balance of amount lp
```
{"height":"8098315","result":{"balance":"10041"}}
````
When we know the amount of lp we can see the breakdown by query the paid address sayveUstPair: 'terra1x23y2hxpxph6wueyqj5m5grlr23z5dt4wvpn0r'

```
{
  "share": {
    "amount": 
  }
}
```
the api
```
https://bombay-lcd.terra.dev/wasm/contracts/terra1x23y2hxpxph6wueyqj5m5grlr23z5dt4wvpn0r/store?query_msg={%22share%22:%20{%22amount%22:%20%2210041%22}}
```
which we can see how much sayve and ust per that lp
```
[
  {
    "info": {
      "token": {
        "contract_addr": "terra16t7x97wuckxm5h927jygjfrt3tcwrzh3u2rlqm"
      }
    },
    "amount": "9901652"
  },
  {
    "info": {
      "native_token": {
        "denom": "uusd"
      }
    },
    "amount": "1029473"
  }
]
```
5. Stake the LP

  we need to provide a {"bond":{}} hook when we send sayveUstLPToken: 'terra1dj3u83nfe9zpqd5r46plwmmmhe30kzr3axtx9x' to the staking contract
  staking: 'terra12yfehzpc69jwxeleck83r6vl99np34frqsx0jv'


```

lp_amount=1
const MILLION=1000000
const stake_lp = new MsgExecuteContract(
  wallet.key.accAddress,
  terra1dj3u83nfe9zpqd5r46plwmmmhe30kzr3axtx9x, 
  {
        send:{
            contract: terra12yfehzpc69jwxeleck83r6vl99np34frqsx0jv,
            amount:lp_amount
            msg: Buffer.from('{"bond":{}}').toString('base64')
        }
      },
);

const tx_stake_lp = await wallet.createAndSignTx({ msgs: [stake_lp] });
const result_buy = await terra.tx.broadcast(stake_lp);

```

6. Check balance of staked on to the staking contract
  staking: 'terra12yfehzpc69jwxeleck83r6vl99np34frqsx0jv'
```
{
  "staker_info": {
    "staker": "terra1t6wyandnky0yr7x46y8n35vz4lcn2a64dnay0v"
  }
}
```
the api
```
https://bombay-lcd.terra.dev/wasm/contracts/terra12yfehzpc69jwxeleck83r6vl99np34frqsx0jv/store?query_msg={%22staker_info%22:%20{%22staker%22:%20%22terra1t6wyandnky0yr7x46y8n35vz4lcn2a64dnay0v%22}}
```
response
```
{"height":"8099849","result":{"staker":"terra1t6wyandnky0yr7x46y8n35vz4lcn2a64dnay0v","reward_index":"1772.222221","bond_amount":"1010000","pending_reward":"1764120370"}}
```

Check contract info
```
{
  "state": {}
}
```
the api
```
https://bombay-lcd.terra.dev/wasm/contracts/terra12yfehzpc69jwxeleck83r6vl99np34frqsx0jv/store?query_msg={"state":{}}
```
response
```
{"height":"8099870","result":{"last_distributed":8099870,"total_bond_amount":"1010000","global_reward_index":"1784.827647732673267326"}}
```


7. Claim Rewards
Withdraw from staking contract staking: 'terra12yfehzpc69jwxeleck83r6vl99np34frqsx0jv'

 ```
const claim_lp_rewards = new MsgExecuteContract(
  wallet.key.accAddress,
  staking, 
  {
    withdraw:{}
  },
);

const tx_claim_lp_rewards = await wallet.createAndSignTx({ msgs: [claim_lp_rewards] });
const result_claim_lp_rewards = await terra.tx.broadcast(claim_lp_rewards);

```


8. Unstake
Unstake from staking contract staking: 'terra12yfehzpc69jwxeleck83r6vl99np34frqsx0jv'
```
const unstake_lp = new MsgExecuteContract(
  wallet.key.accAddress,
  staking, 
{
  unbond: {
    amount: "10000"
  }
},
);

const tx_unstake_lp = await wallet.createAndSignTx({ msgs: [unstake_lp] });
const result_unstake_lp = await terra.tx.broadcast(unstake_lp);
```