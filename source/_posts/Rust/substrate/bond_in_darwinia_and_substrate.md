---
title: Bonding in darwinia and substrate
date: 2020-01-03 17:25
tags: 
    - Rust
    - Substrate
    - blcokchain
---

What does the `bond` mean? Somehow it can be translated as building up a strong `binding` relationship with a PoS network. If you want to put your tokens "at stake", you must `bond` them first. But what does the bond actually do? What's the difference between `substrate` and `darwinia-network`?

## Account abstractions for `bond` -- `Stash` and `Controller`
- Stash Key: The Stash account is meant to hold large amounts of funds. Its private key should be as secure as possible in a cold wallet.

- Controller Key: The Controller account signals choices on behalf of the Stash account, like payout preferences, but should only hold a minimal amount of funds to pay transaction fees. Its private key should be secure as it can affect validator settings, but will be used somewhat regularly for validator maintenance.





## Bond in Substrate
Let's have a look at the following codes:

```Rust
fn bond(origin,
			controller: <T::Lookup as StaticLookup>::Source,
			#[compact] value: BalanceOf<T>,
			payee: RewardDestination
		) {
			let stash = ensure_signed(origin)?;

			if <Bonded<T>>::exists(&stash) {
				return Err("stash already bonded")
			}

			let controller = T::Lookup::lookup(controller)?;

			if <Ledger<T>>::exists(&controller) {
				return Err("controller already paired")
			}

			// reject a bond which is considered to be _dust_.
			if value < T::Currency::minimum_balance() {
				return Err("can not bond with value less than minimum balance")
			}

			// You're auto-bonded forever, here. We might improve this by only bonding when
			// you actually validate/nominate and remove once you unbond __everything__.
			<Bonded<T>>::insert(&stash, &controller);
			<Payee<T>>::insert(&stash, payee);

			let stash_balance = T::Currency::free_balance(&stash);
			let value = value.min(stash_balance);
			let item = StakingLedger { stash, total: value, active: value, unlocking: vec![] };
			Self::update_ledger(&controller, &item);
		}
```

It's really simple. Take the origin account as a stash and lock up `value` of its balance. `controller` will be the account that controls it. `value` must be more than the `minimum_balance` specified by `T::Currency`. The dispatch origin for this call must be _Signed_ by the stash account.

As we all know, "Verify First, Write Last" should be obeyed on Substrate, so verify wheather stash has been bonded and wheather controller has been paired already First. Make sure the `value` is bigger than the `minimum_balance` or the account would be a _dust_ account that should be cleaned. 

Then bond the `stash` and the `controller`.

```rust
<Bonded<T>>::insert(&stash, &controller);
```

Set the Reward Destination:

```rust
<Payee<T>>::insert(&stash, payee);
```

Fianlly update the ledger.

```rust
let stash_balance = T::Currency::free_balance(&stash);
let value = value.min(stash_balance);
let item = StakingLedger { stash, total: value, active: value, unlocking: vec![] };
Self::update_ledger(&controller, &item);
```

Note if `value` is larger than `stash_balance`, the bond should not faill and will bond all the balances in the stash account to the controller.

`bond` doesn't mean transfer tokens to the `controller` account. The tokens are still at the `stash` account. Howerever, the Staking Ledger's active item will become the minium between the `value` and the `stash_balance`.

## Bond in Darwinia
The `bond` operation in darwinia-network is almost the same as the `bond` operation in substrate. However, there are also something different between them. As darwinia has two tokens -- `ring` and `kton`, the `bond` operation should be able to handle both tokens.

```rust
match value {
	StakingBalances::RingBalance(r) => {
		let stash_balance = T::Ring::free_balance(&stash);
		let value = r.min(stash_balance);

		Self::bond_helper_in_ring(&stash, &controller, value, promise_month, ledger);

		<RingPool<T>>::mutate(|r| *r += value);
		<Module<T>>::deposit_event(RawEvent::Bond(
			StakingBalances::RingBalance(value.saturated_into()),
			now,
			promise_month,
		));
	},
	StakingBalances::KtonBalance(k) => {
		let stash_balance = T::Kton::free_balance(&stash);
		let value = k.min(stash_balance);

		Self::bond_helper_in_kton(&controller, value, ledger);

		<KtonPool<T>>::mutate(|k| *k += value);
		<Module<T>>::deposit_event(RawEvent::Bond(
			StakingBalances::KtonBalance(value.saturated_into()),
			now,
			promise_month,
		));
	},
}
```

There are also a special rule in darwinia-network: Users can choose to lock RING for 3-36 months in the process of Staking, and the system will offer a KTON token as reward for users participating in Staking. So we need `bond_helper_in_ring` and `bond_helper_in_kton`.

```rust
fn bond_helper_in_ring(
		stash: &T::AccountId,
		controller: &T::AccountId,
		value: RingBalance<T>,
		promise_month: Moment,
		mut ledger: StakingLedger<T::AccountId, RingBalance<T>, KtonBalance<T>, T::Moment>,
	) {
		// if stash promise to a extra-lock
		// there will be extra reward, kton, which
		// can also be use to stake.
		if promise_month >= 3 {
			ledger.active_deposit_ring += value;
			// for now, kton_return is free
			// mint kton
			let kton_return = inflation::compute_kton_return::<T>(value, promise_month);
			let kton_positive_imbalance = T::Kton::deposit_creating(&stash, kton_return);
			T::KtonReward::on_unbalanced(kton_positive_imbalance);
			let now = <timestamp::Module<T>>::now();
			ledger.deposit_items.push(TimeDepositItem {
				value,
				start_time: now,
				expire_time: now + T::Moment::saturated_from((promise_month * MONTH_IN_MILLISECONDS).into()),
			});
		}
		ledger.active_ring = ledger.active_ring.saturating_add(value);

		Self::update_ledger(&controller, &mut ledger, StakingBalances::RingBalance(value));
	}
```

As the code shows, `bond_helper_in_ring` computes the `kton` should return and update the ledger. 

`bond_helper_in_kton` is simper, it just updates the ledger.

```rust
fn bond_helper_in_kton(
		controller: &T::AccountId,
		value: KtonBalance<T>,
		mut ledger: StakingLedger<T::AccountId, RingBalance<T>, KtonBalance<T>, T::Moment>,
	) {
		ledger.active_kton += value;

		Self::update_ledger(&controller, &mut ledger, StakingBalances::KtonBalance(value));
	}
```